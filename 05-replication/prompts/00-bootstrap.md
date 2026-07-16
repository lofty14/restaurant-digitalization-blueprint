# M1 骨架:框架 / 响应约定 / 鉴权 / 定时

> 复刻路线的第一步:从空目录搭出可跑的项目骨架,并把全部全局约定一次定死。适合准备动手的工程师,以及被喂进 AI 编程助手直接执行。

**读完你会知道:**

- 为什么第一个里程碑不写任何业务,只定约定——以及哪几条约定日后最贵
- 一份可以整块复制、喂给 AI 编程助手的骨架搭建指令
- 时区、定时系统、响应封装这三个「第一天不定、后面天天还债」的决策怎么定
- 一份可勾选的验收清单,跑通了才算 M1 完成

## 目标

空目录 → 一个能启动、能登录、能跑定时任务、能导出 CSV 的 Django 项目骨架。

M1 不交付任何业务功能。它交付的是**约定**:响应格式、入参解析、分页、鉴权、权限、时区、定时系统、文档结构。我们的经验是,这些约定只要在第一个里程碑没定死,后面每个模块都会各写一套,最后花在「统一」上的时间比当初定约定多十倍。定时任务双系统并存就是我们踩过的真实教训(详见[定时任务:双系统并存的教训](../../01-architecture/scheduled-jobs.md))——所以这份指令里,定时系统从第一天起就只有一套。

## 前置依赖

无。这是整条复刻路线的起点。

建议先读完 [01-architecture 架构层](../../01-architecture/README.md),理解「为什么单体 Django 够用」再动手;但不读也不影响执行下面的指令。

## 喂给 AI 的指令

把下面整块复制给你的 AI 编程助手。指令用第二人称直接对 AI 说话。

````markdown
你要从一个空目录开始,搭建一个餐饮连锁管理系统的后端项目骨架。这一步不写业务,
只把全局约定定死。以下每一条都是约定,不是建议;定下后写进文档,之后全项目遵守。

## 1. 技术栈与基础设施

- 新建 Django 项目(选一个 LTS 版本),函数式视图风格,不用 DRF,不用 class-based view。
  路由集中在项目级 urls.py,统一 /api/ 前缀,按业务域分段(如 /api/shop/、/api/mall/)。
- MySQL 作主库:字符集 utf8mb4,SQL 模式开 STRICT_TRANS_TABLES 严格模式。
- Redis 按库分工,一开始就分好,别混用:
  - db0:Celery broker
  - db1:Django 缓存 + session(SESSION_ENGINE 走缓存后端,session 不落数据库)
  - db2:预留(将来 WebSocket / Channels 用)
- Celery:配置 worker(异步任务)+ Beat(定时调度)。
  **Beat 是本项目唯一的定时任务系统。** 不引入 django-crontab,不用系统 crontab,
  所有定时任务只注册到 CELERY_BEAT_SCHEDULE 一处。这是红线。
- 时区决策(定死并写进文档):
  - USE_TZ = False,TIME_ZONE 设为本地时区(如 Asia/Shanghai)。
  - 模型时间字段一律本地 naive datetime(default 用 datetime.datetime.now,不用 timezone.now)。
  - Beat 写法约定:CELERY_BEAT_SCHEDULE 里的小时/分钟按本地钟面时间直接写,
    几点触发就写几点,不做任何 UTC 换算。
  - 把以上三条原样记录到 docs/CALIBER.md,注明「全项目时间口径:本地 naive,禁止混入 aware datetime」。

## 2. 统一响应封装

- 写一个响应函数 to_resp(resp),所有视图的返回都经过它,不允许裸 JsonResponse 散落各处。
- code 约定:成功 = 字符串 '0';失败 = 负数。失败值你必须在字符串负数和整数负数里
  **选定一种**(建议整数,-1 通用错误、-2 未登录、-3 无权限……),写进 docs/CALIBER.md,
  之后全项目不允许两种混用。老项目里两种混存的排查成本,我们领教过。
- 响应结构:{"code": ..., "message": ..., "data": ...}。message 写给人看的话;
  data 里的字段平铺,前端拿到就能直接取,不要再包一层嵌套壳。
