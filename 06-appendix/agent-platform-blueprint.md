# AgentBridge：把「Claude Code 客户端」变成 IM 里的多机器人劳动力池

> **这份文档的用途**：交给任意一个 AI（Claude Code / Codex / Cursor）当**施工图**，让它从零复刻一套
> 「IM 群里 @ 机器人 → 本机 Claude 客户端干活 → 结果发回群」的多机器人平台。
> **完全业务无关**：不含任何行业/公司/业务逻辑，只有机制。你要做的业务能力全部落在**可插拔的 profile 目录**里。
>
> **本页是附录，写法与正文各层不同**：正文是纯自然语言方案，本页为了能直接施工，保留了接口契约、表结构与少量伪码
> （**都是通用示例，不是本项目的产品源码**）。与之配套的自然语言版本见 [机器人体系](../04-ai-engineering/bots-architecture.md)。
>
> 代号 `agentbridge`（随便改）。目标形态：一个可开源的 GitHub 仓库，`server/` + `runner/` + `adapters/` 三包。
> 阅读顺序：§1 心智模型 → §3 数据模型 → §4 接口契约 → §6 Runner → §7 多账号池。§10 是知识库层（第二大脑）。§15 是可直接粘给 AI 的复刻 prompt。

---

## 1. 心智模型：这套东西到底是什么

一句话：**IM 群是工位，任务表是工单，本机的 Claude Code 订阅额度是劳动力**。

三个必须先接受的前提，否则会设计成另一个（做不出来的）东西：

1. **MCP / Connector 是「出方向」的**——它让 Claude 去调外部工具，**不能反过来让一条 IM 消息「打进」你电脑上已登录的 Claude 客户端**。
   所以唯一可行的机制是**本地 runner 中继**：公网后端只建任务，本机进程轮询领任务，用 `claude -p` headless 执行。
2. **后端一行模型代码都不跑**。后端 = 收发消息 + 任务队列 + 文件中转 + 审批。这带来三个好处：
   模型密钥/订阅凭证永远不下放到服务器；后端可以随业务系统一起部署、无 GPU/无额度成本；执行端可随时换（Mac→NUC→服务器）。
3. **代价：执行端必须开机在线**。runner 所在机器休眠/掉线 = 机器人「收到」但不出结果。
   这是本架构的**唯一硬伤**，接受它（并加心跳告警），或把 runner 挪到一台常开的小主机（推荐：一台 mini 主机 24h 常开）。

```
┌────────────┐   事件推送     ┌──────────────────────┐   轮询领单     ┌────────────────────────┐
│ IM 平台     │ ──────────────►│  后端(无模型)         │◄───────────────│ runner(执行端·你的电脑) │
│ 飞书/Slack  │                │  · 消息解析/@判定      │                │  · N 条任务线并行        │
│ 群 + 私聊   │◄───────────────│  · 任务表(状态机)      │───────────────►│  · claude -p headless   │
└────────────┘   发文本/图/文件 │  · 附件下载/产物上传   │   回报结果      │  · 多账号池调度          │
                              │  · 审批闸/白名单       │                │  · git worktree 隔离     │
                              └──────────────────────┘                └────────────────────────┘
                                        │                                        │
                                   任务表(单表)                          账号池 acct#1..#N
                                                                    (每账号独立 CLAUDE_CONFIG_DIR)
```

**一个后端 + N 个 IM 应用 + 1 个 runner 进程 + M 个模型账号 + 1 座知识库 vault** = N 个"同事"。
机器人身份由 IM 应用（app_id）决定，能力由 profile 决定，算力由账号池决定，**共同经验由知识库沉淀**（§10）——
四者正交，这是整套设计的核心：加一个同事不用动代码，加一份算力不用动逻辑，加一条经验不用动任何进程。

---

## 2. 能力清单（复刻目标）

| 能力 | 说明 | 优先级 |
|---|---|---|
| 群里 @ 机器人提需求 → 秒回收到 → 结果回群 | 基础闭环 | P0 |
| 私聊直接提需求（不用 @） | 私聊权限单独收紧，见 §11 | P1 |
| 带附件（图/表格/文档）提需求；回复某条消息再 @ | 引用内容+附件一起带上 | P0 |
| 产出型任务：产出成品文件（图表/表格/文档/海报）直接发回群 | 不碰代码仓 | P0 |
| 改码型任务：诊断→改到分支→人工审批→合并部署 | 主干只有管理员确认后才被合入 | P1 |
| 一岗一机器人：多个 IM 应用共用一个后端回调，按 app_id 分身份 | 免关键词路由误判 | P0 |
| 长期记忆：便签 → 夜间整理 → 开工必读 | 越用越懂你 | P1 |
| 反馈吸收：直接回复机器人答案即被记住 | 不用 @、静默点赞示意 | P2 |
| 定时任务线：日报 / 巡检 / 定时图报 | runner 侧 lane，非 IM 触发 | P2 |
| 多账号池：额度用尽自动切账号/排队，恢复自动重跑 | §7，本方案重点 | P0 |
| **知识库（第二大脑）**：IM 一句「存」投料 → 夜间消化成 wiki → 所有机器人只读引用 | §10，跨岗位共享的组织记忆 | P1 |
| 双脑同步：个人脑 ↔ 组织脑，黑名单制增量互投 | §10.5 | P2 |

---

## 3. 数据模型（一张表打天下）

**只用一张 `job` 表**。不要拆成"任务/审批/交付/记忆"四张表——状态机在一张表里最容易保证幂等和原子认领。

```sql
CREATE TABLE job (
  id            BIGINT PRIMARY KEY AUTO_INCREMENT,
  kind          VARCHAR(32)  NOT NULL DEFAULT 'chat',  -- 任务类型，见下（★身份也编码在这里）
  status        SMALLINT     NOT NULL DEFAULT 0,       -- 状态机，见下
  channel_id    VARCHAR(128) NOT NULL,                 -- IM 会话 id（群/私聊）
  message_id    VARCHAR(160) NOT NULL UNIQUE,          -- ★幂等键：IM 消息 id（+机器人后缀，见 §5.4）
  user_id       VARCHAR(128),                          -- 提问人 open_id
  user_name     VARCHAR(64),
  is_p2p        BOOL         NOT NULL DEFAULT 0,       -- 私聊来源（权限降级用）
  question      TEXT,                                  -- 需求正文（含引用内容拼接）
  attachments   TEXT,                                  -- JSON：[{mid,key,type,name}] 输入附件引用
  result_text   TEXT,                                  -- 结果摘要（回群的文字）
  artifacts     TEXT,                                  -- JSON：已交付的产物文件名
  branch        VARCHAR(128),  commit_hash VARCHAR(64),-- 改码型专用
  needs_approve BOOL DEFAULT 0, approver_id VARCHAR(128), approved_time DATETIME,
  account       VARCHAR(32),                           -- ★本单实际用了哪个模型账号（审计/成本归集）
  error         TEXT,
  claimed_time  DATETIME, finished_time DATETIME,
  create_time   DATETIME NOT NULL, update_time DATETIME NOT NULL,
  KEY idx_poll (kind, status, id),                     -- ★领单索引
  KEY idx_ch   (channel_id, kind, status)              -- 短期上下文查询
);
```

