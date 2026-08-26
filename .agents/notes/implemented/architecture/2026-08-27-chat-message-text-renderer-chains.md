# Agent Note: Chat message text renderer chains

Status: implemented

English | [中文](2026-08-27-chat-message-text-renderer-chains.zh.md)

## Problem

An extension that changes only chat prose could replace the whole `conversation.chat.node` renderer or mutate the rendered DOM. Whole-node replacement makes the extension reproduce the Host's Markdown, images, reasoning disclosures, Tool placement, streaming state, message actions, timing, branching, and future chrome. DOM mutation runs after React composition, has no typed owner data or slot lifecycle, and can lose its changes when streaming updates replace the body.

The Host needs an extension point whose authority ends at the text body. It must preserve the official presentation when no extension accepts a message and must not turn the remaining message row into extension-owned compatibility work.

## Decision

The built-in `user` and `assistant-step` keyed Chat renderers declare two session-scoped chain children:

- `conversation.chat.userText` renders the joined raw text of one durable user message. `UserMessageTextOwnerProps` carries the engine-owned `nodeKey` and `text`.
- `conversation.chat.assistantText` renders one Assistant text block. `AssistantMessageTextOwnerProps` carries `nodeKey`, the source-order `blockIndex`, `text`, and whether that Assistant step is streaming.

Each owner calls `renderSlotChain` with the Host rendering as its fallback. An all-declined user chain therefore retains literal text and reference projection; an all-declined Assistant chain retains the official Markdown renderer, code actions, and file mentions. The chain selector is the only routing decision, following the [slot-system chain contract](2026-07-22-slot-type-chain-implementation.md).

Node keys are stable within one Session, not globally. The selector therefore receives the current `sessionId` through the framework's separate read-only scope argument and combines it with the owner `nodeKey` and, for Assistant blocks, `blockIndex`. The owner types remain message data only; they do not duplicate framework scope identity.

The child slots do not cover user images, unknown content blocks, Assistant reasoning, images, Tool calls, interruption status, message actions, clocks, branches, or Turn-tail extensions. These remain in their existing Host components around the selected body. Pending and durable steering also remain literal Host text because steering presentation participates in the pending-to-durable handoff rather than in third-party transcript decoration.

The keyed Chat renderer owns each child declaration. Unloading that renderer removes the declaration and its contributions through the normal slot lifecycle. Stable Node keys let an elected renderer preserve identity across ordinary updates; `streaming` tells an Assistant renderer when the same logical text is still changing.

## Verification

Conversation component tests cover user takeover, multiple Assistant text blocks, source block indexes, stable Node keys, streaming state, all-declined literal and Markdown fallbacks, and Host-owned reasoning and actions. Slot renderer tests cover immutable root, strict-session, and session-maybe selector scope identities and Session switches; type tests keep the scope-specific `sessionId` contracts distinct. Apply tests cover both child declarations and their removal with the owning plugin fiber. The generated Client slot catalog publishes the two names and owner fields for extension authors.

## Alternatives considered

**Replace the whole keyed Chat node.** This already supports genuinely different business rows, but using it for prose decoration transfers every current and future row responsibility to the extension and couples compatibility work to Host UI changes.

**Mutate message DOM after React renders it.** This avoids a Host API change but has no typed raw-text input, no stable composition lifecycle, and races streaming or rerender replacement.

**Expose one chain for every message kind.** User and Assistant text have different fallback semantics and Assistant needs block-level identity and streaming state. A shared owner union would make every selector narrow unrelated cases and would imply support for steering and context bodies that this decision deliberately leaves Host-owned.

**Expose raw HTML as the fallback contract.** Host HTML is an implementation detail and cannot preserve React ownership, actions, or update semantics when moved between components. The chain passes React fallback content and raw text separately.

## Consequences

Text presentation plugins can integrate through a typed, lifecycle-owned registration without copying the message row or patching the DOM. The Host continues to own layout, non-text content, message chrome, accessibility, and the no-extension result. Assistant extensions receive one dispatch per text block rather than one concatenated message, preserving source order around reasoning and image blocks.

The cost is two additional public slot names and owner types. A selected renderer is responsible for presenting the raw text it accepts, including streaming updates; the Host cannot guarantee Markdown, references, or code controls inside an elected replacement. Extensions needing a different complete row still use `conversation.chat.node` rather than expanding these text chains.
