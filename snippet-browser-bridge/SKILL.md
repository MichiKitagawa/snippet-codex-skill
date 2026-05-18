---
name: snippet-browser-bridge
description: Operate the Snippet app through its browser-exposed Codex bridge. Use when the user asks Codex to inspect, organize, edit, arrange, connect, commit, share, or verify a Snippet room/canvas/workbench in the in-app browser or through Playwright/browser automation. This skill is for Snippet-specific browser bridge operation, not MCP/server-side tool design.
---

# Snippet Browser Bridge

## Goal

Use Snippet's browser bridge as the primary operation path when it exists. Use ordinary browser clicks only for navigation, visual confirmation, or UI that is not exposed by the bridge.

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

4. For a new or unfamiliar room, investigate before acting:

```js
window.__snippetCodexCanvas.searchObjects({ query: "topic" })
window.__snippetCodexCanvas.getObject(objectId)
window.__snippetCodexWorkbench.listChangedObjects()
window.__snippetCodexWorkbench.listCommits()
```

5. Decide from room memory and state, not visual guessing.
6. Run only the needed command.
7. Read memory/state again and verify the result.

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
  selectedObjectIds,
  viewState,
  objects,
  relationships
}
```

Object rows include stable `id`, `type`, `contentData`, `contentText`, `position`, `size`, `visible`, and `containingFrameId`.

Room memory commands:

- `readRoomIndex()` returns room identity, read-only/current-or-past state, object counts, type counts, frames, object previews, and relationships.
- `listObjects(filter)` returns readable object rows with `id`, `type`, `title`, `previewText`, `searchableText`, `labels`, placement, frame membership, relationships, timestamps, `visible`, and `readable`.
- `getObject(objectId)` returns one readable object row or `null`.
- `searchObjects(...)` searches readable object title, preview text, searchable text, labels, URL, description, and type-specific readable metadata.

Use room memory commands instead of inferring meaning from the canvas image.

## Workbench Surface

Use `window.__snippetCodexWorkbench`.

Available commands:

```js
readState()
listChangedObjects()
listCommits()
setPublicShare(isPublic)
createCommit({ name, message, objectIds })
setCommitTargets(objectIds)
openPanel(kind)
openTimeline()
selectCommit(commitId)
returnToCurrent()
```

Workbench state includes room/share/permission/current-or-past/timeline/commit target/loading/error information.

`listChangedObjects()` returns readable summaries for uncommitted changed objects and whether each object is currently a commit target.

`listCommits()` returns commit rows with id, name, message, createdAt, objectIds, object count, and readable object previews when available.

## Safety Rules

- If `canvas.readOnly` is true, do not call mutation commands.
- If `canvas.hasEditPermission` is false, do not call mutation commands.
- If `workbench.hasEditPermission` is false, do not create commits.
- Do not change public share state unless the user asked for sharing/publication or it is required by the task.
- Do not use coordinate dragging when `arrangeObjects` or `updateObjects` can express the operation.
- Do not treat hidden or invisible objects as editable targets unless the user explicitly asks to investigate visibility.
- Do not assume missing search results mean an external fact is false; they only mean the current room memory did not expose a readable match.
- Do not claim to have read private, hidden, paid, or file-body content unless the bridge returned it.

## Room Investigation

For a fresh room, use this sequence:

```js
const index = window.__snippetCodexCanvas.readRoomIndex()
const matches = window.__snippetCodexCanvas.searchObjects({ query: "keyword", limit: 10 })
const detail = matches[0] ? window.__snippetCodexCanvas.getObject(matches[0].id) : null
const changed = window.__snippetCodexWorkbench.listChangedObjects()
const commits = window.__snippetCodexWorkbench.listCommits()
```

Use `listObjects({ type })` or `listObjects({ labels })` when the user asks for a class of room objects. Use `getObject(id)` before editing a specific object so you understand its current content.

## Common Operations

Create objects:

```js
await window.__snippetCodexCanvas.createObjects([
  {
    type: 'sticky',
    contentData: { text: 'Idea' },
    position: { x: 200, y: 160, z: 1 },
    size: { width: 180, height: 140 }
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
await window.__snippetCodexCanvas.createRelationship({
  fromObjectId,
  toObjectId,
  label: 'supports'
})
```

Commit changes:

```js
await window.__snippetCodexWorkbench.setCommitTargets(objectIds)
await window.__snippetCodexWorkbench.createCommit({
  name: 'Update',
  message: 'Organized through Codex',
  objectIds
})
```

## Fallback

If the bridge is missing, use normal browser automation to navigate and inspect the page. Do not invent bridge behavior. Report that the Snippet bridge is unavailable on the current page.
