# 807 set@color stale-state Reflection

## Task
- 记录 xeCJK issue #807 的修复经验：`\set@color` 的 whatsit 恢复补丁在无节点分支没有清理全局状态，导致首次 `\textcolor` 在标点后或段首错误插入 `ecglue`。

## Expected vs Actual
- Expected outcome.
  - `\textcolor` 只应维持颜色 whatsit 与原有 CJK 间距行为，不应在标点后、段首或其他“前面没有可恢复节点”的位置平白插入 `ecglue`。
  - v3.10.0 为 #315/#803 引入的 `\set@color` 补丁应同时保证“有节点”与“无节点”两条分支都不会污染后续恢复链。
- Actual outcome.
  - 包初始化阶段 `\g_@@_last_node_tl` 会被污染为 `default`；首次 `\textcolor` 恰好出现在标点后或段首时，`\xeCJK_if_last_node:TF` 走 false 分支，但该分支直接调用 `\@@_orig_set_color:`，没有先清空全局状态。
  - 随后的 whatsit 恢复链把这份陈旧状态误当成真实边界标记，触发 `\@@_recover_glue_whatsit:`，于是给颜色组内首个 CJK 字前错误补入 `ecglue`。
  - 同一根因还顺带修复了 ctex beamer 测试里超链接注解 whatsit 后接 CJK 字时的既有伪空白问题。

## What Went Wrong
- 排查 `\set@color` 补丁时，若只关注“有节点时是否正确保存/恢复”，很容易忽略 false 分支本身也会决定全局状态是否泄漏到未来输入。
- 对 `\g_@@_last_node_tl` 这类全局变量的审查不够完整：之前已经知道它承载 whatsit 恢复链的边界类型，但没有逐分支确认每条退出路径是否都做了清理。
- 测试若只覆盖“常规颜色包裹文本”或“连续多次颜色调用”，会错过“包加载后第一次调用”这种只受初始化污染影响的一次性场景。
- 一开始容易把 beamer 基线变化当成另一个独立问题；实际上它和 `\textcolor` 首次调用异常一样，都是 stale state 经过 whatsit 恢复链被误读。

## Root Cause
- 根因不是 xcolor 自身，而是 xeCJK 在 #315/#803 之后引入的全局状态机不完整：`\g_@@_last_node_tl` 作为跨-whatsit 回退依据，必须在“当前确实没有可继承节点”时显式失效。
- `\set@color` 补丁的 no-node 分支把“此刻无事可做”误当成“保留现状即可”，但对全局状态来说，不作为本身就是一种错误动作，因为旧值会跨调用存活并参与下一次判定。
- 该问题带有明显的初始化时序特征：污染值来自包初始化阶段，因此最容易在“第一次颜色 whatsit 介入边界判断”时暴露，而后续调用未必稳定复现。

## Missing Docs or Signals
- memory only:
  - 只要修改或审查 `\g_@@_last_node_tl` 相关逻辑，必须逐分支检查“写入、消费、清空”是否闭环，尤其是 false/else/no-node 分支，不能默认它们是无害路径。
  - 对 whatsit 恢复类补丁，回归测试至少要包含三类触发点：包加载后的第一次调用、普通后续调用、以及“前面本来就没有可恢复节点”的位置（如段首、标点后、链接注解后）。
  - 若 xeCJK 修复触发 ctex beamer 或 `config-contrib` 基线变化，应先优先尝试归并为同一状态机根因，而不是立即拆成新问题。
- promotion candidates:
  - `guides/` 可补一条 xeCJK 状态机/whatsit 修复检查清单，明确全局状态变量必须按所有分支审计，并要求测试首次调用场景。
  - `architecture/` 可在 xeCJK 边界恢复链说明中补充 `\g_@@_last_node_tl` 的生命周期约束：无节点分支也必须清空状态，避免把初始化或前序调用残留误当成当前边界。
  - `reference/build-and-test.md` 可补充 whatsit 回归测试设计经验：除常规场景外，要显式覆盖 first-use-after-load 与下游 beamer/hyperref 触发路径。

## Promotion Candidates
- `architecture/`
  - 补充 `\g_@@_last_node_tl` 的状态生命周期说明，强调其是“最近一次真实边界标记”的缓存，而不是可无限保留的默认值。
- `guides/`
  - 增加 xeCJK whatsit/颜色补丁排查清单：检查所有分支是否清理状态，并强制加入首次调用回归样例。
- `reference/`
  - 在测试参考中加入“首次调用 vs 后续调用”这类初始化污染型回归的设计模式，以及 ctex beamer/hyperref 作为下游联动样例。

## Follow-up
- 后续凡是再改 `\set@color`、whatsit 恢复链或 `\g_@@_last_node_tl` 相关代码，都先按“分支闭环 + 首次调用场景 + ctex 下游 beamer/hyperref 联动”三步检查，并复用 `color-ecglue01.lvt` 这类宽度回归测试模板。