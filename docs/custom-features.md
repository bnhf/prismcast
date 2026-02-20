# Custom Features (bnhf fork)

These features exist only in the bnhf fork and must be re-applied when syncing with upstream.
Each entry describes what the feature does, which files it touches, and exactly how to re-apply it.

---

## 1. Predefined Channel Auto-fill on Key Entry

### What it does
When a user opens the "Add Channel" form and types a key that matches a predefined channel (e.g. `abc`, `nbc`), the form automatically populates Name, URL, Station ID, Channel Selector, and Profile with the predefined defaults. Fields the user already filled are not overwritten. The lookup fires on blur of the key field.

### Why upstream doesn't have it
Upstream expects the user to fill in all fields manually. This feature improves UX when re-adding a predefined channel key.

### Files changed
- [src/routes/config.ts](../src/routes/config.ts) — new GET endpoint
- [src/routes/root.ts](../src/routes/root.ts) — client-side blur listener

### Validation
- Open the web UI → Channels tab → Add Channel.
- Type `abc` in the Key field and tab/click away.
- Name, URL, Station ID, Channel Selector, and Profile should auto-fill with ABC's defaults.
- Fields you typed in first must NOT be overwritten.
- A key that is not predefined (e.g. `mymadeupkey`) should do nothing.

### Implementation

#### A. New API endpoint — `src/routes/config.ts`

Insert before the `// GET /config/channels/export` block:

```typescript
// GET /config/channels/predefined-defaults - Returns predefined channel defaults for a given key. Used by the add channel form to pre-populate fields when the
// user enters a key that matches a predefined channel.
app.get("/config/channels/predefined-defaults", (req: Request, res: Response): void => {

  const key = typeof req.query.key === "string" ? req.query.key.trim().toLowerCase() : "";

  if(!key || !isPredefinedChannel(key)) {

    res.json({ found: false });

    return;
  }

  // Honour the user's current provider selection (e.g. "abc" → "abc-hulu"), then resolve inherited properties.
  const resolvedKey = resolveProviderKey(key);
  const channel = getResolvedChannel(resolvedKey);

  if(!channel) {

    res.json({ found: false });

    return;
  }

  res.json({

    channelSelector: channel.channelSelector ?? "",
    found: true,
    name: channel.name ?? "",
    profile: channel.profile ?? "",
    stationId: channel.stationId ?? "",
    url: channel.url
  });
});
```

`isPredefinedChannel`, `resolveProviderKey`, and `getResolvedChannel` are all already imported in config.ts — no import changes needed.

#### B. Client-side blur listener — `src/routes/root.ts`

Find the IIFE block that registers the `add-url` input listener (search for `addUrlInput`). It ends with:

```typescript
    "      updateSelectorSuggestions('add-url', 'add-selectorList');",
    "    }",
    "  })();",
```

Insert the blur listener **between** the closing `}` of the `addUrlInput` block and `})();`:

```typescript
    "    var addKeyInput = document.getElementById('add-key');",
    "    if (addKeyInput) {",
    "      addKeyInput.addEventListener('blur', function() {",
    "        var key = addKeyInput.value.trim().toLowerCase();",
    "        if (!key) return;",
    "        fetch('/config/channels/predefined-defaults?key=' + encodeURIComponent(key))",
    "          .then(function(r) { return r.json(); })",
    "          .then(function(data) {",
    "            if (!data.found) return;",
    "            var fields = [",
    "              { id: 'add-name', value: data.name },",
    "              { id: 'add-url', value: data.url },",
    "              { id: 'add-stationId', value: data.stationId },",
    "              { id: 'add-channelSelector', value: data.channelSelector }",
    "            ];",
    "            for (var i = 0; i < fields.length; i++) {",
    "              var el = document.getElementById(fields[i].id);",
    "              if (el && !el.value) { el.value = fields[i].value; }",
    "            }",
    "            if (data.profile) {",
    "              var profileEl = document.getElementById('add-profile');",
    "              if (profileEl && !profileEl.value) { profileEl.value = data.profile; }",
    "            }",
    "            updateSelectorSuggestions('add-url', 'add-selectorList');",
    "          })",
    "          .catch(function() {});",
    "      });",
    "    }",
```

---

## 2. Cancel Button Resets the Add Channel Form

### What it does
The Cancel button on the Add Channel form calls `hideAddForm()` instead of using inline JS. `hideAddForm()` hides the form, shows the Add Channel button, and calls `form.reset()` to clear all fields — so the next time the form is opened it starts blank.

