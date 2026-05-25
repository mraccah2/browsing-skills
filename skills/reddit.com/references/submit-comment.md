# Reddit — Submit Comment Reference

## Requirements

**Auth:** **Required.** The user must already be logged into Reddit in the active browser session. This action cannot log in.

**Browser:** Use a real browser or Chrome Bridge. Reddit's composer is a web component (`shreddit-composer` / Lexical-based contenteditable) that does not work via plain fetches and requires the user's session cookies + anti-abuse JS.

**User consent:** This is a **write action** that posts a comment under the logged-in account. Only run after the user has explicitly approved the comment text and the target post/comment. Never run on speculation, never run during exploratory browsing. The agent is responsible for confirming `body` matches what the user approved verbatim.

**Account-state safety:**

- This action will fail silently (not throw) if Reddit rate-limits, shadow-blocks, or anti-abuse-challenges the submission. Always inspect the returned `posted` boolean + `newCommentId`/`newCommentPermalink` before claiming success. A truthy `posted` requires the action to have observed the new comment in the DOM, not just a clicked button.
- Reddit's "you're doing that too much" interstitial counts as `posted: false` with `error: "rate_limited"`.
- New, low-karma, or recently-flagged accounts may have `submit` silently dropped even with HTTP 200 on the underlying API — confirm via DOM observation, never by HTTP response alone.

## How to run this action

Navigate to the post page (or comment permalink for a nested reply), wait until the composer is visible, then execute via `page.evaluate()` or Chrome Bridge `/run-action`. The action does its own scroll-into-view + click-to-open-composer where needed.

```js
const result = await page.evaluate(async (code) => {
  const tool = eval(code);
  return await tool.execute({
    body: "Approved comment text here.",
    parentCommentId: null,        // null = top-level on the post; "t1_xxxxxx" to reply to a comment
    mode: "data",
    waitForAppearMs: 8000
  });
}, scriptCode);

const data = JSON.parse(result.content[0].text);
```

Use `mode: "display"` for self-contained HTML output instead of JSON.

---

## Action: submit-comment

Use when the user has explicitly approved a comment body and a target (a Reddit post for a top-level comment, or a specific parent comment for a nested reply).

**Navigate to:** the post page where the comment should appear:

`https://www.reddit.com/r/<subreddit>/comments/<post-id>/<slug>/`

For nested replies, the same post page is sufficient — pass `parentCommentId` to target the reply chain.

**Code:**