- 自定义 JSONEncoder:
  - datetime → 'YYYY-MM-DD HH:MM:SS' 字符串,date 同理;
  - Decimal → float。
- 浮点出参一律 round(建议保留 2 位)后再出。不允许裸 float 直接进 JSON——
  它会产生 0.30000000000000004 这类脏值,还会触发部分前端 JSON 解析库对长浮点的兼容问题。
  这条也写进口径文档。

## 3. 统一入参与分页

- 写 loads_data(request):POST 时解析 JSON body 并返回 dict;GET 或解析失败一律返回 {}。
  所有视图从它取参,不直接碰 request.body。
- 分页约定(写进口径文档):
  - 入参:page(从 1 开始)、page_size;
  - 出参:列表数据 + total(总条数);
  - 写成一个公共分页函数,所有列表接口调它,禁止各接口手写切片。

## 4. 双鉴权装饰器

- 总部用户鉴权(session):
  - 登录接口校验后把用户标识写入 session(session 在 Redis);
  - 提供 get_current_user(request) 从 session 取当前用户;
  - 提供装饰器 @require_login,未登录返回统一的「未登录」失败 code。
- 门店端鉴权(独立一套):
  - 门店端账号**不在总部用户表里**,单独建表、单独登录、单独装饰器
    (如 @require_store_login)。
  - 两套装饰器绝不混用:店端接口套了总部装饰器会全线「未登录」。
    把「门店端账号不在总部用户表」写进 docs/CALIBER.md,这是后续所有店端接口的红线。
- 权限模型:
  - 一张布尔权限表:一行对应一个用户,一个布尔字段对应一个权限开关
    (如 mall_permission、site_permission,后续里程碑会不断加字段);
  - 一个统一入口 check_permission(request, perm_name, admin=False):
    先校验登录,再校验对应布尔字段;
  - 视图里只调这个 check 函数,不允许散落手写 if 判权限。

## 5. 基础设施三件套

- 错误追踪:接入一个错误追踪服务(自建或云版皆可),trace 采样率调低,
  不能为了监控拖慢请求。
- CSV 导出工具函数:全项目导出统一走它;内容开头写入 UTF-8 BOM
  (字节 EF BB BF;Python 里即 '\ufeff' 前缀,或直接用 encoding='utf-8-sig'),
  否则 Excel 打开中文必乱码;不要依赖 xlsx 类库(生产容器未必装得上,CSV 最稳)。
- 操作日志表:字段至少含操作人 id、动作名、目标对象、时间、简要参数;
  配一个公共记录函数,后续所有关键写操作都调它。

## 6. 项目文档:CLAUDE.md 与口径文档

- 在项目根目录创建 CLAUDE.md 初版,至少包含:
  - 项目一句话定位与技术栈;
  - 启动命令(runserver / migrate / celery worker / celery beat);
  - 统一响应与入参约定(指向具体函数名);
  - 双鉴权约定与权限 check 入口;
  - 定时任务约定:只有 Celery Beat 一套,新任务加在哪;
  - 时区决策,以及指向 docs/CALIBER.md 的链接;
  - 一条永久规则:**docs/CALIBER.md 是全项目唯一的口径文档**——后续任何模块的新口径
    (订货、库存、成本、经营分……)一律以「新增一节」的方式写进这一份,
    不允许另建第二份口径文档,路径与文件名(含大小写)以这里为准。
  以后每完成一个里程碑,你都要回来更新这份文件。
- 创建 docs/CALIBER.md 作为口径文档——文件名与路径从此定死,全项目只有这一份,
  后续里程碑只往里加小节。第一批口径就是本指令定下的:
  失败 code 的类型选择、时区三条、分页约定、浮点 round、门店账号不在总部用户表。
  CLAUDE.md 里必须引用它。

## 7. 交付一个 demo 验证闭环

骨架搭完,交付以下最小 demo 供验收(全部走统一封装与装饰器):

- 一个总部登录接口 + 一个需要某布尔权限的 demo 接口;
- 一个门店端登录 + 一个店端 demo 接口;
- 一个带分页的列表 demo 接口(数据可以造几条假的);
- 一个 Beat 定时任务:每天上午 9 点(示例时间)打一行带时间戳的日志,
  用于验证本地时区触发正确;
