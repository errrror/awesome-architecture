# 01. Clarify

## 目标与成功判据

在 `nova` 与 `cinder` 两个组件各新增一个 `welcome` REST API 端点；组件间相互调用对方的 `welcome` 接口，每个组件在处理请求时于自身日志打印 `welcome`；在 `awesome-architecture` 文档记录该能力。

成功判据：
1. `nova` 的 `GET /welcome` 返回成功响应，并在 `nova` 日志打印 `welcome`。
2. `cinder` 的 `GET /welcome` 返回成功响应，并在 `cinder` 日志打印 `welcome`。
3. 访问 `nova` 的 `welcome` 会触发对 `cinder` 的 `welcome` 调用，`cinder` 日志出现 `welcome`；访问 `cinder` 的 `welcome` 会触发对 `nova` 的 `welcome` 调用，`nova` 日志出现 `welcome`。
4. 调用链不会无限递归（有转发标记终止条件）。
5. 单元测试覆盖正向、转发标记边界、对端异常三类场景，全部通过。

## 范围、约束与依赖

范围：
- 三个仓库均基于 `master` 新建 `task/welcome-api` 分支后实施。
- `nova`、`cinder`：新增 controller、路由注册、对端地址配置项、单元测试。
- `awesome-architecture`：`README.md` 增加该能力的说明；同时承载本任务的 SOP 产物。

约束：
- 不修改目标仓库 `AGENTS.md`，不写入 `.wally/versions/`。
- 对端地址走配置项，**不硬编码**任何 `*.woa.com` / 内部平台地址；默认值仅使用回环地址。
- 日志沿用各组件现有 `logging.getLogger(__name__)` + `LOG.info` 模式。
- 04 Deploy / 05 IT 未配置，任务创建时已标记 SKIPPED，本任务不编造环境。

依赖：
- `nova`、`cinder` 的 `requirements.txt` 均已包含 `requests>=2.14.2`，无需新增运行依赖。
- 两组件均为 Python 2.7 / 3.5 时代的 OpenStack 代码，需保持六 / 兼容写法。

## 问题与决策

以下决策来自用户对本任务澄清卡片的明确回复（输入来源：用户 2026-11-26 回复）：

1. 分支：新建任务分支（`task/welcome-api`）。
2. 范围：`nova` + `cinder` + `awesome-architecture`。
3. 接口形式：新增 REST API 端点。
4. 组件行为：相互调用对方接口，被调用方在各自日志打印 `welcome`。

设计层假设（进入 Plan 后由用户确认）：
- 防递归：请求携带转发标记，被调用方见到标记后仅打印日志、不再回呼对端，避免无限循环。
- 对端地址：`nova` 配置 `cinder` 的 welcome URL，`cinder` 配置 `nova` 的 welcome URL，均通过 `oslo.config` 配置项提供。
- 降级：对端调用失败时记录 warning，本地接口仍返回成功，不因对端不可用而失败。

## 问题判据（缺陷 / 性能任务适用）

本任务为功能新增，非缺陷 / 性能任务，不适用。

## 完成结论

目标与验收判据可验证；业务层四项关键决策已由用户确认。防递归与对端配置属方案层取舍，已在 02 Plan 展开并待用户确认。风险：跨组件 HTTP 调用依赖对端可达，已通过超时与降级控制。

## 知识库证据

使用系统内置默认 SOP，无仓库内 `.wally/docs/`；结合对话已选平台知识与仓库源码核验。实际读取路径：
- `awesome-architecture/README.md`：纯文档仓库，确认可承载能力说明。
- `nova/nova/api/openstack/compute/routes.py`：`ROUTE_LIST` 注册机制、`APIRouterV21`。
- `nova/nova/api/openstack/compute/versions.py`：WSGI controller 返回形态。
- `nova/nova/api/openstack/compute/services.py`：controller 组织方式。
- `nova/requirements.txt`、`nova/tox.ini`：HTTP 依赖与 UT 入口。
- `cinder/cinder/api/v2/router.py`、`cinder/cinder/api/v3/router.py`：`mapper.connect` 注册方式。
- `cinder/cinder/api/v2/limits.py`：`create_resource()` 与 controller 形态。
- `cinder/cinder/api/v2/volumes.py`：`LOG.info` 日志模式。
- `cinder/requirements.txt`、`cinder/tox.ini`：HTTP 依赖与 UT 入口。

缺口：无部署 / 集成测试环境配置（04/05 SKIPPED）。受影响文档：`awesome-architecture/README.md`，计划在 03 阶段同步更新。
