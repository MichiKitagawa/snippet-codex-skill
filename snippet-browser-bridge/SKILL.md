---
name: snippet-browser-bridge
description: Operate Snippet rooms through the browser-exposed Codex bridge. Use when Codex needs to inspect, organize, edit, style, arrange, connect, commit, share, or verify a Snippet room in the in-app browser or Playwright. This skill is for Snippet-specific room/canvas/workbench operation, not MCP/server-side tool design.
---

# Snippet Browser Bridge

## Goal

Use Snippet's browser bridge as the primary operation path whenever it exists.

The current Snippet editor is a 2D room canvas with a left tool rail, a center object canvas, a right inspector, and a bottom commit timeline. Do not assume a 3D scene, WebGL renderer, camera orbit, raycast picking, or 3D transform workflow. `position.z` still exists as ordering/depth data, but ordinary editing should treat objects as 2D canvas objects with stable ids, content, appearance, position, size, visibility, relationships, and commit state.

Use ordinary browser clicks only for navigation, visual confirmation, or UI that is not exposed by the bridge.

## Required Order

1. Open or select the Snippet room page.
2. Check for the bridge before operating:

```js
Boolean(window.__snippetCodexCanvas)
Boolean(window.__snippetCodexWorkbench)
```

3. Read room memory first, then operational state:

```js
const index = window.__snippetCodexCanvas.readRoomIndex()
const canvas = window.__snippetCodexCanvas.readState()
const workbench = window.__snippetCodexWorkbench.readState()
```

4. For a new or unfamiliar room, investigate before acting. For a fresh room, use this sequence:

```js
window.__snippetCodexCanvas.listObjects({ visibleOnly: true, limit: 50 })
window.__snippetCodexCanvas.searchObjects({ query: "topic", limit: 10 })
window.__snippetCodexCanvas.getObject(objectId)
window.__snippetCodexWorkbench.listChangedObjects()
window.__snippetCodexWorkbench.readDraftReview()
window.__snippetCodexWorkbench.listCommits()
```

5. For creator workflow tasks, build or inspect the shared selection and commit target set before committing:

```js
const selectedIds = window.__snippetCodexWorkbench.selectObjectsByFilter({ query: "topic", changedOnly: true })
const draft = window.__snippetCodexWorkbench.readDraftReview()
window.__snippetCodexWorkbench.setCommitTargets(selectedIds)
```

6. Decide from room memory, draft review, and state, not visual guessing.
7. Run only the needed command.
8. Read memory/state again and verify the result.

## Canvas Surface

Use `window.__snippetCodexCanvas`.

Available commands:

```js
readState()
readRoomIndex()
listObjects(filter)
getObject(objectId)
searchObjects({ query, type, labels, containingFrameId, visibleOnly, limit })
selectObjects(ids)
createObjects(inputs)
updateObjects(updates)
arrangeObjects(input)
updateCanvasBackground(input)
updateObjectDefaultAppearance(input)
updateObjectAppearance({ objectId, appearance })
createReusableStyleToken({ objectId, name })
deleteObjects(ids)
createRelationship({ fromObjectId, toObjectId, label })
deleteRelationship(relationshipId)
```

State includes:

```js
{
  roomId,
  readOnly,
  viewingCommitId,
  userRole,
  hasEditPermission,
  appearance,
  objectPresets,
  selectedObjectId,
  selectedObjectIds,
  selectedObjectLinkId,
  viewState,
  objects,
  relationships
}
```

Object rows include stable `id`, `type`, `contentData`, `contentText`, `appearance`, `position`, `size`, `labels`, `isMainContent`, `visible`, `containingFrameId`, and `updatedAt`. In R3.2, `appearance.presetId` identifies the Object Preset used by that object when one is set.

Supported create/update object types:

```js
'text'
'document'
'sticky'
'image'
'video'
'audio'
'link'
'shape'
'line'
'frame'
'sketch'
```

Create input:

```js
{
  type,
  contentData,
  presetId,
  position: { x, y, z },
  size: { width, height },
  labels,
  isMainContent
}
```

Update input:

```js
{
  id,
  contentData,
  presetId,
  appearance,
  position: { x, y, z },
  size: { width, height },
  labels,
  isMainContent
}
```

Arrange input:

```js
{
  ids,
  layout: 'row' | 'column' | 'grid',
  start: { x, y, z },
  gap: { x, y }
}
```

## Room Memory

- `readRoomIndex()` returns room identity, read-only/current-or-past state, object counts, type counts, frames, object previews, and relationships.
- `listObjects(filter)` returns readable object rows with `id`, `type`, `title`, `previewText`, `searchableText`, `labels`, placement, frame membership, relationships, timestamps, `visible`, and `readable`.
- `getObject(objectId)` returns one readable object row or `null`.
- `searchObjects(...)` searches readable title, preview text, searchable text, labels, URL, description, and type-specific readable metadata.