**status 状态机**（8 态，两条流水线共用）：

```
0 待处理 ──领单──► 1 处理中 ──┬─ 无需上线 ─────────────────────────► 5 已完成
                              ├─ 有改动 ──► 2 待审批 ──确认──► 3 待部署 ──► 4 部署中 ──► 5 已完成
                              │                        └─取消──► 7 已取消
                              └─ 出错 ──────────────────────────────► 6 失败
```

**kind 的双重身份**（★关键技巧）：`kind` 既是任务类型也是**机器人身份**，格式 `<type>[:<profile_key>]`：
`chat:design`、`chat:ops`、`fix`、`deploy`、`review`。好处是**加一个新机器人零迁移**——不用加列、不用改表。
约束：`profile_key ≤ 9 字符`（留出 `chat:` 前缀空间）。领单用 `kind LIKE 'chat%'` 或精确匹配。

---

## 4. 后端接口契约（复刻时照抄）

全部 **POST + JSON**，runner 侧接口用共享密钥 `RUNNER_TOKEN` 鉴权（请求体带 `token` 字段，或 `Authorization: Bearer`）。
后端框架无所谓（Django/FastAPI 都行）；下面是**语义契约**，AI 照此实现即可互通。

| 端点 | 方向 | 作用 |
|---|---|---|
| `POST /api/bot/callback` | IM → 后端 | 事件回调（**所有机器人共用一个 URL**，按 `header.app_id` 认身份）；处理 URL challenge |
| `POST /api/bot/runner/poll` | runner → 后端 | 原子领单，返回一条任务 + 上下文 |
| `POST /api/bot/runner/resource` | runner → 后端 | 下载输入附件（后端用 IM app token 取，**密钥不下放**）；校验 (mid,key) 属于本 job |
| `POST /api/bot/runner/deliver` | runner → 后端 | 上传一个产物文件；图片只上传拿 `image_key` **暂存不发**，其它文件直接发回会话 |
| `POST /api/bot/runner/report` | runner → 后端 | 收尾：文字结论 + 暂存图组成**一条富文本**发回会话，落库置终态 |
| `POST /api/bot/runner/history` | runner → 后端 | 拉某 profile 近 N 天问答流水（夜间记忆整理用） |
| `POST /api/bot/runner/heartbeat` | runner → 后端 | 上报账号池健康（在线账号数/冷却中账号/队列深度），供告警 |
| `POST /api/brain/save` | IM 回调内部 | 识别到「存」口令时落一条待消化原料（§10.4） |
| `POST /api/brain/pull` / `/ack` | 拉取器 → 后端 | 两段式取料：先落盘再 ack，中途崩溃只会重拉不会丢件 |

### 4.1 poll 的原子认领（并发正确性核心）

```python
# 1) 僵尸单自愈：被领走超 45 分钟没回报的（runner 重启把子进程带走了），退回队列
Job.filter(kind__startswith='chat', status=1, claimed_time__lt=now-45min).update(status=0, claimed_time=None)
# 2) 取一条
job = Job.filter(kind__startswith='chat', status=0).order_by('id').first()
if not job: return {'job': None}
# 3) ★带条件的 UPDATE 才是认领；影响行数 0 = 被别的线抢走了，本次返回空
claimed = Job.filter(id=job.id, status=0).update(status=1, claimed_time=now)
if not claimed: return {'job': None}
```

绝不要"先读后写"两步认领；必须靠 `WHERE status=0` 的条件更新返回值判断。多任务线/多台 runner 因此可以随意加。

**poll 响应体**（runner 只认这个结构）：

```json
{"job": {
  "id": 123, "kind": "chat", "profile": "design", "action": "chat",
  "channel_id": "oc_xxx", "user_id": "ou_xxx", "user_name": "张三", "is_p2p": false,
  "question": "【引用/回复的内容】...\n\n【本次需求】把这张表做成柱状图",
  "resources": [{"mid": "om_xxx", "key": "file_v3_xxx", "type": "file", "name": "data.xlsx"}],
  "recent": [{"q": "上次的问题", "a": "上次答案摘要"}]
}}
```

### 4.2 report

```json
{"token":"...", "job_id":123, "ok":true,
 "result_text":"已按门店做成柱状图，最高的是 A 店…",
 "artifacts":["chart.png","汇总.xlsx"], "account":"acct2", "error":null}
```
后端据此：`status=5/6` + `finished_time` + 组装富文本发回 `channel_id`。

---

## 5. IM 接入层（以飞书为参考实现，Slack/钉钉/企微同构）

### 5.1 一个应用 = 一个机器人 = 一个"同事"
每个岗位建一个 IM 自建应用，**事件订阅全部指向同一个回调 URL**，后端按 `header.app_id` 反查配置表拿到 `profile_key`。
配置表（放服务器 gitignored 的 `local_settings` 或环境变量）：

```python
BOTS = {
  'design': {'app_id':'cli_a','app_secret':'***','allowed_channels':'oc_1,oc_2'},
  'ops':    {'app_id':'cli_b','app_secret':'***','allowed_channels':''},   # 空=不限群（不建议）
}
```

### 5.2 飞书侧必配三件套（**三缺一 = 群里 @ 了没反应**，这是 90% 的排障答案）
1. **权限**：`im:message`（收消息）、接收群内 @ 消息、`im:message:send_as_bot`（发）、`im:resource`（读/传图与文件）、
   可选通讯录只读（解析提问人姓名，缺了只降级为空）、要私聊则加**读取用户发给机器人的单聊消息**。
2. **事件订阅**：方式选「发送至开发者服务器」，URL 填回调，**Encrypt Key 留空用明文**（省一层 AES，内网/HTTPS 足够），
   事件列表必须显式勾 `im.message.receive_v1`。
3. **发布版本**（改完权限/事件一定要重新发版），再把机器人拉进目标群。

回调必须先处理 URL 校验：`if body.get('challenge'): return {'challenge': ...}`。

### 5.3 消息解析要点（踩过的坑都在这）
- **@ 判定**：群聊必须**精确匹配本应用的 bot open_id**（`event.message.mentions[*].id.open_id`），不能按名字匹配。
  同群多机器人时，@ 其中一个，其它应用**也会收到事件**（只要它们开了读全群消息）——靠精确匹配才不会重复接单。
- **富文本/图文**：纯文本消息夹不了图；用户发截图必然是 `post`/`image` 类型，要遍历 content 的 element 树取 `image_key`。
- **引用回复**：取 `message.parent_id` 反查原消息，把内容与附件一并带上，拼成
  `【引用/回复的内容】…\n\n【本次需求】…`（profile 关键词路由只对后半段判定）。