```js
({
  name: "reddit-submit-comment",
  description: "Submit a comment (top-level on the current post, or as a reply to a specific parent comment) under the logged-in Reddit account. Verifies the comment landed in the DOM before returning posted:true.",
  inputSchema: {
    type: "object",
    required: ["body"],
    properties: {
      body: { type: "string", description: "The exact comment text to submit. Required. Action does not modify, truncate, or sanitize." },
      parentCommentId: { type: ["string", "null"], description: "Reddit fullname id (e.g. 't1_abc123') of the parent comment to reply to. Null for a top-level comment on the current post." },
      mode: { type: "string", enum: ["data", "display"], description: "Output mode. data returns JSON. display returns self-contained HTML." },
      waitForAppearMs: { type: "number", description: "Max ms to wait for the new comment to appear in the DOM after clicking submit. Defaults to 8000." },
      submitDelayMs: { type: "number", description: "Delay after typing before clicking submit. Defaults to 250ms." }
    }
  },
  execute: async function(params) {
    var mode = (params && params.mode) || "data";
    var body = String((params && params.body) || "");
    var parentCommentId = (params && params.parentCommentId) || null;
    var waitForAppearMs = Math.max(1000, Math.min(30000, parseInt((params && params.waitForAppearMs) || 8000, 10)));
    var submitDelayMs = Math.max(0, Math.min(5000, parseInt((params && params.submitDelayMs) || 250, 10)));

    function clean(s) { return (s || "").replace(/[ \t]+\n/g, "\n").replace(/\n{3,}/g, "\n\n").replace(/[ \t]{2,}/g, " ").trim(); }
    function flat(s) { return (s || "").replace(/\s+/g, " ").trim(); }
    function esc(s) { return String(s || "").replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;"); }
    function sleep(ms) { return new Promise(function(resolve) { setTimeout(resolve, ms); }); }
    function visible(el) {
      if (!el) return false;
      var rect = el.getBoundingClientRect();
      var style = window.getComputedStyle(el);
      return rect.width > 0 && rect.height > 0 && style.visibility !== "hidden" && style.display !== "none";
    }
    function attr(el, name) { return el && el.getAttribute ? el.getAttribute(name) || "" : ""; }
    function abs(url) {
      if (!url) return "";
      try { return new URL(url, window.location.href).href.split("?")[0]; } catch (e) { return url; }
    }

    function fail(err, extra) {
      var out = Object.assign({ posted: false, error: err, body: body, parentCommentId: parentCommentId, newCommentId: null, newCommentPermalink: null }, extra || {});
      return { content: [{ type: "text", text: JSON.stringify(out, null, 2) }] };
    }

    if (!body || !body.trim()) return fail("empty_body");

    // Quick logged-out sanity check.
    var loggedOutBtn = document.querySelector("a[href*='/login']");
    var loggedInIndicators = document.querySelector("shreddit-app[user-id], faceplate-tracker[noun='user_drawer'], #header-user-dropdown, [data-testid='UserDrawer-Trigger']");
    if (loggedOutBtn && !loggedInIndicators) return fail("not_logged_in");

    // Locate the composer scope. Top-level: the post-level composer. Reply: the composer inside the targeted comment.
    var scope = document;
    var targetComment = null;
    if (parentCommentId) {
      targetComment = document.querySelector("shreddit-comment[thingid='" + parentCommentId + "']") ||
                      document.querySelector("[id='" + parentCommentId + "']") ||
                      document.querySelector("[data-thing-id='" + parentCommentId + "']");
      if (!targetComment) return fail("parent_not_found");
      targetComment.scrollIntoView({ behavior: "instant", block: "center" });
      await sleep(150);
      // Open the reply composer on this comment.
      var replyBtn = targetComment.querySelector("button[name='reply'], button[aria-label^='Reply'], shreddit-async-loader button[slot='reply']");
      if (replyBtn && visible(replyBtn)) {
        replyBtn.click();
        // Wait for composer to mount inside this comment's subtree.
        for (var ri = 0; ri < 40; ri++) {
          var sub = targetComment.querySelector("shreddit-composer, [contenteditable='true']");
          if (sub && visible(sub)) break;
          await sleep(100);
        }
      }
      scope = targetComment;
    }

    var composer = scope.querySelector("shreddit-composer [contenteditable='true']") ||
                   scope.querySelector("[name='comment-composer-text'] [contenteditable='true']") ||
                   scope.querySelector("div[role='textbox'][contenteditable='true']") ||
                   scope.querySelector("[contenteditable='true'][data-lexical-editor]") ||
                   scope.querySelector("[contenteditable='true']");
    if (!composer) return fail("composer_not_found");

    composer.scrollIntoView({ behavior: "instant", block: "center" });
    await sleep(150);
    composer.focus();
    await sleep(50);

    // Insert text. Lexical/Draft editors respond best to a paste event with text/plain.
    var dt = new DataTransfer();
    dt.setData("text/plain", body);
    var pasteEvt;
    try {
      pasteEvt = new ClipboardEvent("paste", { clipboardData: dt, bubbles: true, cancelable: true });
    } catch (e) { pasteEvt = null; }
    var pasted = false;
    if (pasteEvt) pasted = composer.dispatchEvent(pasteEvt);

    // Fallback: execCommand insertText (deprecated but still wired in Chromium).
    if (!pasted || !flat(composer.innerText || composer.textContent)) {
      try { document.execCommand("insertText", false, body); } catch (e) {}
    }

    // Final fallback: direct textContent (last resort; rich editors may strip it).
    if (!flat(composer.innerText || composer.textContent)) {
      composer.textContent = body;
      composer.dispatchEvent(new InputEvent("input", { bubbles: true, inputType: "insertFromPaste", data: body }));
    }

    await sleep(submitDelayMs);

    // Snapshot existing comment ids so we can detect the new one.
    var beforeIds = {};
    var beforeNodes = (scope === document ? document : document).querySelectorAll("shreddit-comment");
    for (var bi = 0; bi < beforeNodes.length; bi++) {
      var tid = attr(beforeNodes[bi], "thingid") || beforeNodes[bi].id;
      if (tid) beforeIds[tid] = true;
    }

    // Locate the submit button (the composer's primary submit, NOT the global header post button).
    var composerHost = composer.closest("shreddit-composer") || composer.closest("[role='dialog']") || composer.parentElement;
    var submitBtn = null;
    if (composerHost) {
      submitBtn = composerHost.querySelector("button[type='submit']") ||
                  composerHost.querySelector("button[slot='submit-button']") ||
                  composerHost.querySelector("button[aria-label='Comment'], button[aria-label='Reply'], button[aria-label='Post']");
    }
    if (!submitBtn || !visible(submitBtn) || submitBtn.disabled) {
      // Sometimes the button enables after a microtask.
      await sleep(200);
      if (composerHost) {
        submitBtn = composerHost.querySelector("button[type='submit']:not([disabled])") ||
                    composerHost.querySelector("button[slot='submit-button']:not([disabled])") ||
                    submitBtn;
      }
    }
    if (!submitBtn || submitBtn.disabled) return fail("submit_button_unavailable", { composerHasText: !!flat(composer.innerText) });

    submitBtn.click();

    // Wait for a new shreddit-comment to appear with text that matches.
    var bodyMatch = flat(body).slice(0, 80);
    var deadline = Date.now() + waitForAppearMs;
    var newComment = null;
    while (Date.now() < deadline) {
      var afterNodes = document.querySelectorAll("shreddit-comment");
      for (var ai = 0; ai < afterNodes.length; ai++) {
        var n = afterNodes[ai];
        var nid = attr(n, "thingid") || n.id;
        if (!nid || beforeIds[nid]) continue;
        var nbody = flat(n.innerText || n.textContent);
        if (nbody && nbody.indexOf(bodyMatch.slice(0, 40)) !== -1) {
          newComment = n;
          break;
        }
      }
      if (newComment) break;

      // Rate-limit / error toast detection.
      var toast = document.querySelector("[role='alert'], [data-testid='toast'], shreddit-toast");
      var toastText = toast ? flat(toast.innerText) : "";
      if (toastText && /too (much|fast|often)|rate.?limit|try again later|wait/i.test(toastText)) {
        return fail("rate_limited", { toast: toastText.slice(0, 200) });
      }
      if (toastText && /removed|blocked|forbidden|error/i.test(toastText)) {
        return fail("submission_rejected", { toast: toastText.slice(0, 200) });
      }
      await sleep(200);
    }

    if (!newComment) return fail("not_observed_in_dom", { waitedMs: waitForAppearMs });

    var newId = attr(newComment, "thingid") || newComment.id || "";
    var permalink = abs(attr(newComment, "permalink") || (newComment.querySelector("a[href*='/comments/']") || {}).getAttribute && newComment.querySelector("a[href*='/comments/']").getAttribute("href"));

    var data = {
      posted: true,
      error: null,
      newCommentId: newId,
      newCommentPermalink: permalink || "",
      parentCommentId: parentCommentId,
      body: body,
      postUrl: abs(window.location.href),
      observedAt: new Date().toISOString()
    };

    if (mode === "display") {
      var h = "<div style='font-family:-apple-system,BlinkMacSystemFont,Segoe UI,sans-serif;max-width:700px;margin:0 auto;padding:20px;'>";
      h += "<div style='color:#0a7;font-weight:700;margin-bottom:8px;'>✓ Comment posted</div>";
      h += "<div style='color:#666;font-size:13px;margin-bottom:12px;'>" + esc(data.newCommentId) + (parentCommentId ? " (reply to " + esc(parentCommentId) + ")" : " (top-level)") + "</div>";
      h += "<div style='border:1px solid #ddd;padding:12px;border-radius:6px;white-space:pre-wrap;'>" + esc(body) + "</div>";
      if (permalink) h += "<div style='margin-top:10px;'><a href='" + esc(permalink) + "'>" + esc(permalink) + "</a></div>";
      h += "</div>";
      return { content: [{ type: "text", text: h }] };
    }

    return { content: [{ type: "text", text: JSON.stringify(data, null, 2) }] };
  }
})
```