- 一个 CSV 导出 demo 接口,导出含中文的两行数据(示例数据即可)。
````

## 踩坑与红线

这四条都是我们真实踩过、且都发生在「骨架期没定死约定」的场景里:

- **定时任务改了不生效 / 跑两遍**
  症状:改了任务时间却按老时间跑,或同一任务莫名执行两次。
  根因:cron 与 Beat 两套定时系统并存,改任务时只改了一套。
  铁律:从第一天起只用 Celery Beat 一套;历史包袱有多疼见[定时任务:双系统并存的教训](../../01-architecture/scheduled-jobs.md)。

- **定时任务差 8 小时**
  症状:定在上午 9 点的任务凌晨 1 点跑了。
  根因:USE_TZ / Beat 时区配置与「按本地钟面写时间」的理解不一致。
  铁律:USE_TZ=False + 本地时区,Beat 时间按本地钟面直接写,并把这条决策白纸黑字写进口径文档——口头约定活不过三个月。

- **CSV 导出中文乱码**
  症状:导出文件用 Excel 打开全是乱码,用编辑器打开却正常。
  根因:无 BOM 的 UTF-8,Excel 按本地编码瞎猜。
  铁律:CSV 出口只有一个工具函数,永远带 BOM(即 `\ufeff`,字节 EF BB BF);别赌生产环境装了 xlsx 库。

- **前端拿到 12.300000000000001**
  症状:页面偶发显示超长小数,个别前端 JSON 库解析报错。
  根因:后端裸 float 直接进 JSON。
  铁律:浮点出参一律 round,封装在统一响应层,不指望每个视图自觉。

## 验收清单

逐条勾完才算 M1 完成。验不过的项,把现象贴回给 AI 让它修,不要手工绕过。

- [ ] `python manage.py migrate` 无报错,`runserver` 能起
- [ ] Celery worker 与 Beat 都能启动;把 demo 定时任务临时改到「2 分钟后」的本地钟面时间,能准点触发,日志时间戳是本地时间(验完改回)
- [ ] 总部登录接口通,session 存在 Redis 里(重启 Redis 后登录态消失,即为证)
- [ ] 权限 demo 接口:有权限的用户拿到 code `'0'`;无权限拿到选定的统一失败 code
- [ ] 门店端 demo 接口:用总部 session 访问被拒,店端鉴权通过才放行
- [ ] 任意接口响应:datetime 已是 `YYYY-MM-DD HH:MM:SS` 字符串、浮点已 round、data 字段平铺
- [ ] 分页 demo:传 page/page_size 生效,返回带 total;不传分页参数时行为符合口径文档的约定
- [ ] CSV 导出的文件用 Excel 直接打开,中文不乱码
- [ ] 手动触发一条测试异常,错误追踪平台能收到
- [ ] 操作日志表里能查到 demo 操作的记录
- [ ] 根目录有 CLAUDE.md、docs/CALIBER.md 存在,CLAUDE.md 引用了口径文档,且登记了「docs/CALIBER.md 是全项目唯一口径文档」这条规则;时区决策与 Beat 写法约定在文档里能找到原文

## 延伸阅读

- [技术选型与取舍:为什么单体 Django 够用](../../01-architecture/tech-stack.md) — 本页选型的完整论证
- [定时任务:双系统并存的教训](../../01-architecture/scheduled-jobs.md) — 为什么这页只允许 Beat 一套
- [后端坑:时区 / 迁移 / 连接 / 序列化](../../03-pitfalls/backend.md) — 骨架期约定所防的坑的完整版
- [数据口径:最贵的一类坑](../../03-pitfalls/data-caliber.md) — 口径文档为什么值得从第一天建
- [CLAUDE.md:给 AI 的入职手册](../../04-ai-engineering/claude-md-practice.md) — CLAUDE.md 初版之后怎么长大
- 下一步:[M2 核心域:门店 / 员工 / 权限](01-core-domain.md)

---

[← 返回本层目录](../README.md) · [返回总目录](../../README.md)