- **私聊图文拆条**：私聊里用户习惯"先丢图，再说事"，是两条消息。纯附件消息**不要单独成单**（会跑出一个 question 为空的废单），
  按 `(channel,bot)` 缓存 10 分钟，下一条带文字的消息把它们并进来。群聊 @ 天然图文同条，不需要。
- **发送前洗 markdown**：IM 的 text 消息不渲染 markdown，`**粗体**`/`##` 会原样刷屏。发送前统一 `strip_md()`。

### 5.4 幂等与去重（★一条消息 @ 多个机器人）
IM 会重推同一条事件 → 用 `message_id UNIQUE` + `get_or_create` 天然幂等。
但一条消息同时 @ 三个机器人时，三个应用收到**同一个 message_id**，若直接当重复丢弃就只有一个会响应。
解法：**去重键按机器人隔离** → `dedup_id = message_id + '#' + bot_key`；用到真实 mid 时取 `'#'` 前半截。

---

## 6. Runner（执行端）

### 6.1 进程模型：一进程多 lane
```
main()
 ├─ chat-lane × K      产出型任务（可并行，每单一个独立临时空目录）
 ├─ code-lane × 1      改码型任务（★必须串行：git 工作区互斥）
 ├─ review-lane        事件驱动的轻任务（如"有人提交了什么就去看一眼"）
 ├─ cron-lane          定时产出（日报/巡检），本地时钟触发
 └─ memory-lane        每天固定时刻整理记忆
```
每条 lane 是一个 daemon 线程，循环 `poll → process → report`，异常必须兜底 report 失败（否则任务永远卡在"处理中"，
只能等 45 分钟僵尸自愈）。首次轮询按 lane 序号错开 1s，避免同秒争抢。

### 6.2 调用 Claude 客户端的标准形态
```python
cmd = [CLAUDE_BIN, "-p", prompt,
       "--permission-mode", "bypassPermissions",   # 全自动：允许直接改文件/跑命令
       "--output-format", "text"]
cmd += ["--add-dir", assets_dir]                   # 授权它读写素材/记忆目录
cmd += ["--add-dir", d for d in extra_repos]       # 改码型：授权其它代码仓
if MODEL: cmd += ["--model", MODEL]                # 空=继承客户端默认档
p = subprocess.run(cmd, cwd=workdir, env=acct_env, # ★env 决定用哪个账号，见 §7
                   capture_output=True, text=True, timeout=TIMEOUT)
```
- **产出型任务的 cwd = 一个空白临时目录**（`tempfile.mkdtemp()`），**不在任何 git 仓内**，任务结束 `rmtree`。
  这是最强的安全边界：它想碰代码也没得碰。
- **改码型任务的 cwd = 专供机器人的 git worktree**，每单开工先 `git fetch && git checkout -f -B bot/job-<id> origin/main`
  （`-f` 丢弃上一单残留）。★worktree 里**永远不要 checkout 字面主干分支**（主工作区占着，会失败），一律开独立分支。
- **超时**：产出型和改码型分开设（默认 1800s）。超时必须 kill 子进程树并 report 失败。

### 6.3 产物收集与降噪
`_collect_outputs(workdir)` 遍历工作目录根，**排除**：`inputs/`（用户附件）、`assets/`（注入的素材）、`scratch/`（过程草稿）、
隐藏文件、以及脚本类中间产物 `.py .sh .js .ts .ipynb .r`（模型干活写的 `build_xxx.py` 不该刷进群；真要交付脚本让它打 zip）。
prompt 里同步要求：**中间文件放 scratch/，成品少而精，能合成一个文件绝不拆几个**。

### 6.4 交付管道
1. 每个产物 `POST /runner/deliver`（multipart 或 base64），后端用**该机器人自己的应用凭证**上传 IM；
2. 图片只上传拿 `image_key` **暂存**（放缓存，key=job_id），文件（xlsx/docx/pdf…）走文件接口**直接发**；
3. `report` 时把结论文字 + 暂存的图组成**一条富文本消息**发出 → 图文一体、省 API 调用、群里不刷屏。

### 6.5 额度耗尽的识别与退避
```python
def is_limit_hit(text):
    t = (text or '').lower()
    return ('usage limit' in t or 'weekly limit' in t
            or ('hit your' in t and 'limit' in t) or ('limit' in t and 'resets' in t))
```
命中时：**不 report 失败**，而是把任务**退回队列**（`status=0`）+ 标记该账号进入冷却（§7），
群里 30 分钟内只提示一次（避免刷屏）。额度恢复后自动重跑——用户视角就是"慢了点"，不是"坏了"。

---

## 7. ★多账号池（本方案的算力层）

### 7.1 为什么要多个账号
单个订阅账号有**周/日额度上限**和**并发上限**：一条重活（改码/大文档）可能吃掉几个小时的额度，
后面所有 lane 全部干等。多账号池把「lane 并发」与「账号额度」解耦：**lane 是工位，账号是电池**，工位数不变，电池可以轮换。

> ### ⚠️ 合规前置声明（本节必读，先看完再往下）
>
> 本节讲的是**把多个各自独立订阅的账号，在其持有人明确授权下，集中到同一台执行机器上排队使用**——
> 相当于"团队里 N 个人各自的额度，由同一台机器代跑"。三条硬约束：
>
> 1. **每个账号必须对应一个真实使用者本人的订阅**，且该本人知情并授权；
> 2. 每单在 `job.account` 字段留痕，**成本可归集到人**、可审计；
> 3. 这**不是**把一个账号拆给多人并发使用，也**不是**绕过任何服务商的额度、并发或计费限制。
>
> 落地前请**自行核对你所用服务的服务条款**（含团队/企业方案的席位与共享规则）——**条款优先于本文的任何工程建议**。
> 只用一个账号也完全可行：跳过本节，撞额度时排队重试（§6.5）本身就是完整可用的方案，多账号池只是把等待时间摊平。

### 7.2 账号隔离机制（已实测验证）
Claude Code 客户端的登录态默认在 macOS Keychain 里，**同一用户目录只能有一个登录账号**。两个环境变量解决：

| 变量 | 作用 | 实测结论 |
|---|---|---|
| `CLAUDE_CONFIG_DIR` | 指定配置/会话/历史目录 | 指到一个新目录后 `claude auth status` → `{"loggedIn": false}`，即**登录态随目录隔离** |
| `CLAUDE_CODE_OAUTH_TOKEN` | 直接注入长期令牌 | 设为一个令牌后 `auth status` → `{"loggedIn":true,"authMethod":"oauth_token"}`，**无需交互登录** |

