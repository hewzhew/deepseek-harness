# Agent Note: 聊天消息正文 renderer chain

Status: implemented

[English](2026-08-27-chat-message-text-renderer-chains.md) | 中文

## 问题

只想改变聊天正文的扩展原本只能替换整个 `conversation.chat.node` renderer，或者修改渲染完成后的 DOM。替换整个 Node 会迫使扩展重做 Host 的 Markdown、图片、推理展开项、Tool 位置、streaming 状态、消息操作、计时、分支以及未来新增的 chrome。DOM 修改发生在 React 组合之后，既没有类型化 owner 数据与 slot 生命周期，也可能在 streaming 更新替换正文时丢失。

Host 需要一处权威止于正文的扩展点。没有扩展接收消息时，它必须保留官方表现，也不能把消息行的其余部分变成扩展需要长期兼容的工作。

## 决策

内建 `user` 与 `assistant-step` keyed Chat renderer 声明两个会话作用域的 chain 子项：

- `conversation.chat.userText` 渲染一条持久 user 消息拼接后的原始文本。`UserMessageTextOwnerProps` 携带引擎拥有的 `nodeKey` 与 `text`。
- `conversation.chat.assistantText` 渲染一个 Assistant 文本 block。`AssistantMessageTextOwnerProps` 携带 `nodeKey`、源码顺序中的 `blockIndex`、`text`，以及该 Assistant step 是否仍在 streaming。

每个 owner 调用 `renderSlotChain` 时都传入 Host renderer 作为 fallback。因此所有 user selector 都拒绝时保留字面文本与引用投影；所有 Assistant selector 都拒绝时保留官方 Markdown renderer、代码操作与文件提及。selector 抛错时会被报告并按拒绝处理。已选中的 chain 组件抛错时，它自己的逐次分派错误边界会报告失败并渲染同一个 owner fallback，但不移除注册，因此其他消息仍能选中该贡献。chain selector 仍是唯一的路由判定，遵循 [slot 体系的 chain 约定](2026-07-22-slot-type-chain-implementation.zh.md)。

Node key 只在单个 Session 内稳定，并非全局稳定。因此 selector 经框架独立的只读 scope 参数获得当前 `sessionId`，再将它与 owner 的 `nodeKey` 以及 Assistant block 的 `blockIndex` 组合使用。owner 类型仍然只承载消息数据，不重复框架 scope identity。

这些子 slot 不覆盖 user 图片、未知内容 block、Assistant 推理、图片、Tool call、中断状态、消息操作、时钟、分支或 Turn-tail 扩展。它们继续留在所选正文周围的既有 Host 组件中。待处理与持久 steering 也继续使用 Host 字面文本，因为 steering 表现参与从待处理项到持久消息的交接，而不属于第三方 transcript 装饰。

keyed Chat renderer 拥有每个子项的声明。卸载该 renderer 会通过普通 slot 生命周期移除声明及其贡献。稳定 Node key 使当选 renderer 能在普通更新之间保持 identity；`streaming` 则告诉 Assistant renderer 同一逻辑文本是否仍在变化。

## 验证

Conversation 组件测试覆盖 user 接管、多个 Assistant 文本 block、源码 block index、稳定 Node key、streaming 状态、所有 selector 拒绝时的字面文本与 Markdown fallback，以及 Host 持有的推理和操作。Slot renderer 测试覆盖不可变的 root、严格会话与 session-maybe selector scope identity、Session 切换、selector 异常，以及逐次分派组件失败返回 owner fallback 且不禁用健康分派；类型测试保持各 scope 的 `sessionId` 约定彼此分明。apply 测试覆盖两个子项的声明，以及随所属插件 fiber 一起移除。生成的 Client slot 目录向扩展作者公开这两个名称与 owner 字段。

## 考虑过的替代方案

**替换整个 keyed Chat Node。** 这条路径已经适合真正不同的业务行，但用它装饰正文会把当前和未来的整行职责都转交给扩展，并把兼容工作绑定到 Host UI 的每次变化。

**在 React 渲染后修改消息 DOM。** 这可以避免增加 Host API，却没有类型化原始文本输入与稳定的组合生命周期，还会与 streaming 或重渲染替换竞争。

**为所有消息 kind 提供一个共同 chain。** User 与 Assistant 正文的 fallback 语义不同，Assistant 还需要 block 级 identity 与 streaming 状态。共同的 owner union 会迫使每个 selector 收窄无关分支，并暗示本决定有意留给 Host 的 steering 与 context 正文也受到支持。

**把原始 HTML 作为 fallback 约定。** Host HTML 是实现细节；在组件间移动它无法保留 React ownership、操作或更新语义。chain 分开传递 React fallback 内容与原始文本。

## 后果

正文表现插件可以通过类型化、受生命周期管理的注册完成集成，不必复制消息行或修补 DOM。Host 继续拥有布局、非文本内容、消息 chrome、无障碍语义以及没有扩展时的结果。renderer 缺陷只会让选中它的那一次分派降级为 Host 表现，不会留下空消息正文，也不会在全局禁用该贡献。Assistant 扩展按文本 block 分别收到分派，而不是收到拼接后的整条消息，因此能保留推理与图片 block 周围的源码顺序。

代价是增加两个公开 slot 名称与 owner 类型。当选 renderer 要负责呈现自己接收的原始文本，包括 streaming 更新；Host 无法保证替代内容内部仍有 Markdown、引用或代码控件。需要完整不同消息行的扩展仍应使用 `conversation.chat.node`，而不是扩张这两个正文 chain。
