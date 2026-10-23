# Bug 修复说明（v0.7.6）

## 标题

剪贴板导入时标签数量超过 20 个，整单导入失败——与解析器自身的容错契约及客户端镜像实现不一致。

## 证据链

`lib/clipboard-parser.js` 的 `parseClipboardAnnotation` 是为「AI 生成的剪贴板内容」设计的容错解析器，其模块头与函数注释明确承诺：对超长备注、被丢弃的标签给出 warnings，**而不是让导入失败**（"Surfaces a list of warnings for truncated notes and dropped tags instead of failing the import"）。实测基线（上游 0.7.5，HEAD `b5f1887`）行为：

| 输入 | 基线结果 |
| --- | --- |
| 25 个合法标签 + 备注 | `ok: false`，error =「每个会话最多 20 个标签」，备注与全部合法标签一并丢失 |
| 2500 字符备注（上限 2000） | `ok: true`，备注截断到 2000 + `{kind: "truncated"}` 警告 |
| 单个 40 字符标签（上限 32） | `ok: true`，该标签被丢弃 + `{kind: "rejected"}` 警告 |

即同类「超限」里唯独标签**数量**超限会让整单导入失败。用最小脚本（直接调用 `parseClipboardAnnotation(JSON.stringify({tags: 25 个合法标签, note: "ok note"}))`）在修复前复现 `ok: false`，`git stash` 回到基线复跑结果一致；修复后同一脚本 `ok: true`、保留前 20 个标签、warnings 为 `[{field: "tags", kind: "truncated", from: 25, to: 20}]`。

佐证预期行为：浏览器端内联镜像实现 `parseImportedAnnotation`（`lib/client.js`）对同一情形早已做截断处理——`if (tags.length >= TAG_LIMITS.tags) { warnings.push({ field: "tags", kind: "truncated" }); continue; }`。两份镜像实现行为分歧，`lib/clipboard-parser.js` 一侧为缺陷方。既有测试 `test/clipboard-parser.test.mjs` 已有用例钉住「drops invalid tags instead of failing」，且 20 个标签整单通过，但没有覆盖 >20 的情形。

## 根因

`normalizeStringArray()`（`lib/clipboard-parser.js`）逐项做了去空、长度/非法字符拒绝（丢弃+警告）、大小写去重，但从未执行 `ANNOTATION_LIMITS.tags`（20）的数量上限；随后 `parseClipboardAnnotation` 末尾调用 `normalizeAnnotationPatch(patch)` 复核时，上限校验抛错并被转成 `{ ok: false }`，整单导入被拒。

## 改动文件

- `lib/clipboard-parser.js`（唯一逻辑改动，+8 行）：在去重之后、入列之前补数量上限检查——已达 20 个时多余标签计入 dropped 并跳过，循环结束后统一补一条 `{field: "tags", kind: "truncated", from, to}` 警告（与备注截断的警告形状一致；检查顺序与客户端镜像实现一致：先去重、后截断）。
- 版本同步（0.7.5 → 0.7.6，无逻辑变化）：`package.json`（version、`repository.url` 改指本修复版仓库）、`lib/client.js` 的 `CURRENT_VERSION` 常量、`lib/index.js` 的 buildId 与三处 `pluginVersion()` 兜底字面量。
- `test/dsh-adapter.test.mjs`：其中 `pluginVersion()` 用例断言的是从 package.json 读到的**当前发布版本号**（版本钉子），随本次版本号一并更新期望值 0.7.5 → 0.7.6；上游其余测试一字未改。
- `README.md`：顶部追加 lhg 修复版说明头，上游原文一字未删。
- 新增本文件与 `UPSTREAM.md`；`LICENSE` 原样保留。

## 复验结果

- 最小复现脚本：修复前 `ok: false`（stash 基线复跑一致）→ 修复后 `ok: true`，保留前 20 个标签、1 条 truncated 警告；超长备注、超长单标签两个对照场景行为不变。
- 全量测试：`npm test` 629/629 通过（与修复前基线计数一致，零回归）；`npm run check`（语法与入口导入检查）通过。

## 验证边界

- 未在真实 DSH 宿主/浏览器中做端到端导入验证；修复范围仅为 Node 侧 `lib/clipboard-parser.js`。客户端内联镜像实现本来就已正确截断，本版未改动它。
- 20 个及以内标签的导入行为与基线完全一致；超限时保留的是**先出现**的前 20 个（去重后顺序），与客户端镜像实现一致。
- 上游 open issue 仅有 2 条功能请求（#30、#35），无此 bug 的报告；本修复基于静态审查 + 最小复现 + 两份镜像实现对照得出。