Use room memory commands instead of inferring meaning from the canvas image. Use browser screenshots only to confirm layout, overlap, readability, or user-visible presentation.

## Appearance

R3.2 exposes room style, Object Presets, and object style state through `canvas.appearance`, `canvas.objectPresets`, object `appearance.presetId`, and appearance commands.

Object Presets are renderer families for the same Snippet Object structure. They are not skins, purpose categories, room templates, or minor token changes. Choosing a preset should preserve the same Object types, count, content, position, relationships, permissions, commit behavior, and viewer/read-only boundaries while changing the object rendering genre.

Official R3.2 Object Presets:

```js
'modern-saas'
'neo-brutalism'
'glassmorphism'
'pixel-retro'
'holographic'
```

Use the product names returned by `canvas.objectPresets` when showing or reporting them. `modern-saas` is the existing baseline. Do not rename presets as task outcomes such as brainstorming, research, command board, or gallery.

Set an Object Preset when creating or updating objects:

```js
const ids = await window.__snippetCodexCanvas.createObjects([
  {
    type: 'document',
    presetId: 'glassmorphism',
    contentData: { title: 'Research brief', content: 'Context\\nEvidence\\nDecision' },
    position: { x: 420, y: 160, z: 1 },
    size: { width: 320, height: 220 }
  }
])

await window.__snippetCodexCanvas.updateObjects([
  { id: ids[0], presetId: 'neo-brutalism' }
])
```

Per-object appearance overrides still apply on top of the selected preset. `updateObjectAppearance` changes overrides or style tokens; it should not be used as a substitute for preset selection when the task is to change renderer family.

When verifying Object Presets, create or inspect all 11 toolbar Object types: `text`, `document`, `sticky`, `image`, `video`, `audio`, `link`, `shape`, `line`, `sketch`, `frame`. Do not judge a preset from only card-like objects. Type-specific expectations matter:

- `image`, `video`, and `audio` must render as media objects, not text cards.
- `link` must expose URL/link affordance, not only plain text.
- `document` must read as a document object, not a sticky note clone.
- `sketch` is a drawable canvas surface; do not pre-fill it with decorative scribbles unless user content contains drawing data.
- `line` is a line object; do not place it inside a card.
- `frame` is a frame/boundary; do not render it as a content card.
- `shape` should not inherit irrelevant text-card accents.

Canvas background:

```js
await window.__snippetCodexCanvas.updateCanvasBackground({
  color: '#EEF6F1',
  gridMode: 'none' | 'dots' | 'grid',
  paperStrength: 0.45,
  gridOpacity: 0.12
})
```

Do not expose or rely on dot/grid size as a normal user choice. The UI treats pattern size as fixed and exposes pattern mode plus overall strength.

Object default appearance:

```js
await window.__snippetCodexCanvas.updateObjectDefaultAppearance({
  surface: { color: '#eef6f1', opacity: 0.96, strength: 'soft' }
})
```

Per-object appearance:

```js
await window.__snippetCodexCanvas.updateObjectAppearance({
  objectId,
  appearance: {
    surface: { color: '#fef3c7', opacity: 0.96, strength: 'soft' },
    border: { width: 1, style: 'solid', visible: true },
    radius: 8,
    density: 'normal',
    typography: { scale: 'normal' },
    accent: { color: '#0f766e', placement: 'left' }
  }
})
```

Reusable style token:

```js
const token = await window.__snippetCodexCanvas.createReusableStyleToken({
  objectId,
  name: 'Research card'
})
await window.__snippetCodexCanvas.updateObjectAppearance({
  objectId: otherObjectId,
  appearance: { tokenId: token.id }
})
```

When changing style, verify both state and visible result:

```js
const state = window.__snippetCodexCanvas.readState()
state.appearance
state.objectPresets
state.objects.find((object) => object.id === objectId)?.appearance
```

## Workbench Surface

Use `window.__snippetCodexWorkbench`.

Available commands:

```js
readState()
listChangedObjects()
readDraftReview()
listCommits()
setPublicShare(isPublic)
createCommit({ name, message, objectIds })
setCommitTargets(objectIds)
selectObjectsByFilter({ query, type, labels, containingFrameId, changedOnly, limit })
openPanel(kind)
openTimeline()
selectCommit(commitId)
returnToCurrent()
```

Workbench state includes room/share/permission/current-or-past/timeline/selection/commit target/loading/error information.

`listChangedObjects()` returns readable summaries for uncommitted changed objects, high-level `changeTypes`, and whether each object is currently a commit target.