于是一个账号 = 一份环境：
```bash
# 每个账号一次性准备（在该账号持有人的机器/会话上执行）
claude setup-token            # 需要该账号有订阅；输出长期令牌，妥善保存
```
```python
# runner 侧：账号 = 一组 env
ACCOUNTS = [
  {"name":"acct1", "config_dir":"~/.agentbridge/acct1", "token": os.environ["AB_TOKEN_1"], "weight":2},
  {"name":"acct2", "config_dir":"~/.agentbridge/acct2", "token": os.environ["AB_TOKEN_2"], "weight":1},
  {"name":"acct3", "config_dir":"~/.agentbridge/acct3", "token": os.environ["AB_TOKEN_3"], "weight":1},
]
def acct_env(a):
    e = os.environ.copy()
    e["CLAUDE_CONFIG_DIR"] = os.path.expanduser(a["config_dir"])
    e["CLAUDE_CODE_OAUTH_TOKEN"] = a["token"]
    e.pop("ANTHROPIC_API_KEY", None)          # ★别让 API key 抢走计费口
    return e
```
★令牌是**凭证**：只放环境变量/密钥管理器，绝不进 git、绝不写日志、绝不打印到群里。轮换时直接换 env 重启 runner。

### 7.3 池调度器（可直接实现的伪代码）

```python
class AccountPool:
    """租约式账号池：lane 借一个账号跑一单，跑完归还；撞额度的账号进冷却。"""
    def __init__(self, accounts, per_acct_concurrency=1):
        self.slots = {a["name"]: per_acct_concurrency for a in accounts}   # 剩余并发位
        self.cool  = {}          # name -> 冷却到期时间戳
        self.acct  = {a["name"]: a for a in accounts}
        self.lock, self.cv = threading.Lock(), threading.Condition()
        self.affinity = {}       # channel_id -> 上次用的账号（提高 prompt cache 命中）

    def acquire(self, channel_id=None, timeout=None):
        with self.cv:
            deadline = time.time() + (timeout or 1e9)
            while True:
                now = time.time()
                cands = [n for n in self.slots
                         if self.slots[n] > 0 and self.cool.get(n, 0) <= now]
                if cands:
                    pref = self.affinity.get(channel_id)                 # 1) 亲和优先
                    name = pref if pref in cands else max(               # 2) 否则按权重×余位挑
                        cands, key=lambda n: (self.acct[n]["weight"], self.slots[n]))
                    self.slots[name] -= 1
                    if channel_id: self.affinity[channel_id] = name
                    return self.acct[name]
                if time.time() >= deadline: return None                  # 全忙/全冷却 → 调用方退回队列
                self.cv.wait(timeout=5)

    def release(self, name, limit_hit=False, reset_at=None):
        with self.cv:
            self.slots[name] += 1
            if limit_hit:
                # 有 reset 时间就睡到那时刻，否则默认冷却 15 分钟
                self.cool[name] = reset_at or (time.time() + 900)
            self.cv.notify_all()

    def status(self):    # 供 heartbeat 上报
        now = time.time()
        return {n: ("cooling" if self.cool.get(n,0) > now else "idle" if self.slots[n] else "busy")
                for n in self.slots}
```

**lane 里的用法**：
```python
job = poll()
acct = pool.acquire(channel_id=job["channel_id"], timeout=60)
if not acct:                       # 全员冷却
    requeue(job["id"]);  notify_once(job["channel_id"], "排队中，额度恢复后自动继续"); continue
try:
    rc, out, err = run_claude(job, env=acct_env(acct))
    if is_limit_hit(out + err):
        pool.release(acct["name"], limit_hit=True, reset_at=parse_reset(out+err))
        requeue(job["id"]);  continue          # ★换个账号自动重跑，不算失败
    report(job["id"], ok=(rc == 0), result_text=out, account=acct["name"])
finally:
    pool.release(acct["name"])
```

### 7.4 调度策略要点
- **亲和性**：同一会话尽量固定同一账号 → 提示缓存命中率高、上下文连贯。冷却时才漂移。
- **权重**：额度大的账号 `weight` 高。重活（改码/长文档）优先给高权重账号，轻活（判断/日报/评论）给低权重。
- **档位分级**：不同任务类型显式指定不同模型档（重活高档、轻活低档），这是**比加账号更便宜的省额度手段**。
  客户端默认档留空即继承本机设置。
- **一账号一并发**（`per_acct_concurrency=1`）是安全默认值；确认账号策略允许再往上调。
- **公平性**：`acquire` 用条件变量而非忙等；全员冷却时**退回队列**而不是阻塞 lane（否则新任务也进不来）。
- **可观测**：`heartbeat` 每 60s 上报 `{accounts: pool.status(), queue_depth: n}`；后端发现"全员冷却 > 30min"或
  "runner 心跳丢失 > 5min"就往运维群推一条告警——**这是本架构唯一的单点，必须有告警**。

### 7.5 横向扩展：多台 runner
账号池是**进程内**的，但任务认领是**数据库原子的**，所以直接**再起一台 runner**（不同机器、不同账号集合）即可水平扩展，
无需任何协调服务。唯一约束：**改码 lane 全局只能有一条**（或按仓库加分布式互斥锁），因为 git 工作区互斥。

---

## 8. Profile：机器人的"人设 + 能力 + 素材"

profile 是**业务落点**，也是这套架构与具体业务的唯一接口。一个 profile = 一个 key + 一段角色 prompt + 偏好技能 + 一个素材目录。

```python
PROFILES = [{
  "key": "design",                       # ≤9 字符，与 BOTS 的 key 一致
  "cn":  "设计助手",
  "role": "你是资深视觉设计…（人设/口径/红线，几十行）",
  "skills": "海报/网页用 artifact-design、图表用 dataviz",   # 只是在 prompt 里点名让它优先用哪个
  "keywords": ["设计","出图","海报"],     # 仅通用兜底机器人用；一岗一应用时不需要
}]
```

**素材目录** `assets_base/<key>/`（**不进任何 git 仓**）：
```
assets/<key>/
  ├─ 口径与红线.md      ← 必读，与记忆冲突时红线赢
  ├─ 模板/ 品牌/ 字体/   ← 可复用素材
  ├─ 记忆.md            ← 长期记忆（§9）
  ├─ _inbox/            ← 记忆便签（下划线开头 = 不注入工作区）
  └─ .memory_last_sync  ← 增量标记
```
任务开跑前把整目录（排除 `_`/`.` 开头）拷进工作区 `assets/`，prompt 里写明"先读再动手"。
**放进去即时生效，无需改代码、无需重启**——这是让业务同学自己维护机器人的关键设计。

**路由**：一岗一应用时由后端下发 `profile`（准确）；通用兜底机器人才走关键词（先看正文前 12 字，再看全文包含；
引用场景只对「【本次需求】」之后判定）。★关键词路由必然误伤（研发群聊"成本"被当财务），所以**能一岗一应用就别用关键词**。

---