### Why upstream doesn't have it
Upstream's Cancel button only toggles visibility; it does not reset field values. This causes stale values to appear the next time the form is opened.

### Files changed
- [src/routes/config.ts](../src/routes/config.ts) — Cancel button HTML

### Notes
`hideAddForm()` is defined by upstream in `src/routes/root.ts` and already includes the `form.reset()` call. No changes to root.ts are needed for this feature — only config.ts.

### Validation
- Open Add Channel form, type something in Name.
- Click Cancel.
- Open Add Channel form again — Name field must be empty.

### Implementation

#### `src/routes/config.ts` — Cancel button

Find (in `generateChannelsPanel`):

```typescript
  lines.push("<button type=\"button\" class=\"btn btn-secondary\" onclick=\"document.getElementById('add-channel-form').style.display='none'; ",
    "document.getElementById('add-channel-btn').style.display='inline-block';\">Cancel</button>");
```

Replace with:

```typescript
  lines.push("<button type=\"button\" class=\"btn btn-secondary\" onclick=\"hideAddForm();\">Cancel</button>");
```

---

## 3. Loading Slate — Stream Immediately with Logo While Channel Tunes (PLANNED, NOT IMPLEMENTED)

### What it does
Instead of making the HLS client wait 10-20 seconds for channel tuning to complete, PrismCast immediately serves a "loading" video (the PrismCast logo on a dark background) as soon as the browser capture is established. When the channel finishes tuning and video is ready, the stream naturally transitions to live TV — no codec change, no HLS discontinuity.

### Why upstream doesn't have it
Upstream publishes the stream ID only after full tuning completes. This feature moves stream publication earlier in the pipeline.

### Architecture (why this works)
Chrome's `tabCapture` API binds the capture to the **tab**, not the page. Navigating to a new URL within the same tab does not stop the MediaRecorder — it continues capturing whatever the tab renders. This means:

1. Capture starts on a local loading page (served by PrismCast itself)
2. HLS segments begin flowing immediately — clients unblock ~1-2 seconds after request
3. The tab navigates to the channel URL and tunes in the background
4. The capture stream transitions from loading page → live TV automatically
5. No init.mp4 mismatch — same MediaRecorder session throughout, same codec parameters

### Current startup sequence (in `src/streaming/setup.ts`)
```
create page → CSP bypass → getStream() [capture starts] → page.goto(channelUrl) → tune → return
```
Stream ID is published in `src/streaming/hls.ts` only AFTER `setupStream()` fully returns.

### Required startup sequence after this change
```
create page → CSP bypass → getStream() [capture starts]
  → page.goto('/loading') [local page, instant]
  → create segmenter, pipe capture → store first segments
  → publish stream ID  ← clients unblock here (~1-2s)
  → [background] page.goto(channelUrl) → tune → video ready
  → capture now shows live TV, stream transitions naturally
```

### Files to change

- `src/routes/assets.ts` — add `GET /loading` route serving `loading.html`
- `src/streaming/hls.ts` — restructure `initializeStream()` to publish stream ID early (after loading page segments exist, before tuning)
- `src/streaming/setup.ts` — split `createPageWithCapture()` into two phases: (1) capture + loading page, (2) channel navigation + tuning
- `src/streaming/monitor.ts` — suppress health-check recovery during the intentional tuning phase (the monitor will see the tab navigating away from the loading page as a potential stall)
- New file: `src/assets/loading.html` — simple HTML with centered PrismCast logo on dark background

### Key risks and mitigations

| Risk | Mitigation |
|---|---|
| Health monitor triggers recovery during tuning | Pass a "startup" flag to the monitor; suppress recovery until tuning completes or `bufferingGracePeriod` expires |
| Brief blank/white flash during navigation loading→channel | Acceptable; clients see it as a brief stall, not a stream error |
| init.mp4 reflects loading page content (wrong dimensions) | Won't happen — loading page has no `<video>` element; MediaRecorder captures the tab screen at the configured output dimensions regardless |
| Concurrent streams each navigate through loading page | Each has its own page/tab; no interaction |

### Effort estimate
- Loading page HTML + route: ~30 minutes
- Early stream publication restructure: 2-4 hours
- Monitor suppression during tuning: 1-2 hours
- Testing (all providers, concurrent streams, recovery scenarios): significant
