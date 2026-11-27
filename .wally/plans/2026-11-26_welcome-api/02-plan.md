# 02. Plan

## 方案与改动范围

关联判据：为 nova / cinder 各新增 `GET /welcome` REST 端点，组件间相互调用、各自日志打印 `welcome`，并在 awesome-architecture 记录能力。

风险等级：**中**。理由：跨组件（nova ↔ cinder）HTTP 调用，引入相互依赖与潜在循环，但接口为新增、不改既有契约与数据结构、可独立回滚。

最小改动文件清单：

nova 仓库：
- `nova/nova/api/openstack/compute/welcome.py`（新增）：`WelcomeController(wsgi.Controller)`，`index(req)` 打印 `LOG.info('welcome ...')`，调用对端 cinder 的 welcome；`create_resource()` 返回 `wsgi.Resource`。
- `nova/nova/conf/welcome.py`（新增）：`welcome_group` 定义 `cinder_welcome_url`（`cfg.StrOpt`，默认回环地址 `http://127.0.0.1:8776/v3/<project>/welcome`）；日志与超时配置。若项目 conf 汇总不便，改在 `welcome.py` 内以 `CONF.register_opts` 注册，实现时按现有风格定夺。
- `nova/nova/api/openstack/compute/routes.py`（改）：import welcome 模块、新增 `welcome_controller` partial，在 `ROUTE_LIST` 增加 `('/welcome', {'GET': [welcome_controller, 'index']})`。
- `nova/nova/tests/unit/api/openstack/compute/test_welcome.py`（新增）：UT。

cinder 仓库：
- `cinder/cinder/api/v2/welcome.py`（新增）：`WelcomeController(wsgi.Controller)`，`index(req)` 打印 welcome 日志，调用对端 nova 的 welcome；`create_resource()`。
- `cinder/cinder/api/v3/welcome.py`（新增）：同上，供 v3 router 使用；若两版本逻辑一致，可由 v2 模块复用，实现时确定。
- `cinder/cinder/api/v2/router.py`、`cinder/cinder/api/v3/router.py`（改）：import welcome，`mapper.connect('welcome', '/welcome', controller=..., action='index', conditions={'method': ['GET']})`。
- cinder 对端地址与超时配置项：按 cinder 现有配置组织方式新增（如 `cinder/common/config.py` 或 welcome 模块内注册）。
- `cinder/cinder/tests/unit/api/v2/test_welcome.py`、`cinder/cinder/tests/unit/api/v3/test_welcome.py`（新增）：UT。

awesome-architecture 仓库：
- `README.md`（改）：新增“Welcome 接口”小节，说明 `nova` / `cinder` 的 `/welcome` 端点、相互调用与日志打印行为。
- `.wally/plans/2026-11-26_welcome-api/`（本任务 SOP 产物，随提交纳入）。

## 调用链与兼容性

调用链：
- 外部 `GET /nova/welcome` → nova `WelcomeController.index` → `LOG.info('welcome ...')` → HTTP `GET <cinder_welcome_url>` 且携带转发标记 → cinder `WelcomeController.index` → `LOG.info('welcome ...')` → 检测到转发标记，**不再回呼** → 返回 welcome 响应 → nova 返回 welcome 响应。
- 反向同理：外部 `GET /cinder/welcome` → cinder 打印并回呼 nova → nova 打印并终止。

防循环（关键设计）：转发标记采用请求头 `X-Welcome-Forwarded: 1`。主动回呼时设置该头；被调用方 `index` 检测到该头后只打印日志、不再回呼。一次外部请求最多产生一跳对端调用，避免无限递归。

兼容性：
- 新增路由与模块，不改动既有 REST 契约、数据结构或响应格式；对现有接口无影响。
- 响应体保持简单：`{'welcome': 'welcome', 'service': '<nova|cinder>'}`（JSON，ASCII，Unicode 风险低，但 UT 仍覆盖中文对端消息场景）。
- 对端地址、超时通过配置项提供，默认回环地址；不含任何 `*.woa.com` 或内部平台地址。

异常 / fallback / 多终态：
- 对端 HTTP 超时 / 连接失败：`requests` 设置 `timeout`，捕获异常，记录 `LOG.warning`，本地接口仍返回成功（降级），不向调用方传播对端错误。
- 配置为空：跳过回呼，仅本地打印，视为可用。
- 副作用时机：日志在每次 `index` 进入时打印；回呼在打印之后、返回之前执行。

回滚：
- 删除新增 controller / conf / UT 文件，移除 `routes.py`、`router.py`、`README.md` 中的新增行；各仓库分支重置即可，无数据迁移。

## UT 设计

nova `test_welcome.py`：
- W01 正向：`GET /welcome` 无转发标记 → 断言返回 200、响应含 `welcome`、`LOG.info` 被调用、对端 `requests.get` 被调用且 URL 来自配置项。
- W02 边界（防递归）：请求带 `X-Welcome-Forwarded: 1` → 断言打印日志但**不**调用对端。
- W03 异常恢复：对端 `requests.get` 抛 `RequestException` → 断言本地仍返回成功、`LOG.warning` 被调用。
- W04 Unicode：对端返回中文消息串 → 断言本地响应仍可正常序列化（覆盖 JSON/中文场景）。

cinder `test_welcome.py`（v2、v3 各一组或参数化）：
- C01 正向：`GET /welcome` 无标记 → 200、打印 welcome、回呼 nova。
- C02 边界：带转发标记 → 打印但不回呼。
- C03 异常：对端失败 → 降级返回成功。
- C04 Unicode：中文消息正常处理。

测试位置遵循现有目录：`nova/nova/tests/unit/api/openstack/compute/`、`cinder/cinder/tests/unit/api/v2/` 与 `v3/`。UT 命令：按项目现有入口（优先 `tox -e py35` / `ostestr` 单模块），实现时记录真实命令与退出码。

## IT 设计（六步适用）

04 Deploy / 05 IT 未配置，任务创建时已标记 SKIPPED，本任务不执行环境级部署与集成验证。保留代码层 UT 作为主要验证。未验证边界：真实跨组件 HTTP 联通性、日志落盘位置、部署配置生效，需在具备环境时另行验证；不得声称端到端已验证。

## 风险与完成结论

关键设计决定：
- 采用转发请求头 `X-Welcome-Forwarded` 防止相互调用导致的无限递归（待用户确认）。
- 对端地址走配置项、默认回环地址，符合“不硬编码内部地址”约束（待用户确认）。
- 对端故障降级，本地接口仍成功（待用户确认）。

未覆盖边界：真实环境跨组件联通与部署配置（04/05 SKIPPED）。
授权需求：Commit / Push / 创建 MR 需在实现、UT 通过后另行获得用户明确授权。

完成结论：方案已基于真实源码设计，UT 先于实现设计；待用户确认上述三项设计决定后进入 03 Implement。

## 知识库证据

沿用 01-clarify 中已记录的源码核验路径（routes.py、router.py、versions.py、limits.py、volumes.py、requirements.txt、tox.ini 等）。无仓库内 `.wally/docs/`，使用系统默认 SOP；已核对现有文档与平台知识。方案依据为实际读取的路由注册、controller、日志与依赖写法，未凭对话推测。最终 Review 对照完整 diff 复查。
