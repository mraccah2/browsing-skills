# Reddit — Delete Own Comment Reference

## Requirements

**Auth:** **Required.** The comment must be owned by the logged-in account.
**Browser:** Required.
**User consent:** **Destructive write.** Permanent. Reddit shows `[deleted]` in place, then garbage-collects.

## How to run this action

Navigate to the comment's permalink, then `page.evaluate()`.

---

## Action: delete-own-comment

Delete a comment owned by the logged-in user.

**Navigate to:** the comment permalink, e.g. `https://www.reddit.com/r/<sub>/comments/<post>/comment/<id>/`

**Code:**

```js
({
  name: "reddit-delete-own-comment",
  description: "Delete a Reddit comment owned by the logged-in account. Verifies the comment is gone or marked [deleted].",
  inputSchema: {
    type: "object",
    required: ["commentId"],
    properties: {
      commentId: { type: "string", description: "Reddit fullname, e.g. 't1_abc123'." },
      waitForUpdateMs: { type: "number" }
    }
  },
  execute: async function(params) {
    var commentId = params.commentId;
    var waitForUpdateMs = Math.max(1000, Math.min(15000, parseInt(params.waitForUpdateMs || 5000, 10)));

    function clean(s) { return (s || "").replace(/\s+/g, " ").trim(); }
    function sleep(ms) { return new Promise(function(r) { setTimeout(r, ms); }); }
    function fail(err) { return { content: [{ type: "text", text: JSON.stringify({ deleted: false, error: err }, null, 2) }] }; }

    if (document.querySelector("a[href*='/login']") && !document.querySelector("shreddit-app[user-id]")) return fail("not_logged_in");

    var commentEl = document.querySelector("shreddit-comment[thingid='" + commentId + "']");
    if (!commentEl) return fail("comment_not_found");

    // Open the comment's overflow menu.
    var menuBtn = commentEl.querySelector("button[aria-label*='more' i], faceplate-dropdown-menu button");
    if (!menuBtn) return fail("menu_button_not_found");
    menuBtn.click();
    await sleep(300);

    // Click Delete in the popover.
    var deleteItem = null;
    var menuItems = document.querySelectorAll("[role='menuitem'], button");
    for (var i = 0; i < menuItems.length; i++) {
      if (/^delete\b/i.test(clean(menuItems[i].innerText))) { deleteItem = menuItems[i]; break; }
    }
    if (!deleteItem) return fail("delete_menuitem_not_found");
    deleteItem.click();
    await sleep(300);

    // Confirm in the modal.
    var modal = document.querySelector("[role='dialog']");
    if (!modal) return fail("confirm_modal_not_shown");
    var confirmBtn = null;
    var modalBtns = modal.querySelectorAll("button");
    for (var ci = 0; ci < modalBtns.length; ci++) {
      if (/^(delete|confirm|yes)/i.test(clean(modalBtns[ci].innerText))) { confirmBtn = modalBtns[ci]; break; }
    }
    if (!confirmBtn) return fail("confirm_button_not_found");
    confirmBtn.click();

    // Verify: comment body should change to [deleted] or comment disappears.
    var deadline = Date.now() + waitForUpdateMs;
    while (Date.now() < deadline) {
      await sleep(200);
      var fresh = document.querySelector("shreddit-comment[thingid='" + commentId + "']");
      if (!fresh) return { content: [{ type: "text", text: JSON.stringify({ deleted: true, commentId: commentId, mode: "removed", error: null }, null, 2) }] };
      var bodyText = clean(fresh.innerText);
      if (/\[deleted\]|\[removed\]/i.test(bodyText)) return { content: [{ type: "text", text: JSON.stringify({ deleted: true, commentId: commentId, mode: "tombstone", error: null }, null, 2) }] };
    }
    return fail("not_observed_in_dom");
  }
})
```

**Returns:** `{ deleted, commentId, mode: "removed" | "tombstone", error }`

`error` codes: `not_logged_in`, `comment_not_found`, `menu_button_not_found`, `delete_menuitem_not_found`, `confirm_modal_not_shown`, `confirm_button_not_found`, `not_observed_in_dom`.
