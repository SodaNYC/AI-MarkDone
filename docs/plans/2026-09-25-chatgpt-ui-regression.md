# Codex Task — Repair ChatGPT Web compatibility after 2026-09-25 UI rollout

## Session configuration

- Model: **GPT-6 Luna**
- Reasoning: **High**
- Work branch: `codex/fix-chatgpt-2026-09-25-ui-regression`
- Base: `main`
- Do not bump version or prepare a store release in this task.

## Problem statement

AI-MarkDone 5.4.1 worked on ChatGPT Web before the UI rollout observed on 2026-09-24/25 and now appears effectively non-functional on the newly rolled out ChatGPT UI.

The fork is currently identical to upstream at commit `d6cc562931607f378c48023420f814de1f7c9d60`, so this is not a fork-sync problem.

External reports on 2026-09-25 also describe ChatGPT's navigation ladder and userscripts that depended on the prior DOM as breaking after the same UI rollout. Treat those reports as corroboration only; confirm the actual host DOM before editing.

## Current brittle host assumptions to inspect first

Do not assume these are all broken; they are the highest-probability boundaries to verify against the live page.

### Message discovery

`src/drivers/content/adapters/sites/chatgpt.ts`

- `getMessageSelector()` currently requires:
  - `[data-message-author-role="assistant"][data-message-id]`
- `getMessageContentSelector()` currently expects:
  - `.markdown.prose`
- user prompt extraction expects:
  - `article[data-turn]`
  - `[data-message-author-role="user"]`
  - `.whitespace-pre-wrap`

### Turn / slot discovery

`src/drivers/content/chatgpt/domConversationDiscovery.ts`

- role discovery depends on `[data-message-author-role]`
- turn wrappers depend on:
  - `[data-testid^="conversation-turn-"][data-turn]`
  - `article[data-turn]`
  - `section[data-turn]`
- persistent virtualized slots depend on:
  - `[data-turn-id-container]`

### Toolbar placement / readiness

`src/drivers/content/adapters/sites/chatgpt.ts`

- official action readiness depends on:
  - `button[data-testid="copy-turn-action-button"]`
- row placement currently prefers:
  - `div.z-0.flex`

A failure in message identity, turn topology, action-row readiness, or all three can make the extension look completely dead.

## Required workflow

Follow `AGENTS.md`, `.codex/rules/critical-rules.md`, and `.codex/guides/bug-fix.md`.

1. **Reproduce first.**
   - Use the current logged-in ChatGPT web UI if the Codex browser session can access it.
   - Reproduce in a normal conversation and, if available, a Project conversation.
   - Record which AI-MarkDone surfaces fail: message toolbar, directory rail, stepper, Reader, export/full-history refresh.
   - Check the browser console for AI-MarkDone warnings/errors.

2. **Capture live host evidence before implementation.**
   - Inspect one mounted user turn and assistant turn from the 2026-09-25 UI.
   - Capture only structural information needed for selectors/topology:
     - tag names
     - relevant `data-*` attributes
     - `aria-*` attributes
     - stable ancestor/descendant relations
     - action-row structure
   - Do **not** commit personal conversation text, account data, cookies, tokens, or full private DOM dumps.
   - Determine whether ChatGPT still preserves persistent virtualized slots outside the viewport.

3. **Write a failing regression fixture/test before changing production code.**
   Prefer adapting/adding coverage under:
   - `tests/unit/drivers/content/chatgpt/ChatGPTConversationDiscoveryAdapter.test.ts`
   - `tests/unit/drivers/content/chatgpt/domConversationDiscovery.host-slots.test.ts`
   - `tests/unit/drivers/content/chatgpt/ChatGPTConversationHostMonitor.test.ts`
   - `tests/unit/drivers/content/chatgpt/ChatGPTConversationSurface.test.ts`
   - `tests/unit/drivers/content/chatgpt/ChatGPTPageIndex.test.ts`
   - integration discovery lifecycle tests when appropriate.

4. **Confirm the root cause.**
   Classify it as one or more of:
   - message selector drift
   - message-id identity drift
   - turn wrapper / role drift
   - persistent slot topology drift
   - action-row readiness / toolbar-anchor drift
   - observer-root / scroll-root drift
   - hydration / virtualization lifecycle change
   - another verified host contract change

5. **Implement the smallest resilient compatibility fix.**
   Requirements:
   - Prefer semantic attributes/relations over generated Tailwind classes.
   - Avoid replacing one brittle selector with another one-off CSS class.
   - Preserve current 5.4.1 behavior as fallback where possible.
   - Keep DOM discovery centralized in the existing ChatGPT adapter/discovery layer.
   - Do not bypass repository/content/surface contracts.
   - Preserve assistant-only virtualization behavior.
   - Preserve Deep Research handling.
   - Preserve Chrome MV3 and Firefox MV2 parity.

6. **Make the compatibility boundary more tolerant where justified by evidence.**
   Candidate directions, only if live DOM proves they are needed:
   - allow assistant role nodes without `data-message-id` when a stable turn/message identity can be resolved from an ancestor
   - support the new turn wrapper while retaining old wrappers
   - resolve content roots from semantic descendants rather than requiring the exact old `.markdown.prose` combination
   - resolve action-row anchoring through semantic testids/relations instead of `div.z-0.flex`
   - keep persistent slot discovery if ChatGPT still exposes a durable outer slot under a changed attribute

7. **Verification.**
   Run at minimum:
   - `npm run test:chatgpt-discovery`
   - `npm run test:acceptance`
   - `npm run test:core`
   - `npm run build`

   Also run the targeted failing regression test directly while iterating.

## Acceptance criteria

- AI-MarkDone discovers at least one current ChatGPT user/assistant round on the 2026-09-25 UI.
- Per-message AI-MarkDone toolbar mounts on completed assistant responses.
- Streaming responses do not receive a false completed-state capture.
- Directory rail and lower-right previous/next navigation receive valid turns.
- Reader opens with current message content rather than an empty state.
- Existing full-history refresh/discovery remains functional for long virtualized conversations.
- Project conversations work if their DOM contract differs from ordinary chats.
- Existing 5.4.1 fixture coverage remains green.
- Deep Research behavior remains green.
- Chrome and Firefox builds succeed.

## Non-goals

- No redesign of AI-MarkDone UI.
- No new features.
- No broad refactor of the discovery architecture unless the new ChatGPT DOM makes the existing contract impossible to preserve.
- No release/version bump.
- No changes based solely on screenshots or speculation.

## Deliverable

Push the implementation and tests to this branch and update the draft PR with:

1. verified root cause;
2. live DOM contract changes observed (sanitized);
3. files changed and why;
4. tests/build commands and results;
5. remaining edge cases;
6. whether the fix should be proposed upstream to `zhaoliangbin42/AI-MarkDone`.