## 9. 记忆与学习（三层 + 一个夜班）

| 层 | 机制 | 生效时延 |
|---|---|---|
| 素材/口径 | 人工放 `assets/<key>/` | 即时 |
| 短期上下文 | poll 响应带本会话最近 3 轮已完成问答（`recent`） | 实时 |
| 记忆便签 | 每单收尾，prompt 要求把值得长期记的点写 `_inbox/job-<id>.md`（用户明说「记住…」的**必写**） | 当晚 |
| 长期记忆 | memory-lane 每天固定时刻：便签 + `/runner/history` 拉的问答流水 → 一次模型调用整理进 `记忆.md` | 次日 |
| （再上一层） | **组织知识库**：跨岗位、跨人的结论沉淀 → 见 §10 | 次日 |
| 反馈吸收 | 用户**直接回复**机器人的答案（不 @）→ 校验被回复消息确属本机器人（比对 sender app_id）→ 存一条终态"反馈"记录，不跑任务、给对方消息点个👍示意；立刻进 `recent`，夜里按最高优先级并入记忆 | 当天 |

`记忆.md` 的整理约束（写进 prompt）：分节（事实/纠正与偏好/口径确认/常见任务模式/**待人工确认**）、≤150 行、
新旧矛盾以新为准、**红线文档优先、绝不擅改正式口径文档**。成功后清空 `_inbox`、写增量标记。
「待人工确认」小节 = 机器人建议升级为正式口径的内容，人工每周瞄一眼。人可随时手改/删 `记忆.md`。

---
## 10. 知识库层：第二大脑（双脑同步）

§9 的记忆是**岗位私产**（一个 profile 一份 `记忆.md`，越滚越长、彼此不通、只对得起自己那条会话线）。
再往上要一层**组织级知识库**：把所有机器人和所有人类会话里沉出来的结论，收敛成**一座可检索、可版本化、可被任意 agent 只读引用的 wiki**。
模式照抄 Karpathy 的 LLM-Wiki：**人只负责往收件箱扔原料，agent 负责消化、沉淀、组织**。

### 10.1 三层记忆的分工（别混）

| 层 | 载体 | 谁写 | 谁读 | 生命周期 |
|---|---|---|---|---|
| 会话记忆 | 各项目 `.../memory/*.md` + `MEMORY.md` 索引 | agent 自动 | 该项目下的会话 | 跟项目走 |
| 岗位记忆 | `profiles/<key>/记忆.md` + `_inbox/` | 该岗位机器人 | 该岗位 | 跟岗位走 |
| **知识库** | **vault：`wiki/<领域>/*.md` + `INDEX.md`（git 版本化）** | **夜间消化 agent** | **所有机器人 + 所有人 + Obsidian** | **永久，唯一事实源在 `raw/archive/`** |

判据：**只对一条会话有用 → 会话记忆；只对一个岗位有用 → 岗位记忆；换个人换个场景还成立 → 知识库**。

### 10.2 Vault 规格（两座脑共用同一套规则）

```
vault/
├─ CLAUDE.md                 ← ★管理员规则（口令/落点判断/格式/红线），agent 每次消化前必读
├─ raw/inbox/                ← 收件箱：未消化原料（一行字也算）
├─ raw/archive/YYYY-MM/      ← 已消化原料按月归档；★唯一事实源，永不删改
├─ wiki/
│   ├─ INDEX.md              ← 全库索引，一页一行钩子（检索入口）
│   └─ <一级领域>/*.md        ← 结构化知识页
├─ memory-links/             ← 各项目会话记忆目录的软链（只读参考，绝不改写）
└─ peer-mirror/              ← 另一座脑的本地只读镜像（软链，见 10.5）
```

**wiki 页硬格式**：frontmatter 必带 `title` / `tags`（首个=一级领域）/ `sources`（archive 路径或 URL）/ `updated`；
正文是**综合提炼**（观点、结论、与既有认知的冲突或印证），不是原文摘抄堆砌；一页一主题，超 ~200 行拆页互链；
相关页之间打 `[[双链]]`（Obsidian 可视化 + 给 agent 做图谱遍历）。

### 10.3 消化 lane（两条口令就是全部逻辑）

```bash
# 夜间消化（launchd/systemd 定时；★收件箱空就直接退出，别白烧额度）
[ -z "$(ls vault/raw/inbox/*.md 2>/dev/null)" ] && exit 0
cd vault && claude -p "消化收件箱" --permission-mode bypassPermissions >> .sync/digest.log 2>&1
```
`CLAUDE.md` 里定义的「消化收件箱」标准流程（agent 照做，不用写代码）：
1. 列收件箱，空则结束；2. **逐个完整读取**（纯链接先抓全文存成同名 `.md` 再消化）；
3. 判落点——**优先重写融合进已有页**，确无归属才新建（★这一条决定了库会不会退化成流水账）；
4. 写 frontmatter + 打双链；5. 原料移进 `raw/archive/当月/`；6. 更新 `INDEX.md`；7. `git add -A && git commit` 一行说清消化了什么。

第二条口令 **「体检」**（= Karpathy 说的 lint，每周或手动跑）：全库扫页面间矛盾、被新来源取代的过时结论、
孤立页（无入站链接）、INDEX 与实际文件不一致、frontmatter 缺失——**发现即修，一次 commit**。
没有体检，知识库三个月后必然自相矛盾。

### 10.4 四条投料通道

| 通道 | 机制 | 时延 |
|---|---|---|
| **IM 一句「存」** | 群里/私聊对机器人说「存 <链接或一段话>」→ 后端落 `brain_item` 表 → 本机拉取器每 10 分钟拉走落盘 | ≤10 分钟 |
| 机器人便签 | 每单收尾写 `_inbox/job-<id>.md`（§9），夜里由岗位记忆整理；值得全局知道的另投知识库收件箱 | 当晚 |
| 人工丢料 | 直接把 md/pdf/截图丢进 `raw/inbox/`（手机上用 Obsidian 同步也行） | 当晚 |
| 会话沉淀 | 任何 agent 会话产出有价值综合结论时，**主动提议**写回对应 wiki 页（"wiki 是工件，聊天只是接口"） | 即时 |

**「存」口令的后端契约**（两段式，防止拉到了但没落盘导致丢件）：

```
POST /api/brain/save   {token, channel_id, user_id, content}   # IM 回调侧识别到「存」口令时调用；落库 status=0
POST /api/brain/pull   {token}            → {data:[{id, content, create_time}, ...]}   # 只返回未 ack 的
POST /api/brain/ack    {token, ids:[...]} → {acked: n}          # ★全部写盘成功后才 ack，置 status=1
```
拉取器落盘用 **`tmp` + `os.rename`** 原子写，文件名 `YYYY-MM-DD-存<id>-<首行摘要>.md`，
frontmatter 记 `source/saved/brain_id` 便于回溯。**先落盘、再 ack**，中途崩溃只会重复拉取（幂等，文件已存在则跳过），不会丢。

### 10.5 双脑同步（个人脑 ↔ 组织脑）

两座结构完全相同的 vault：**个人脑**在你自己机器上（含 `个人/` 目录等私域内容），**组织脑**在 runner 机器上（机器人共享，各岗位便签汇入）。
每 4 小时跑一次双向同步：

```python
# ① 拉：把组织脑镜像到本地只读副本（个人脑里以软链挂进来，Obsidian 可直接看，★绝不在镜像里改）
git -C ~/peer_mirror pull --ff-only

# ② 推：个人脑的 wiki 页投递进组织脑收件箱——★黑名单制 + md5 增量
for page in vault/wiki/**/*.md:
    if page.startswith("个人/"):                     continue   # 私域不出门
    if "不共享" in tags or any("机密" in t for t in tags): continue   # 手动豁免 / 机密标签
    if md5(page) == state.get(rel):                  continue   # 没变不重发；改了自动发新版
    scp page  peer:~/vault/raw/inbox/YYYYMMDD-personal-<name>.md
    state[rel] = md5(page)
