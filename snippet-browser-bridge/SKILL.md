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

3. Read state first:

```js
const canvas = window.__snippetCodexCanvas.readState()
const workbench = window.__snippetCodexWorkbench.readState()
```

4. Decide from state, not visual guessing.
5. Run only the needed command.
6. Read state again and verify the result.

## Canvas Surface

Use `window.__snippetCodexCanvas`.

Available commands:

```js
readState()
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

## Workbench Surface

Use `window.__snippetCodexWorkbench`.

Available commands:

```js
readState()
setPublicShare(isPublic)
createCommit({ name, message, objectIds })
setCommitTargets(objectIds)
openPanel(kind)
openTimeline()
selectCommit(commitId)
returnToCurrent()
```

Workbench state includes room/share/permission/current-or-past/timeline/commit target/loading/error information.

## Safety Rules

- If `canvas.readOnly` is true, do not call mutation commands.
- If `canvas.hasEditPermission` is false, do not call mutation commands.
- If `workbench.hasEditPermission` is false, do not create commits.
- Do not change public share state unless the user asked for sharing/publication or it is required by the task.
- Do not use coordinate dragging when `arrangeObjects` or `updateObjects` can express the operation.
- Do not treat hidden or invisible objects as editable targets unless the user explicitly asks to investigate visibility.

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
