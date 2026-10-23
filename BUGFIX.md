# BUGFIX — 错误分类器漏检空格形式的永久性失败

## 标题

`isTransientFailure` 的短语模式只容忍 `_`/`-` 分隔符：仅带自然语言消息（无 code/status）的永久性失败——如 "Invalid API key"、"Bad Request"、"Model not found"——被误判为临时性失败，引擎会对永远重试不成功的错误自动续聊。

## 证据链（失败 → 通过）

复现脚本（用仓库自带 esbuild 打包真实 `src/shared/core.ts` 后调用真函数；失败事实按 `parseFailureFacts` 的真实兜底形状构造：载荷只有 message 时 code 兜底为 `'UNKNOWN'`、无 status）：

修复前（上游 0.15.0，HEAD `ad1939c11d7c02ae3b71607b3baf2f50dc8f896e`），12 个用例中 7 个误判：

```
FAIL  transient=true expected=false  [认证失败(空格形式)] Invalid API key provided
FAIL  transient=true expected=false  [OpenAI 风格认证失败消息] Incorrect API key provided: sk-****
FAIL  transient=true expected=false  [400 请求非法(空格形式)] Bad Request
FAIL  transient=true expected=false  [请求非法(空格形式)] invalid request: messages is required
FAIL  transient=true expected=false  [模型不存在(空格形式)] Model not found
FAIL  transient=true expected=false  [模型不存在(空格形式)] unknown model: foo-1
PASS  transient=false expected=false  [代码形式对照] invalid_api_key / model_not_found
PASS  transient=true expected=true   [限流/网络/502/临时不可用 对照组]
7 case(s) misclassified
```

修复后同一脚本：`ALL PASS`（12/12），含全部真临时性对照组（限流、网络错误、502 Bad Gateway、temporarily unavailable 仍判 transient，未被误伤）。

## 根因

`src/shared/core.ts` 的 `isTransientFailure` 用一组正则识别永久性失败。其中跨词短语有两种写法：用 `.*` 连接的（`insufficient.*(balance|quota)`、`token.*limit`、`context.*(length|…)`）天然兼容空格；而用 `[_-]?` 连接的（`api[_-]?key`、`invalid[_-]?request`、`bad[_-]?request`、`unknown[_-]?model`、`model.*not[_-]?found`、`model[_-]?not[_-]?found`）只匹配代码式写法（`invalid_api_key`、`model_not_found`），普通英文消息里的空格分隔形式全部漏检。带 code/status 的失败通常被代码形式或 401/403 短路救下，但经网关/中转、或从存储会话错误文本还原出的失败事实经常只有 message，此时分类完全依赖这些短语模式，漏检即误判为可重试。

## 改动文件

- `src/shared/core.ts`：三条正则中 6 处短语连接符 `[_-]?` → `[\s_-]?`（允许空格作分隔符；严格超集，原有代码形式匹配不变）。其余逻辑一字未动。
- `lib/index.js`、`lib/client.js.map`：按上游 `npm run build` 重新生成的入库构建产物（客户端 bundle 将该函数 tree-shake，`lib/client.js` 内容不变，仅 map 的 sourcesContent 随源码更新）。
- `package.json` / `package-lock.json`：version 0.15.0 → 0.15.1；repository.url 改为本仓库。

## 复验结果（修复后，基线同为全绿）

- 最小复现脚本：12/12 PASS（修复前 7 FAIL）。
- `npm test`：exit 0（fail 0）。
- `npm run test:runtime`：exit 0（五版 DSH fixture 运行时测试全过）。
- `npm run typecheck`：exit 0。
- `npm run build`：exit 0，产物与源码一致。

## 验证边界

- 本修复只覆盖作者内置短语集的「空格形式」漏检；短语集之外的表述（如 "The model does not exist" 中的 "does not exist"）不在内置模式内，本版未扩充短语集——这类失败仍依赖 code/status 或用户在设置中填写的专属模式。
- 未在真实 DSH 宿主 + 真实供应商账号下做端到端触发验证；结论基于纯函数复现与上游全量测试套件（含五版 fixture 的运行时模拟）。
- 上游 issue #64（settings/conflict）在 HEAD 已由上游 PR #65 修复，未作为本版目标。