```
要点：
- **黑名单制而非白名单**（内部使用默认互通，只拦少数机密标签）——白名单制的结局是没人维护、什么都同步不过去。
- **md5 state 文件**做增量幂等：页面没变不重发，改了自动发新版本让对方重新消化融合。
- 投递的是**已消化的 wiki 页**，不是原始便签——跨脑传播的应该是结论，不是原料。
- 镜像**只读**（软链进 vault 供人和 agent 查阅），写权限只在源头，避免双写冲突。

### 10.6 消费侧：机器人怎么用这座库

- 干活前**只读挂载**：`claude -p ... --add-dir <vault>/wiki`，prompt 里写明"**先看 `wiki/INDEX.md` 找相关页，再动手**"。
- **INDEX 优先**：库大了以后不要让模型 grep 全库，`INDEX.md` 的一行钩子就是检索层（一页一行、写清"什么时候该点开它"）。
- **优先级链**（写进 prompt，冲突时逐级让位）：正式口径文档 > 知识库 wiki > 岗位记忆 > 会话上下文。
- 知识库页里凡是"该升级为正式文档"的内容，只写进页面的 **`⚠️待人工确认`** 小节，**agent 绝不擅改正式口径文档**。

### 10.7 调度表（macOS launchd / Linux systemd timer 等价）

| 任务 | 频率 | 说明 |
|---|---|---|
| 收件箱拉取（IM「存」→ 落盘） | 每 10 分钟 | 轻，纯 HTTP + 写文件，不吃模型额度 |
| 夜间消化 | 每天 23:00 | **吃模型额度**：交给账号池里权重低的账号，避开白天工作时段（见 §7.4） |
| 双脑同步 | 每 4 小时 | git pull + scp，不吃额度 |
| 全库体检 lint | 每周一次或手动 | 吃额度，安排在深夜 |

### 10.8 红线（照抄，别自由发挥）

1. **只写原料里有的内容** + 明确标注为 agent 自己的综合判断；**编造 = 事故**，拿不准标 `⚠️待核实`。
2. **消化是无损压缩**：宁可页面写长，不许丢关键事实和数字。
3. `raw/archive/` 与外部记忆软链**绝不修改、绝不删除**——唯一事实源必须可回溯。
4. **机密分级**：只有明确打了机密标签的内容不得进入任何产出；内部敏感（成本/财务/条款）内部不设限，
   但**绝不进入对外公开产出**（媒体稿、开源仓库、官网）。★开源本项目时尤其注意：vault 本身绝不入公开仓。
5. vault 只做**本地 git commit，不配远端、不 push**（除非明确要求）——它是你的原始积累，不是代码。
6. 凭证（同步 token / SSH key）放 `.sync/` 并在 vault 的 `.gitignore` 里排除。

---

## 11. 安全模型（开源前必须实现全部）

| 层 | 措施 |
|---|---|
| 触发面 | 会话白名单（`allowed_channels`）+ 群聊必须精确 @ + 私聊需显式开权限 |
| 通道 | runner↔后端共享密钥；`resource/deliver` 校验 (mid,key) 确属本 job；HTTPS |
| 凭证 | IM 应用密钥只在后端；模型令牌只在 runner env；**两边都不进 git**，日志脱敏 |
| 代码 | 产出型任务 cwd = 临时空目录（物理隔离）；改码型只落分支，**主干仅在管理员确认后合入** |
| 审批 | 审批消息校验管理员 id 白名单；幂等（重复"确认"不重复上线）；私聊来源**强制人工确认**，且审批请求抄送到白名单群（否则群里的人看不见私聊审批） |
| 权限降级 | 私聊来源任务 prompt 注入**只读条款**（要写数据请回群里说 → 群里有留痕） |
| 生产数据 | 默认只读。开写要**两阶段**：先产出变更脚本 + dry-run 预览 → 管理员确认 → 才执行。黑名单写进 prompt：禁 DELETE、禁无主键批量 UPDATE、禁资金/权限/建表/改配置；一律事务+幂等+审计留痕 |
| 审计 | 每单落库：谁问的、跑了什么、改了哪些文件、用了哪个账号、谁批的 |
| 兜底 | `--permission-mode bypassPermissions` 意味着**它在执行端是全权的**——所以执行端应当是一台**专用机器**，不放私人数据、不登录无关账号 |

---

## 12. 运维（"改了什么 → 要重启什么"）

| 改动 | 生效方式 |
|---|---|
| `assets/` 素材/口径/记忆 | **即时**，什么都不用动 |
| 后端代码/固定台词 | 部署后端 + 热重启 |
| runner 脚本 / prompt 模板 / profile 定义 | **重启 runner** |
| 账号池（加账号/换令牌） | 改 env + 重启 runner |
| IM 应用权限/事件 | 平台上改完**必须重新发版** |
| vault 里的 wiki 页/`CLAUDE.md` 规则 | **即时**（下一单挂载时就读到新的） |
| 定时器（拉取/消化/同步频率） | 改 plist / timer 后 `launchctl unload && load`（或 `systemctl daemon-reload`） |

**排障口诀**：群里 @ 了没秒回 → 先看后端访问日志有没有那一刻的 `POST /api/bot/callback`：
**没有 POST = IM 侧没推**（权限/事件/发版三件套没配齐，占九成）；**有 POST 没建单 = 后端问题**（@ 判定/白名单/去重）；
**建了单没结果 = runner 侧**（进程死了/账号全冷却/claude 未登录）。错过的消息可按原 message_id 重放补办。

**必备告警**：runner 心跳丢失、账号全员冷却、任务在"处理中"超 45 分钟（僵尸自愈已触发说明有单死过）。

**知识库侧排障**：说了「存」但库里没有 → 依次看 ① 后端 `brain_item` 有没有这条（口令识别）、② 拉取器日志 `.sync/pull.log` 有没有落盘（token/网络）、③ `raw/inbox/` 里文件还在不在（在=当晚没消化，看 `digest.log`：多半是那台机器 23:00 睡了或 claude 未登录）。

---

## 13. 开源仓库结构与选型（GitHub 现成件）

```
agentbridge/
├─ server/                    # 无模型后端
│  ├─ app.py                  # FastAPI：7 个端点
│  ├─ models.py               # 单表 job（SQLModel/SQLAlchemy）
│  ├─ adapters/               # ★IM 适配器（可插拔）
│  │   ├─ base.py             #   接口：parse_event / send_text / send_rich / upload_image / upload_file / download_resource
│  │   ├─ lark.py             #   飞书（首发实现）
│  │   ├─ slack.py  dingtalk.py  wecom.py
│  └─ config.example.toml
├─ runner/
│  ├─ main.py                 # lane 编排
│  ├─ pool.py                 # ★AccountPool
│  ├─ claude.py               # 子进程封装 + 额度识别 + 超时
│  ├─ lanes/ chat.py code.py memory.py cron.py
│  └─ prompts/                # 各任务类型 prompt 模板（外置成文件，改完重启即生效）
├─ brain/                     # ★知识库层（§10）
│  ├─ pull_inbox.py           #   IM「存」→ 落盘（两段式 pull/ack，原子写）
│  ├─ digest.sh               #   夜间消化：收件箱非空才唤起 claude -p "消化收件箱"
│  ├─ sync_peer.py            #   双脑同步：拉镜像 + 黑名单增量投递
│  ├─ vault_template/         #   ★空 vault 骨架：CLAUDE.md(管理员规则) + raw/inbox + wiki/INDEX.md
│  └─ schedules/              #   launchd plist / systemd timer 样例（10min / 23:00 / 4h）
├─ profiles/example/          # 示例 profile（口径与红线.md / 记忆.md 模板）
├─ docs/  ARCHITECTURE.md  DEPLOY.md  SECURITY.md  KNOWLEDGE_BASE.md
└─ docker-compose.yml         # server + db（runner 不进容器：它要用宿主机的 claude 登录态）
```

**选型建议（都是 GitHub 上的成熟件）**：

| 位置 | 选型 | 理由 |
|---|---|---|
| 后端框架 | **FastAPI** + uvicorn | 端点少、JSON 契约清晰；已有 Django 系统的话直接挂个 app 更省事 |
| ORM/DB | SQLModel/SQLAlchemy + **SQLite**（单机）或 MySQL/Postgres | 单表，SQLite 足够；要多 runner 并发领单再上 MySQL/PG |
| IM SDK | 飞书 **lark-oapi**（官方 Python SDK）；Slack **slack_sdk**；企微/钉钉官方 SDK | 只用到"发消息/传文件/下载资源/验签"少数几个接口，也可以直接裸 HTTP |
| HTTP 客户端 | httpx + tenacity（重试） | |
| 定时 | APScheduler，或就用 lane 里的本地时钟循环 | 定时任务少时不必上调度框架 |
| 执行端 | **Claude Code CLI**（`claude -p`，用订阅额度；本方案核心） | 也可换 Claude Agent SDK / 其它 CLI Agent——把 `runner/claude.py` 换掉即可 |
| 进程守护 | macOS `launchd` / Linux `systemd` / `pm2` | runner 必须开机自启 + 崩溃重拉 |
| 密钥 | `.env` + direnv / 1Password CLI / age 加密 | 令牌绝不进 git |
| 知识库 | **纯文件 + git**（wiki 是 md，索引是 md）；查看用 **Obsidian**（双链/图谱免费拿） | ★别上向量库：库到几百页时 `INDEX.md` + 双链 + grep 就够，agent 检索靠索引页比靠 embedding 更可控、可审计；真到几千页再考虑加检索层 |

**替换性**：`adapters/` 换掉 = 换 IM；`runner/claude.py` 换掉 = 换模型执行器；`profiles/` 换掉 = 换业务；`brain/vault_template/` 之外的 vault 内容不入仓。
四处正交，这是开源后别人能用起来的前提。★**开源仓里只放 vault 骨架和规则，绝不放任何真实 wiki 内容**。

---

## 14. 分阶段落地计划（每阶段有验收标准）

| 阶段 | 内容 | DoD（做到这个才算过） |
|---|---|---|
| **M0 骨架**（0.5d） | 单表 job + 4 个端点（callback/poll/report/heartbeat）+ 最小 runner 单 lane 单账号 | 群里 @ 说"你好" → 30s 内群里收到模型回的一句话 |
| **M1 附件与交付**（1d） | resource 下载 + deliver 上传 + 富文本合并 + 产物降噪过滤 | 发一个 xlsx 说"做成柱状图" → 群里收到一条图文消息 + 一个成品文件，且**没有** `build.py` 之类中间产物 |
| **M2 多机器人与 profile**（1d） | 第二个 IM 应用共用回调 + app_id 路由 + profiles/ + 素材注入 | 两个机器人在同一群，@ 各自只有本人接单；往 `assets/` 丢一份口径文件，下一单立刻遵守 |
| **M3 账号池**（1d） | AccountPool + 额度识别 + 退回队列 + 心跳告警 | 人为把 acct1 令牌置无效 → 任务自动切 acct2 完成；三个账号全冷却 → 群里只提示一次并在恢复后自动跑完 |
| **M4 改码闭环**（2d） | worktree + 分支 + 审批 + 部署命令 + 迁移标注 | @ 说一个真 bug → 出诊断 + 分支 diff + @管理员；回"确认 N"后才合并部署；非管理员回"确认"被挡 |
| **M5 记忆与日报**（1d） | 便签 + 夜间整理 + 反馈吸收 + 定时 lane | 说一句"记住 X" → 次日新会话里它已经知道 X；直接回复它的答案纠正 → 被👍并在当天后续任务里生效 |
| **M6 知识库**（1d） | vault 骨架 + `CLAUDE.md` 规则 + 「存」口令三端点 + 拉取器 + 夜间消化 + 只读挂载 | 群里说「存 <一段结论>」→ 10 分钟内 `raw/inbox/` 出现文件 → 次日库里有一页融合好的 wiki（**不是**新开一页流水账）、`INDEX.md` 有钩子、有一条 git commit；换个机器人问相关问题时它引用到了这页 |
| **M7 双脑同步**（0.5d） | 镜像拉取 + 黑名单增量投递 + 体检口令 | 打了机密标签的页**没有**被投递出去；改一页后 4 小时内对面收到新版；跑一次「体检」能报出至少一处 INDEX 与实际不一致 |

**先做 M0–M3 就已经是完整可用的产品**；M6 是"越用越值钱"的部分（越早开始沉淀越好，建议紧跟在 M3 后面做）；
M4（改码/部署）风险最高，放最后，且默认关闭自动部署。

---

## 15. 给 AI 的一次性复刻 prompt（可直接粘贴）

> 你要实现一个叫 `agentbridge` 的开源项目：把本机已登录的 Claude Code 客户端，变成 IM（首发飞书）群里的多机器人劳动力。
> 严格按以下约束实现，不要自作主张换架构：
> 1. **后端不跑任何模型**，只做：IM 事件回调解析、单表 `job` 任务队列、附件下载/产物上传、审批闸、白名单。用 FastAPI + SQLModel。
> 2. **执行端是独立进程 runner**，轮询后端领单，用 `subprocess` 调 `claude -p <prompt> --permission-mode bypassPermissions --output-format text`。
> 3. 任务表**只有一张**，`kind` 字段格式 `<type>[:<profile_key>]` 同时编码任务类型和机器人身份；`message_id` 唯一做幂等；
>    领单必须是 `UPDATE ... WHERE status=0` 的条件更新按影响行数判断；被领超 45 分钟未回报的自动退回队列。
> 4. **多个 IM 应用共用一个回调 URL**，按 `header.app_id` 路由到 profile；群聊必须精确匹配本应用 bot 的 open_id 才接单；
>    去重键 = `message_id + '#' + bot_key`。
> 5. **多账号池**：账号 = `{CLAUDE_CONFIG_DIR, CLAUDE_CODE_OAUTH_TOKEN}` 一组环境变量，每个 job 通过 `subprocess.run(env=...)` 指定；
>    实现租约式 `AccountPool`（会话亲和 + 权重 + 撞额度冷却 + 全员冷却时任务退回队列并只提示一次）。
> 6. 产出型任务的工作目录必须是 `tempfile.mkdtemp()` 的空目录、结束即删；改码型任务在 git worktree 里开 `bot/job-<id>` 分支，
>    **绝不直接改主干**，合并部署必须等管理员在 IM 里确认。
> 7. 产物收集排除 `inputs/ assets/ scratch/`、隐藏文件和 `.py .sh .js .ts .ipynb .r`；图片暂存后与结论合成**一条富文本**发出。
> 8. `profiles/<key>/` 目录是业务落点：角色 prompt、口径红线、素材、`记忆.md`、`_inbox/`；任务开跑前拷进工作区 `assets/`，
>    `_`/`.` 开头的不拷。**改素材不需要重启任何进程**。
> 9. 全部密钥走环境变量，绝不进 git、不写日志、不发进 IM。
> 10. **知识库层**：一个纯文件 git vault（`raw/inbox/` → 夜间 `claude -p "消化收件箱"` → `wiki/<领域>/*.md` + `INDEX.md` + 一次 commit）；
>     IM 里说「存 …」经 `/api/brain/save|pull|ack` 两段式（**先落盘再 ack**、原子写、文件已存在则跳过）进收件箱；
>     机器人干活时 `--add-dir <vault>/wiki` **只读**挂载并被要求"先看 INDEX 再动手"；
>     消化规则全部写在 vault 自己的 `CLAUDE.md` 里（**优先重写融合进已有页，确无归属才新建**），不要用代码实现消化逻辑。
> 交付：可运行的仓库 + `docs/ARCHITECTURE.md` + `docs/KNOWLEDGE_BASE.md` + `.env.example` + `brain/vault_template/` + M0–M3 的端到端自测脚本。

**验收清单**（逐条实测，不要只看代码）：
- [ ] 同一条消息 @ 两个机器人，两个都回，且各自只建一单
- [ ] IM 重推同一事件，不重复建单、不重复上线
- [ ] 两条 lane 同时轮询，不会领到同一单
- [ ] kill -9 runner 后重启，卡住的单在 45 分钟内自动回到队列并跑完
- [ ] 令牌失效/额度耗尽 → 自动换账号或排队，恢复后自动完成，群里只提示一次
- [ ] 非管理员的"确认"被拒；私聊里的改码任务不会自动部署
- [ ] 临时目录在任务结束后确实被删除；产物里没有中间脚本
- [ ] 往 `profiles/<key>/` 丢一份口径文件，下一单立刻遵守（未重启任何进程）
- [ ] 「存」了一条后立刻断网/杀掉拉取器，恢复后**不丢件也不重复落盘**
- [ ] 同一主题连存三条，夜间消化后是**一页被融合更新**，不是三页新建；`INDEX.md` 只多一行
- [ ] 打了机密标签的页在双脑同步里没有被投出去；`raw/archive/` 里的文件消化后未被改写

---

## 16. 实战坑清单（复刻时直接规避）

1. **@ 了没反应** → 九成是 IM 侧权限/事件/发版三件套缺一，不是代码问题。先查后端有没有收到 POST。
2. **多机器人同群互相吞单** → 去重键没带 bot_key。
3. **markdown 原样刷屏** → IM 的 text 消息不渲染 md，发送前必须 `strip_md`。
4. **私聊图文拆条跑出空需求废单** → 纯附件消息要暂存 10 分钟等下一条文字。
5. **中间脚本刷进群** → 产物过滤 + prompt 里要求"中间文件放 scratch/"，两边都要做。
6. **worktree 里 checkout 主干失败** → 主工作区占着该分支，一律开独立分支。
7. **改码 lane 并行 = git 互斥爆炸** → 改码线全局只能一条。
8. **任务永久卡"处理中"** → lane 里任何异常都必须兜底 report 失败；再加僵尸自愈。
9. **额度耗尽被当成失败** → 必须单独识别并退回队列，否则用户看到的是"机器人坏了"。
10. **prompt 改了没生效** → runner 内存里的模板要重启；素材目录才是即时的。
11. **`ANTHROPIC_API_KEY` 残留在环境里** → 会绕过订阅走计费口，账号池 env 里显式 `pop` 掉。
12. **执行端是你的日常电脑** → `bypassPermissions` 下它权力很大；用一台专用机器，别把私人凭证放它旁边。
13. **知识库退化成流水账** → 消化规则里没写"优先重写融合进已有页"，模型就会每次新建一页；再加每周「体检」清矛盾/孤立页。
14. **先 ack 后落盘 = 丢件** → 两段式必须"全部写盘成功才 ack"，落盘用 `tmp + rename` 原子写。
15. **夜间消化白烧额度** → 收件箱空必须直接 `exit 0`；消化和体检安排在深夜、指定低权重账号（§7.4）。
16. **消化那台机器 23:00 睡了** → 定时任务永远没跑，库看着"没坏但也不长"；给消化任务加"连续 N 天没 commit"告警。
17. **vault 误入公开仓** → 开源时只放 `vault_template/`（骨架+规则），真实 wiki、`.sync/` 凭证一律 gitignore。

---

[← 返回附录导读](README.md) · [← 返回总目录](../README.md)
