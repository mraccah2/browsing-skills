---
name: browsing-reddit
description: "Use when the user wants to interact with Reddit — extract subreddit feeds, Reddit search results, post/thread data with comments, user profile activity, logged-in personal lists, or submit a comment under the logged-in account. Read actions work on public pages without login; logged-in lists and submit-comment require the user's existing browser session. Use a real browser or Chrome Bridge because Reddit is a JavaScript-rendered app and may challenge plain fetches."
---

# Reddit — Browsing Skill

Use this index to choose the Reddit action that matches the user request, then open only the linked reference file for the complete navigation, requirements, code, and return shape.

## Action Index

- **subreddit-feed** — Extract visible posts from a subreddit feed such as hot, new, top, rising, or controversial. Full spec: [references/subreddit-feed.md](references/subreddit-feed.md).
- **search** — Search Reddit globally or within a subreddit and extract visible result posts, communities, users, and comments when present. Full spec: [references/search.md](references/search.md).
- **post-thread** — Extract a Reddit post plus visible nested comments, including comment depth, author, score, created time, permalinks, and body text. Full spec: [references/post-thread.md](references/post-thread.md).
- **user-profile** — Extract visible profile metadata and recent visible user activity from a Reddit user page. Full spec: [references/user-profile.md](references/user-profile.md).
- **current-user-list** — Extract visible items from logged-in personal Reddit lists such as saved, upvoted, downvoted, hidden, and history. Full spec: [references/current-user-list.md](references/current-user-list.md).
- **submit-comment** *(write action — requires login + explicit user consent)* — Post a top-level comment on a Reddit post, or reply to a specific parent comment, under the logged-in account. Verifies the comment appears in the DOM before reporting success; detects rate-limit and removal toasts. Full spec: [references/submit-comment.md](references/submit-comment.md).
- **upvote-post** *(write — requires login + consent)* — Toggle the upvote on the loaded Reddit post. Returns already-voted state without acting unless `force:true`. Full spec: [references/upvote-post.md](references/upvote-post.md).
- **upvote-comment** *(write — requires login + consent)* — Toggle the upvote on a specific comment by fullname id. Full spec: [references/upvote-comment.md](references/upvote-comment.md).
- **delete-own-comment** *(destructive write — requires login + consent)* — Delete a comment owned by the logged-in account. Permanent. Full spec: [references/delete-own-comment.md](references/delete-own-comment.md).

## Notes

- **Read vs. write.** Six of these are read-only; four are write actions (`submit-comment`, `upvote-post`, `upvote-comment`, `delete-own-comment`). Read actions do not vote, save, hide, join communities, send messages, or change account settings. Write actions are clearly tagged in the action index above and require explicit user consent + DOM verification per the write-action policy.
- **Reply to a comment:** use `submit-comment` with `parentCommentId: "t1_<id>"`. There is no separate `reply-to-comment` action — `submit-comment` covers both top-level and reply cases.
- **Logged-in actions.** `current-user-list` and `submit-comment` require the user's existing browser session. They cannot log in. Prefer Chrome Bridge for these.
- **Virtualized feeds.** Reddit virtualizes long feeds. Each read action extracts currently loaded visible items; scroll or load more before running when the user needs deeper coverage.
- **Challenge pages.** Prefer Chrome Bridge when the page depends on the user's existing logged-in Chrome session or when Reddit presents a challenge page to new browser contexts.
- **Write-action policy.** `submit-comment` is non-idempotent. The agent must (a) get explicit user approval for the exact body text and target, (b) check the returned `posted` + `newCommentId` before claiming success, (c) follow up with `post-thread` to confirm the comment is visible to logged-out readers (shadow-bans render to author only), and (d) honor cooldowns: ≥10 min between comments for new/low-karma accounts, ≥60s for aged accounts.
