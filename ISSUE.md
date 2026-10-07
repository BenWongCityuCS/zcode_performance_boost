# Issue 稿（可贴 zai-org/feedback 或官方讨论区；官方仓关闭了 PR，patch 以附件/贴文形式给）

**标题**：fix: 渲染层会话窗口收敛——修超长对话的内存增长与卡顿（附 patch）

## 现象

- 超长对话轮下界面**明显卡顿**（实测 ZCode Desktop 3.14.x，i5-12400 / 64GB 桌面机，排除低配因素）；
- renderer 内存随会话时长**只涨不缩**；
- 单轮长时间干活（多工具调用）期间，重订阅会把**早前的提问从界面上挤走**。

## 根因（源码定位，v3.14.3）

1. `packages/shared/src/zcode-protocol-v4/apply.ts` 的 `row.appended` 只追加无淘汰，`rows.window` 无限增长；
2. `packages/ui/src/v4/conversationTurnNavigatorHelpers.ts` 的宽屏自动 `loadAllOlder` 全量补齐把整个历史拉回渲染进程（`loadAllOlder` 合并绕过收敛）；
3. 快照 `rows.window` 只带尾部 `snapshotTailWindowRows`（60）行，`conversationProjectionStore.ts` 的快照分支**整体替换**——单轮长回合里尾 60 行全是工具输出，任何一次重订阅（帧断档 → forceSnapshot）都把早前提问整段冲掉。

## 修复（3 处改动、纯客户端；协议层 apply.ts 与服务端 rowsRange 分页不动）

- **窗口收敛**（`conversationWindowBound.ts` 新增纯函数）：真人发新提问时，窗口收敛为「最近 10 个真实提问所在的回合」；合成行（goalContinuation 等）不触发；没裁掉不换数组引用。被收敛的行不是删除——`canLoadOlder` 继续成立，滚动到顶的 `loadOlder` 照常分页取回完整历史。
- **快照守卫式尾拼**：同 `logEpoch` 且追加性未破坏（totalCount/末行 rowId 单调、拼回行数 ≤ 快照声称的更早行数）时，把本地窗口更早行拼回尾巴前；否则保持整体替换。输出仍是单次原子 setState 的全新对象。
- **导航自动全量补齐加开关**（现停用）：刻度本就从 `rows.window` 实时派生，全量补齐会把收敛后的窗口重新拉满。停用后刻度 = 已加载窗口内的提问书签（loadOlder 展开时随之变多）；分享视图的目录补齐走独立通道不受影响。**只关这一条，滚动到顶的 `loadOlder` 预取保留**——实测发现把预取一并停用会切断旧历史加载。后续可考虑不拉全量行的轻量 query 目录。

## 验证

- `pnpm typecheck` 通过；`oxlint` 触碰文件 0 警告 0 错误；`architecture:check --changed` 0 违规；
- 新增单测 8 项全过（`packages/ui/test/conversationWindowBound.test.ts`，node:test）：收敛语义（恒等稳定/按提问数/合成行不触发）+ 尾拼守卫（同 epoch 拼接、rewind 放弃、上限放弃）；
- 同等语义在 3.14.4 桌面端二进制上实机实测迭代两轮（keep-N 收敛 + 尾拼 + 预取保留）：首轮发现滚动到顶预取被一并停用会切断旧历史加载，恢复预取后复测通过。最终行为：第 11 个新提问收回旧回合、长回合不挤走提问、滚到顶仍可加载更早消息。

## 附件

`0001-fix-ui-bound-renderer-conversation-window-on-real-us.patch`（`git am` 或 `git apply` 即可）。

## 备注

- 收敛阈值是常量 `CONVERSATION_WINDOW_MAX_QUERY_TURNS = 10`（`conversationWindowBound.ts`），可按需调整或做成设置项；
- 语义取舍写在文件头注释：为什么只在真人提问时收敛、为什么按提问数不按行数、为什么恒等稳定。