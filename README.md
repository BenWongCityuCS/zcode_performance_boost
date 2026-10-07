# ZCode 长会话性能补丁（渲染层窗口收敛）

**一条 `git am`，修掉 ZCode 超长对话的三大问题**：CPU 占用高、内存占用多、界面卡顿——顺带修「长回合把早前提问挤走」。基于官方开源仓库 [zai-org/ZCode](https://github.com/zai-org/ZCode)（Apache-2.0）的源码级补丁——定位到根因、带单测、可直接应用。

> **非官方项目**，与 Z.ai / ZCode 无隶属关系。本仓库只包含补丁（diff）与文档，**不含上游源码树或任何二进制**；「ZCode」名称仅用于指代上游项目。补丁以 Apache-2.0 发布，按现状提供，不附带任何担保。仅供学习研究。

**English summary**: A single `git am` patch against the official open-source [zai-org/ZCode](https://github.com/zai-org/ZCode) that fixes three long-conversation issues — high CPU usage, renderer memory growing over time, and UI lag — plus a correctness bug where earlier prompts get displaced during long tool-calling turns. Root causes identified at source level, unit-tested, client-only (protocol and server untouched). Full details below in Chinese.

## 适合谁

- **从源码构建 ZCode 的开发者**，以及想看修复方案/根因分析的人——直接可用；
- **只安装了官方桌面版的终端用户**——用不上：本补丁作用于源码树，不是安装包补丁。可以等官方修复，或把 [ISSUE.md](ISSUE.md) 转给官方反馈渠道。

## 问题（实测）

长会话下 ZCode 的资源占用与操作体验一起劣化：

- **CPU 占用高，内存占用随会话持续上涨**——内存不足的机器上尤为明显；
- 光吃资源也就罢了，**界面还卡**：超长对话轮下操作明显迟滞——而且不挑配置，12 代 i5 + 64 GB 上照样卡；
- 除资源与体验外还有正确性问题：单轮长时间干活（多工具调用）期间，重订阅会把**早前的提问从界面挤走**。

## 根因（源码定位，zai-org/ZCode v3.14.3）

| # | 位置 | 机制 |
|---|---|---|
| 1 | `packages/shared/src/zcode-protocol-v4/apply.ts` | `row.appended` 只追加、无淘汰，`rows.window` 无限增长 |
| 2 | `packages/ui/src/v4/conversationTurnNavigatorHelpers.ts` | 宽屏自动 `loadAllOlder` 全量补齐，把整个历史拉回渲染进程 |
| 3 | `packages/ui/src/v4/conversationProjectionStore.ts` | 快照窗口只有尾部 60 行且整体替换——长回合重订阅冲掉早前提问 |

根因 1 的病灶在共享的状态应用层，但**修复放在 UI 层**：不动协议语义、不动服务端，把改动面收到最小——兼容性零风险，上游合并成本也最低。

## 修复（3 处改动、纯客户端；协议层与服务端 `rowsRange` 分页不动）

窗口行集有界后，渲染层每帧的渲染/协调开销随之有界——对应 **CPU 高 / 卡顿**；内存占用不再随会话时长增长——对应 **内存持续上涨**。

1. **窗口收敛**：真人发新提问时，显示窗口收敛为「最近 10 个真实提问所在的回合」（按提问数不按行数；合成行不触发；没裁掉不换数组引用）。被收敛的行不是删除——滚动到顶 `loadOlder` 照常分页取回完整历史；
2. **快照守卫式尾拼**：同 `logEpoch` 且追加性未破坏（totalCount / 末行 rowId 单调、拼回行数 ≤ 快照声称上限）时把本地更早行拼回尾巴前，否则保持整体替换；
3. **导航自动全量补齐加开关（默认停用）**：刻度本就从窗口实时派生，全量补齐会把收敛后的窗口重新拉满；停用后刻度 = 已加载窗口内的提问书签（分享视图通道不受影响）。**只关这一条，滚动到顶的 `loadOlder` 预取保留不动**——实测教训：把预取一并停用会切断旧历史加载（见「验证」）。

## 怎么用

构建环境要求跟上游一致（`mise.toml` 固定 Node.js 24.14.0 + pnpm 10.33.2，详见[上游 README](https://github.com/zai-org/ZCode)）。

```bash
# 1. 准备补丁和上游源码
git clone https://github.com/BenWongCityuCS/zcode_performance_boost
git clone https://github.com/zai-org/ZCode && cd ZCode

# 2. 预检 + 打补丁
git apply --check ../zcode_performance_boost/0001-fix-ui-bound-renderer-conversation-window-on-real-us.patch
git am ../zcode_performance_boost/0001-fix-ui-bound-renderer-conversation-window-on-real-us.patch

# 3. 构建运行（构建步骤详见上游 README）
pnpm bootstrap && pnpm dev:desktop
```

- **调阈值**：常量 `CONVERSATION_WINDOW_MAX_QUERY_TURNS = 10`（`packages/ui/src/v4/conversationWindowBound.ts`），可自行调整或做成设置项；
- **回滚**：`git am --abort`（应用中途放弃）；已应用后撤销用 `git apply -R ../zcode_performance_boost/0001-*.patch`。

## 怎么确认生效

1. 连发到第 11 个新提问时，显示窗口收敛到最近 10 个提问所在的回合（更早的没丢，滚到顶 `loadOlder` 取回）；
2. 单轮长时间干活（多工具调用）期间，早前的提问不再被挤走；
3. 滚动到顶加载更早消息正常。

## 验证

- `pnpm typecheck` 通过；`oxlint` 触碰文件 0 警告 0 错误；`architecture:check --changed` 0 违规；
- 新增单测 8/8 通过（`packages/ui/test/conversationWindowBound.test.ts`，node:test）：收敛语义（恒等稳定 / 按提问数 / 合成行不触发）+ 尾拼守卫（同 epoch 拼接、rewind 放弃、上限放弃）；
- **同语义在 ZCode Desktop 3.14.4 上实机实测迭代两轮**：首轮发现「滚动到顶的旧历史预取被一并停用」会切断旧历史加载（回归）——预取必须保留，这也是修复 3 只关导航全量补齐、不动 `loadOlder` 的原因；恢复预取后复测通过。最终行为：第 11 个新提问收回旧回合、长回合不挤走提问、滚到顶加载更早正常。

## 维护

- 基于 zai-org/ZCode **v3.14.3** 树生成，**v3.14.4** 同语义实测；上游更新后补丁可能需要 rebase，先跑 `git apply --check` 预检；
- 语义取舍（为什么只在真人提问时收敛、为什么按提问数不按行数、为什么恒等稳定）写在 `conversationWindowBound.ts` 文件头注释与补丁 commit message 里。

## 文件

| 文件 | 说明 |
|---|---|
| `0001-fix-ui-bound-renderer-conversation-window-on-real-us.patch` | 补丁本体（`git format-patch` 格式，4 文件 +240/−7） |
| `ISSUE.md` | 投稿文案（现象 / 根因 / 修复 / 验证），可直接贴官方反馈仓库（官方仓已关闭外部 PR，patch 以附件/贴文形式给） |
| `NOTICE.md` | 上游 `NOTICE.md` 原文保留（Apache-2.0 §4 义务；文中路径指向上游仓库） |
| `LICENSE` | Apache-2.0（与上游一致；本补丁同样以 Apache-2.0 发布） |