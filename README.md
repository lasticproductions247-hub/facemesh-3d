# FaceMesh 3D — Interactive Portrait Animator

Turn a portrait photo into an animated 3D face mesh. **Everything runs in your
browser** — the photo is never uploaded, and there is no server, API key or
backend anywhere in this project.

## Live

https://lasticproductions247-hub.github.io/facemesh-3d/

## What it does

1. You drop in a portrait photo.
2. MediaPipe Face Mesh detects **468 facial landmarks** on-device via WASM.
3. Those landmarks are turned into a real 3D mesh using MediaPipe's own
   face topology — **854 triangles** across 468 vertices.
4. You can turn, nod, change the relief depth, and switch between an animated
   idle sway and direct pointer control.
5. Export the result as a `.OBJ` you can open in Blender, MeshLab or Maya, or
   save a PNG snapshot.

## The part that actually matters: the triangulation

The naive way to turn a landmark list into a mesh is to stitch consecutive
points:

```js
for (let i = 0; i < points.length - 1; i++) {
  if (i % 3 === 0) indices.push(i, i + 1, i + 2);
}
```

That looks plausible and produces garbage — landmark order in MediaPipe is
*not* a surface scan, so those triangles criss-cross the face into a tangle.

MediaPipe ships the correct answer as `FACEMESH_TESSELATION`, but it is an
**edge list** (2,556 `[a, b]` pairs), not triangles. A renderer needs triangles,
so `tri.js` derives them: every 3-cycle in that edge graph is a triangle.

```js
for (const [a, b] of FACEMESH_TESSELATION) {
  adj[a].add(b); adj[b].add(a);
}
for (const a of adj) for (const b of adj[a]) for (const c of adj[b]) {
  if (c !== a && adj[a].has(c)) triangles.add(sort(a, b, c));
}
```

2,556 edges → **854 triangles**, all indices in `0..467`. `tri.js` is
generated from the table, never hand-typed.

## Dependencies

All loaded from a CDN at runtime, pinned to versions verified to serve the
assets the library actually requests:

| What | Version | Why pinned |
|---|---|---|
| three.js | r128 (cdnjs) | renderer |
| OBJExporter | three@0.128.0 (jsDelivr) | `.obj` export |
| @mediapipe/face_mesh | 0.4.1633559619 | landmark detection |

Note on MediaPipe assets: the WASM and model files live at the **package root**
on jsDelivr, not under `/wasm/`. `locateFile` resolves against the pinned
package base, which is what the loader actually asks for.

`refineLandmarks` is deliberately **off**. It returns 478 points, but the
tesselation indexes the canonical 468-point mesh.

## Privacy

No network calls carry your image anywhere. Inference is local WASM, the object
URLs are revoked when you reset, and there is no analytics.

## Local use

It is a static site — no build step:

```bash
python -m http.server 8000
# then open http://localhost:8000
```

`tri.js` must be served alongside `index.html` (it is loaded with a relative
path, so opening the file over `file://` will also work in most browsers).

## Licence

MIT