**Returns (success):** `{ posted: true, error: null, newCommentId, newCommentPermalink, parentCommentId, body, postUrl, observedAt }`

**Returns (failure):** `{ posted: false, error: "<reason>", body, parentCommentId, newCommentId: null, newCommentPermalink: null, ...optional context }`

Possible `error` values:
- `empty_body` — caller passed no body
- `not_logged_in` — no user session detected on the page
- `parent_not_found` — `parentCommentId` was provided but the comment isn't in the DOM (didn't scroll into view / hidden / wrong post)
- `composer_not_found` — page didn't render a usable contenteditable composer
- `submit_button_unavailable` — composer rendered but its submit button never enabled (often: composer didn't recognize the typed text)
- `rate_limited` — Reddit displayed a rate-limit toast
- `submission_rejected` — Reddit displayed a removal/block toast
- `not_observed_in_dom` — submit clicked, no error toast, but no new comment matching the body appeared within `waitForAppearMs` (treat as ambiguous — re-check via `post-thread` next)

## Notes for agents

- **Verify, don't trust.** Even on `posted: true`, follow up with [post-thread](post-thread.md) for the same post to confirm the comment is visible to other users. Shadow-banned comments render to the author but not to logged-out readers.
- **Idempotency.** The action is not idempotent. If you suspect a previous attempt actually landed (e.g. `not_observed_in_dom` after a successful click), reload the post and check before retrying — Reddit will accept and double-post if you re-submit.
- **Cooldowns.** New accounts: budget ≥10 minutes between comments. Aged accounts: ≥60 seconds. Bursting will trigger silent shadow-removal that this action cannot detect.