`readDraftReview()` returns a reviewable creator workflow summary:

```js
{
  changedObjects,
  selectedObjectIds,
  commitTargetIds,
  changeTypeCounts
}
```

Use `readDraftReview()` before creating commits so the commit target is intentional.

`selectObjectsByFilter(...)` builds the shared selection set from readable room objects. Use it when a task asks to find, organize, or commit a class of objects by keyword, type, labels, frame, or changed state.

`listCommits()` returns commit rows with id, name, message, createdAt, objectIds, object count, and readable object previews when available.

## Safety Rules

- If `canvas.readOnly` is true, do not call mutation commands.
- If `canvas.hasEditPermission` is false, do not call mutation commands.
- If `workbench.hasEditPermission` is false, do not create commits.
- Do not change public share state unless the user asked for sharing/publication or it is required by the task.
- Do not use coordinate dragging when `arrangeObjects`, `updateObjects`, or appearance commands can express the operation.
- Do not treat hidden or invisible objects as editable targets unless the user explicitly asks to investigate visibility.
- Do not assume missing search results mean an external fact is false; they only mean the current room memory did not expose a readable match.
- Do not claim to have read private, hidden, paid, or file-body content unless the bridge returned it.
- Do not bypass permission, visibility, pricing, commit, or viewer boundaries with direct backend calls while operating a room through the browser bridge.

## Common Operations

Investigate a room:

```js
const index = window.__snippetCodexCanvas.readRoomIndex()
const objects = window.__snippetCodexCanvas.listObjects({ visibleOnly: true, limit: 50 })
const changed = window.__snippetCodexWorkbench.listChangedObjects()
const draft = window.__snippetCodexWorkbench.readDraftReview()
const commits = window.__snippetCodexWorkbench.listCommits()
```

Create objects:

```js
const ids = await window.__snippetCodexCanvas.createObjects([
  {
    type: 'sticky',
    contentData: { text: 'Idea' },
    position: { x: 200, y: 160, z: 1 },
    size: { width: 180, height: 140 }
  },
  {
    type: 'document',
    contentData: { title: 'Research brief', content: 'Context\\nEvidence\\nDecision' },
    position: { x: 420, y: 160, z: 1 },
    size: { width: 320, height: 220 }
  }
])
```

Update content and placement:

```js
await window.__snippetCodexCanvas.updateObjects([
  {
    id: ids[0],
    contentData: { text: 'Refined idea' },
    position: { x: 240, y: 180 },
    size: { width: 220, height: 150 }
  }
])
```

Arrange objects:

```js
await window.__snippetCodexCanvas.arrangeObjects({
  ids,
  layout: 'row',
  start: { x: 160, y: 180, z: 1 },
  gap: { x: 40, y: 24 }
})
```

Connect objects:

```js
const relationshipId = await window.__snippetCodexCanvas.createRelationship({
  fromObjectId,
  toObjectId,
  label: 'supports'
})
```

Style objects:

```js
await window.__snippetCodexCanvas.updateObjectAppearance({
  objectId,
  appearance: {
    surface: { color: '#eef6f1', opacity: 0.96, strength: 'soft' },
    accent: { color: '#0f766e', placement: 'left' }
  }
})
```

Set Object Presets:

```js
await window.__snippetCodexCanvas.updateObjects([
  { id: objectId, presetId: 'holographic' }
])
const state = window.__snippetCodexCanvas.readState()
state.objects.find((object) => object.id === objectId)?.appearance?.presetId
```

Commit changes:

```js
const review = window.__snippetCodexWorkbench.readDraftReview()
await window.__snippetCodexWorkbench.setCommitTargets(objectIds)
await window.__snippetCodexWorkbench.createCommit({
  name: 'Update',
  message: 'Organized through Codex',
  objectIds
})
```

Find and select changed objects:

```js
const selectedIds = window.__snippetCodexWorkbench.selectObjectsByFilter({
  query: 'research',
  changedOnly: true,
  limit: 20
})
const review = window.__snippetCodexWorkbench.readDraftReview()
```

## Visual Confirmation

Use screenshots or DOM checks after bridge operations when the user cares about layout or UI quality. Confirm:

- objects are visible and not unintentionally overlapping
- selected object and right inspector agree
- canvas appearance matches the requested background/pattern/style
- each Object Preset visibly changes the renderer family while preserving the same object structure
- all 11 Object types remain type-specific under each preset
- viewer and past commit modes remain read-only
- hidden or unreadable objects are not exposed

## Fallback

If the bridge is missing, use normal browser automation to navigate and inspect the page. Do not invent bridge behavior. Report that the Snippet bridge is unavailable on the current page.
