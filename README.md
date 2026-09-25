# MOZART Feasibility Analysis Tool

This application is part of the [MOZART project](https://mozart-robotics.eu/).

It serves as a Feasibility Analysis Tool designed to support the presentation and evaluation of innovative food packaging concepts and the handling of fragile and fresh food items in realistic production environments.

The application runs on **Meta Quest 3** and uses a **Diminish Reality** approach to combine the physical environment with virtual industrial content. This makes it possible to demonstrate and assess how future robotic and mixed reality workflows could be applied in real manufacturing settings.

## Focus

- Presentation of innovative food packaging concepts
- Exploration of handling scenarios for fragile and fresh food items
- Feasibility analysis in a real production context
- Mixed reality visualization on Meta Quest 3 using the Diminish Reality concept

## Getting started

- **Unity 6000.3.10f1** (see `ProjectSettings/ProjectVersion.txt`), Android build target for Meta Quest 3.
- Clone with submodules — the shared package lives in `Packages/mozart-unity-shared`:

  ```bash
  git clone --recurse-submodules git@github.com:robofit/mozart-unity-fat.git
  # or, in an existing clone:
  git submodule update --init
  ```

- Add the third-party libraries that are not committed (see
  [Dependencies not committed](#dependencies-not-committed-download-separately)).
- The app needs an **ARCOR2** backend and the **mesh service** at runtime; their addresses are set
  in `CommunicationManager.ServerUri` and `MeshDownloadManager.meshServerBaseUrl`. See
  [Setup and External Dependencies](Docs/setup-and-external-dependencies.md).

Further developer documentation is in [`Docs/`](Docs/index.md).

## Object-Shaped Portals

Object-shaped portals mask a portal in the shape of a real object's geometry (instead of a
bounding box), so naturally open regions (e.g. the gap between table legs) stay visible.

### Using the portals (in-headset)

Two ToolMenu toggle buttons control **object-shaped** portals:

- **Add Portal** — aim the right-controller laser at an object (green = on a valid cluster,
  red = not), pull the trigger to turn that object's silhouette into a portal.
- **Delete Portal** — aim at an existing portal and pull the trigger to remove it.

**Box-shaped** portals are added by the **Add Box Portal** button, which spawns a ready-made bounding box in
front of you and selects it for editing — grip to move/rotate it, thumbsticks to resize. See
[Edit Mode, Object Selection and Deletion](Docs/edit-mode-and-deletion.md) and the portal section
of [Runtime Object Lifecycle](Docs/runtime-object-lifecycle.md).

Multiple portals can be active at once. Picking is by exact per-cluster `MeshCollider`
raycast (`ObjectPicker.cs`), so the selection matches the object's real shape.

### Generating the object clusters

The per-object shapes ("clusters") are produced **offline** by segmenting a room mesh, and
`ObjectPicker` loads them at runtime.

The generated cluster files are **not committed** — they can be recreated on demand from the
source mesh (and from any newer mesh once available). Only the segmentation script and the
source mesh are in the repository.

### What lives in `Assets/StreamingAssets/clusters/`

| Kept in the repo | Generated (not committed — recreate with the script) |
|---|---|
| `export_clusters_normals.py` — the segmenter (RANSAC plane removal + normals-augmented DBSCAN) | `clusterN.obj` — one mesh per detected object |
| `export_clusters.py` — original plain RANSAC+DBSCAN variant | `clusters.json` — the manifest `ObjectPicker` reads (`{count, indices[]}`) |
| `visualize_clusters.py` — preview clusters in distinct colours | `centres.txt` — per-cluster centres (debug only) |
| `mesh-3hz-4.{obj,mtl,jpg}` — the source room mesh to segment | |

### How to generate the clusters

Requires **Python 3.11** with **Open3D** and **NumPy** (Open3D supports Python 3.8–3.11).

```bash
# one-time: install the dependencies
py -3.11 -m pip install open3d numpy

# segment the mesh -> writes clusterN.obj + clusters.json (+ centres.txt) in this folder
cd Assets/StreamingAssets/clusters
py -3.11 export_clusters_normals.py
```

The script reads `mesh-3hz-4.obj`, removes the dominant planes (floor / walls / ceiling) with
RANSAC, clusters the remaining points with DBSCAN on a position+normal feature (so objects
that touch are split at the normal discontinuity), crops + simplifies each cluster's mesh, and
writes the outputs **in place** — exactly where `ObjectPicker` loads them from. After running
it, Build & Run in Unity and the portals use the freshly generated clusters. No files need to
be moved.

**Segmenting a different / newer mesh:** replace `mesh-3hz-4.obj` with the new mesh (same
filename), or edit the input filename near the top of `export_clusters_normals.py`, then re-run.
The room mesh itself is normally downloaded at runtime from the external mesh server via
`MeshDownloadManager`; a copy (`mesh-3hz-4.obj`) is kept here so the segmentation can be run
offline.

`visualize_clusters.py` can be run afterwards to preview the generated clusters in distinct
colours for a sanity check.

## Room Scanning (RGB dataset capture)

The **Scan Room** button records an RGB image dataset on the headset for later offline 3D
reconstruction (e.g. COLMAP / OpenMVS). Press to start, press again to stop; a live counter
shows the number of captured photos.

Each session is written to `Application.persistentDataPath/dataset_<sessionId>/` on the
device (`/sdcard/Android/data/cz.fitvut.fat/files/dataset_<sessionId>/`):

- `frames/left_000001.jpg …` — RGB photos
- `poses/frames.csv` — per-frame camera position, rotation, and timestamp
- `calibration/left_camera.json` — camera intrinsics (fx, fy, cx, cy) at the frame resolution,
  plus the raw sensor values and lens offset
- `manifest.json` — session id and frame count

Each scan gets its own timestamped folder (the `<sessionId>` is the scan start date and time,
e.g. `dataset_2026-07-08_11-09-31`), so scans never overwrite each other.

**Extracting a dataset — one click:** with the headset connected via USB, double-click
`pull_scan_run.bat` (repo root). It finds the newest `dataset_*` on the headset, pulls it to
`pulled_scans/`, and opens the photos folder — no manual commands.

Or manually with `adb`:

```bash
adb shell ls /sdcard/Android/data/cz.fitvut.fat/files/       # list sessions
adb pull /sdcard/Android/data/cz.fitvut.fat/files/dataset_<sessionId> .
```

The reconstruction of a mesh from these images is future work; the mesh would then feed the
same segmentation pipeline as the object-shaped portals above.

## MRUK Dynamic Pipeline (experimental — disabled by default)

A second, experimental route obtains the room mesh **directly from the headset's MRUK scan**
(instead of reconstructing it offline from images) and segments it live in the same session:

`MRUK scan → GlobalMeshProvider → MeshSegmentationClient → segmentation server → ObjectPicker`

Files: `Assets/Scripts/Mesh/GlobalMeshProvider.cs`, `MeshSegmentationClient.cs`,
`ScanRoomFlow.cs`, and the off-device Flask server `tools/segmentation_server/app.py`
(wraps `export_clusters_normals.py`).

The full pipeline is implemented and coordinate conversion is verified, but the coarse MRUK
mesh does not segment as reliably as the pre-scanned mesh. It is therefore **disabled by
default** (`ScanRoomFlow.drivePortalsFromScan = false`); the pre-generated clusters drive the
portals. Set the flag to `true` to experiment with the live pipeline.

To run the server (on a laptop on the same Wi-Fi as the headset):

```powershell
py -3.11 -m pip install flask open3d numpy
py -3.11 tools/segmentation_server/app.py
# then set MeshSegmentationClient.segmentServerUrl to http://<laptop-ip>:5000
```

## Note on dynamic occlusion

The portal-content occlusion shader (`PortalContentUnlit.shader`) currently uses box-based
occlusion (`PortalFrontFaceWorld`, driven by `_PortalBoxCenter` / `_PortalBoxExtents`), so it
occludes **box-shaped** portals correctly but **not** the object-shaped cluster portals (an
arbitrary silhouette has no box). A shape-agnostic approach — reconstructing the opening depth
from `_CameraDepthTexture`, which works for any mask shape — is the intended follow-up.

## Dependencies not committed (download separately)

Following the project convention, third-party libraries are not committed. Place them into
`Assets/3rdParty/` (ignored by git) before building:

- **TriLib** and **SimpleCollada** (both required by `MeshImporter.cs`) — taken from the private
  Robo@FIT package source; see [Setup and External Dependencies](Docs/setup-and-external-dependencies.md)
  for how to get access.

For the offline segmentation and the experimental MRUK server: **Python 3.11** with
**Open3D**, **NumPy**, and (for the server) **Flask**.

## License

This project is licensed under the [Apache License 2.0](LICENSE).

Third-party components used by the project (e.g. the Meta XR SDK, TriLib, SimpleCollada) remain
under their own licenses.
