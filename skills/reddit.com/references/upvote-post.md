# Reddit — Upvote Post Reference

## Requirements

**Auth:** **Required.**

**Browser:** Required.

**User consent:** **Write action.** Reddit's anti-abuse system treats sweep-voting as bot behavior — only run on user-approved targets.

## How to run this action

Navigate to the post page, then `page.evaluate()`.

---

## Action: upvote-post

Toggle the upvote on a Reddit post. No-op if already upvoted (unless `force: true`, which un-votes).

**Navigate to:** `https://www.reddit.com/r/<subreddit>/comments/<post-id>/<slug>/`

**Code:**

```js
({
  name: "reddit-upvote-post",
  description: "Toggle upvote on the loaded Reddit post under the logged-in account. Returns already-voted state without acting (unless force).",
  inputSchema: {
    type: "object",
    properties: {
      force: { type: "boolean", description: "If true, toggle even when already voted (un-vote). Default false." },
      waitForUpdateMs: { type: "number" }
    }
  },
  execute: async function(params) {
    var force = !!(params && params.force);
    var waitForUpdateMs = Math.max(1000, Math.min(15000, parseInt((params && params.waitForUpdateMs) || 5000, 10)));

    function clean(s) { return (s || "").replace(/\s+/g, " ").trim(); }
    function sleep(ms) { return new Promise(function(r) { setTimeout(r, ms); }); }
    function fail(err, extra) { return { content: [{ type: "text", text: JSON.stringify(Object.assign({ voted: false, error: err }, extra || {}), null, 2) }] }; }
    function numFrom(v) {
      v = clean(v);
      if (!v) return null;
      if (/k$/i.test(v)) return Math.round(parseFloat(v) * 1000);
      var n = parseInt(v.replace(/[^0-9-]/g, ""), 10);
      return isNaN(n) ? null : n;
    }

    var loggedInSignal = document.querySelector("shreddit-app[user-id], #header-user-dropdown");
    var loggedOutSignal = document.querySelector("a[href*='/login']");
    if (loggedOutSignal && !loggedInSignal) return fail("not_logged_in");

    // Locate the post-level upvote button (NOT comment-level). The post is the
    // top-level shreddit-post element.
    var postEl = document.querySelector("shreddit-post");
    if (!postEl) return fail("post_element_not_found");

    var upBtn = postEl.querySelector("button[upvote], button[aria-label*='upvote' i], shreddit-async-loader button[name='upvote']");
    if (!upBtn) return fail("upvote_button_not_found");

    var scoreEl = postEl.querySelector("[data-post-click-location='vote-count'], faceplate-number, [class*='score' i]");
    var beforeScore = scoreEl ? numFrom(scoreEl.innerText) : null;
    var beforePressed = upBtn.getAttribute("aria-pressed") === "true" || upBtn.classList.contains("upvoted");

    if (beforePressed && !force) {
      return { content: [{ type: "text", text: JSON.stringify({ voted: false, alreadyVoted: true, scoreBefore: beforeScore, scoreAfter: beforeScore, error: null }, null, 2) }] };
    }

    upBtn.click();

    var deadline = Date.now() + waitForUpdateMs;
    var afterScore = beforeScore;
    var afterPressed = beforePressed;
    while (Date.now() < deadline) {
      await sleep(150);
      var fresh = postEl.querySelector("button[upvote], button[aria-label*='upvote' i]") || upBtn;
      afterPressed = fresh.getAttribute("aria-pressed") === "true" || fresh.classList.contains("upvoted");
      var freshScoreEl = postEl.querySelector("[data-post-click-location='vote-count'], faceplate-number");
      if (freshScoreEl) afterScore = numFrom(freshScoreEl.innerText);
      if (afterPressed !== beforePressed) break;
    }

    var voted = afterPressed && !beforePressed;
    if (!voted && !(force && beforePressed && !afterPressed)) return fail("not_observed_in_dom");

    return { content: [{ type: "text", text: JSON.stringify({
      voted: voted,
      alreadyVoted: false,
      scoreBefore: beforeScore,
      scoreAfter: afterScore,
      error: null,
      postUrl: window.location.href.split("?")[0],
      observedAt: new Date().toISOString(),
    }, null, 2) }] };
  }
})
```

**Returns:** `{ voted, alreadyVoted, scoreBefore, scoreAfter, error, postUrl, observedAt }`

`error` codes: `not_logged_in`, `post_element_not_found`, `upvote_button_not_found`, `not_observed_in_dom`.
