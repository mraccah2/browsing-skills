# Reddit — Upvote Comment Reference

## Requirements

**Auth:** **Required.**
**Browser:** Required.
**User consent:** **Write action.** See post-upvote for same anti-sweep caveats.

## How to run this action

Navigate to the post page, then `page.evaluate()` passing the comment fullname id.

---

## Action: upvote-comment

Toggle upvote on a specific comment by id.

**Navigate to:** `https://www.reddit.com/r/<subreddit>/comments/<post-id>/<slug>/` (the comment must be visible in the rendered tree)

**Code:**

```js
({
  name: "reddit-upvote-comment",
  description: "Toggle upvote on a Reddit comment identified by its fullname (t1_...).",
  inputSchema: {
    type: "object",
    required: ["commentId"],
    properties: {
      commentId: { type: "string", description: "Reddit fullname, e.g. 't1_abc123'." },
      force: { type: "boolean" },
      waitForUpdateMs: { type: "number" }
    }
  },
  execute: async function(params) {
    var commentId = params.commentId;
    var force = !!params.force;
    var waitForUpdateMs = Math.max(1000, Math.min(15000, parseInt(params.waitForUpdateMs || 5000, 10)));

    function clean(s) { return (s || "").replace(/\s+/g, " ").trim(); }
    function sleep(ms) { return new Promise(function(r) { setTimeout(r, ms); }); }
    function fail(err) { return { content: [{ type: "text", text: JSON.stringify({ voted: false, error: err }, null, 2) }] }; }
    function numFrom(v) {
      v = clean(v);
      if (!v) return null;
      if (/k$/i.test(v)) return Math.round(parseFloat(v) * 1000);
      var n = parseInt(v.replace(/[^0-9-]/g, ""), 10);
      return isNaN(n) ? null : n;
    }

    if (document.querySelector("a[href*='/login']") && !document.querySelector("shreddit-app[user-id]")) return fail("not_logged_in");

    var commentEl = document.querySelector("shreddit-comment[thingid='" + commentId + "']") ||
                    document.querySelector("[id='" + commentId + "']");
    if (!commentEl) return fail("comment_not_found");

    commentEl.scrollIntoView({ behavior: "instant", block: "center" });
    await sleep(150);

    var upBtn = commentEl.querySelector("button[upvote], button[aria-label*='upvote' i]");
    if (!upBtn) return fail("upvote_button_not_found");

    var scoreEl = commentEl.querySelector("faceplate-number, [class*='score' i]");
    var beforeScore = scoreEl ? numFrom(scoreEl.innerText) : null;
    var beforePressed = upBtn.getAttribute("aria-pressed") === "true";

    if (beforePressed && !force) {
      return { content: [{ type: "text", text: JSON.stringify({ voted: false, alreadyVoted: true, scoreBefore: beforeScore, scoreAfter: beforeScore, error: null }, null, 2) }] };
    }

    upBtn.click();

    var deadline = Date.now() + waitForUpdateMs;
    var afterPressed = beforePressed;
    var afterScore = beforeScore;
    while (Date.now() < deadline) {
      await sleep(150);
      var fresh = commentEl.querySelector("button[upvote], button[aria-label*='upvote' i]") || upBtn;
      afterPressed = fresh.getAttribute("aria-pressed") === "true";
      var freshScoreEl = commentEl.querySelector("faceplate-number, [class*='score' i]");
      if (freshScoreEl) afterScore = numFrom(freshScoreEl.innerText);
      if (afterPressed !== beforePressed) break;
    }

    var voted = afterPressed && !beforePressed;
    if (!voted && !(force && beforePressed && !afterPressed)) return fail("not_observed_in_dom");

    return { content: [{ type: "text", text: JSON.stringify({
      voted: voted,
      alreadyVoted: false,
      commentId: commentId,
      scoreBefore: beforeScore,
      scoreAfter: afterScore,
      error: null,
      observedAt: new Date().toISOString(),
    }, null, 2) }] };
  }
})
```

**Returns:** `{ voted, alreadyVoted, commentId, scoreBefore, scoreAfter, error, observedAt }`
