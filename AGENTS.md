# AGENTS.md

本文件是 `qq-farm-bot-ui` 仓库内多位 Codex 提交者的统一执行规范。  
目标：减少“一个小功能改太多文件”的情况，避免编码乱码、日志污染、配置漏改、联调卡住。

## 1. 项目现状基线（2026-03-09）

- 仓库结构：`pnpm workspace`，核心目录为 `core/`（Node.js 后端）与 `web/`（Vue 3 + Vite 前端）
- CI（`.github/workflows/ci.yml`）当前检查：
  - `pnpm install --frozen-lockfile`
  - `pnpm -C web lint`
  - `pnpm -C core exec eslint "src/**/*.js" "client.js"`
  - `pnpm build:web`
- 打包命令：
  - Windows: `pnpm package:win`
  - Linux: `pnpm package:linux`
  - Mac: `pnpm package:mac`
  - Release: `pnpm package:release`
- 配置端口来源：`core/src/config/config.js` 中 `process.env.ADMIN_PORT`（默认 `3000`）

## 2. 全局硬性规则

- 所有新增/修改文本文件必须是 `UTF-8` 编码（建议 UTF-8 无 BOM）。
- 禁止引入与需求无关的改动（包括但不限于：部署配置、构建配置、README、版本号、锁文件、UI 样式大改）。
- 小功能 PR 必须小范围提交，若改动文件较多，PR 描述中必须逐项说明“为什么必须改这些文件”。
- 临时调试代码（临时日志、临时 watchdog、临时超时、临时开关）在合并前必须清理。
- 不得提交会影响默认安全面的改动（如随意放开 host/CORS/鉴权/限流）除非需求明确要求。

## 3. 编码与文本规范

- 中文文案、中文日志允许且推荐使用简体中文。
- 不允许出现乱码文本（如 `鎻愮ず` 这类异常编码内容）进入提交。
- 提交前至少执行一次编码自检（示例）：

```bash
python - <<'PY'
import pathlib
exts={'.js','.ts','.vue','.json','.md','.yml','.yaml','.mjs','.cjs','.css','.html','.txt'}
ignore={'node_modules','.git','dist'}
bad=[]
for p in pathlib.Path('.').rglob('*'):
    if not p.is_file(): continue
    if any(x in p.parts for x in ignore): continue
    if p.suffix.lower() not in exts: continue
    b=p.read_bytes()
    if b.startswith(b'\xef\xbb\xbf'): b=b[3:]
    try: b.decode('utf-8')
    except Exception as e: bad.append((str(p),str(e)))
print('non_utf8:',len(bad))
for x in bad[:50]: print(*x, sep=' | ')
PY
```

## 4. 设置类需求的“必改链路”

凡是 `Settings` 页面新增/修改开关、策略、字段，至少检查以下链路是否完整：

- 前端展示与编辑
  - `web/src/views/Settings.vue`
  - `web/src/stores/setting.ts`
- 后端配置默认值与持久化
  - `core/src/models/store.js`
- 后端配置校验
  - `core/src/services/config-validator.js`
- 运行时配置下发/读取
  - `core/src/runtime/data-provider.js`
  - `core/src/controllers/admin.js`（如涉及接口字段）
- 业务真正生效处
  - 相关 service（如 `core/src/services/farm.js`、`friend.js` 等）

要求：

- 仅改 UI 不改后端默认值，视为不完整。
- 仅改后端不改 UI 字段，视为不完整。
- 新增字段必须有默认值、校验、保存、读取、业务消费全链路。

## 5. 日志规范（重点）

- 业务日志尽量中文，方便线上排查。
- 使用统一日志通道，不要随意 `console.log`：
  - 结构化日志优先（`module/event/result` 等字段）
  - 错误日志包含关键信息但避免泄露敏感数据
- 调试日志规则：
  - 临时加的调试日志必须打上明显标记并在 PR 前删除
  - 不要把“排障日志”长期留在高频路径
- 同一业务要区分日志语义（例如常规施肥 vs 多季补肥），避免混淆。

## 6. 调度与超时规则

- 调度类改动（farm/friend/task/email）必须考虑“长耗时但正常”的场景。
- 禁止硬编码短超时导致误报（例如固定 60s 直接判失败）除非明确有配置项并默认安全。
- 若必须增加 watchdog：
  - 必须可配置
  - 必须说明默认值依据
  - 必须有超时日志且不影响其他模块继续调度

## 7. PR 范围控制与提交要求

- 一个 PR 只做一类事情：
  - 功能改动
  - 修复 bug
  - 重构
  - 文档/配置
  - 上述类型不要混杂
- 以下内容默认禁止混入“小功能 PR”：
  - `web/vite.config.ts`、部署文件（如 `zeabur.json`）、CI workflow、根 README、大量样式重排
- PR 描述必须包含：
  - 背景与目标
  - 改动文件清单（按模块）
  - 风险点
  - 回归验证步骤

建议 PR 模板最小字段：

- `What`: 做了什么
- `Why`: 为什么做
- `How`: 改了哪些模块
- `Risk`: 潜在影响
- `Verify`: 本地如何验证

## 8. 提交前自检清单

- 已执行 `pnpm lint`
- 已执行 `pnpm build:web`
- Settings 改动已走完“必改链路”
- 无临时日志、无临时超时、无临时开关残留
- 无乱码、无非 UTF-8 文件
- 无无关文件改动

## 9. 冲突处理原则

- 若发现工作区存在自己未产生的改动，先停止并确认来源，再继续。
- 不得使用破坏性命令回滚他人改动（如 `git reset --hard`）。
- 发生需求不清时，优先做最小可用改动，并在 PR 中明确假设与边界。

