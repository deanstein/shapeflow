# ShapeFlow Architecture

## 1. Introduction and vision

**ShapeFlow** is an easy-to-use 3D modeler aimed at architectural and industrial designers. It combines a familiar modeling workflow with parametric rules called *Flows* and a node-based *FlowChart* that reflects the model hierarchy and applied flows.

**Differentiators**

- **Flows:** Parametric rules or actions attached to geometry (e.g. “split face into panels and extrude”). When the source shape changes, flows can be re-run to regenerate derived geometry.
- **FlowChart:** A 2D (or possibly 3D) node graph always visible next to the 3D view, showing instance/geometry relationships and flows. Selection in the FlowChart equals selection in the viewport and drives the Properties panel.
- **3D sketching:** The modeling experience and capabilities are built around **3D sketching** workflows: drawing on surfaces (e.g. a line on a face that splits it into two faces), then operating on the result (extrude one face, apply a flow, etc.). The kernel (OpenCascade) supports splitting faces by on-surface geometry; ShapeFlow exposes this so the designer can sketch directly on the model and refine it face by face.
- **Local-first:** Core modeling runs on local compute with local storage to avoid cloud cost and latency; no dependency on the cloud for day-to-day modeling.

**Target platforms**

- **Web app:** Broader reach, easier onboarding. Performance and feature set are intentionally more limited (e.g. WASM/worker constraints).
- **Electron app:** For power users. More speed, more memory, and potentially more features (e.g. native OpenCascade or heavier WASM usage). A freemium model is under consideration: web free, Electron paid.

---

## 2. Core concepts

### Shape

A **Shape** is anything the user can select in the model: an instance, a body, a face, an edge, etc. Each Shape has **Properties**, which include:

- **Attributes:** Metadata (e.g. string tags, numbers) attached to that Shape.
- **Flows:** Rules or actions that operate on that Shape and can produce or update geometry.

### Containers: Groups and Components

Geometry can be grouped into **containers**. Two kinds are supported:

- **Group** — A container whose copies are independent. Duplicating a group produces a separate copy; editing one does not affect the others.
- **Component** — A container whose copies are linked (instanced). Duplicating a component produces instances that share a definition; editing the definition (or one instance) propagates to all instances. Useful for repeated elements (e.g. windows, columns).

Both are selectable Shapes and can hold other geometry (and nested groups or components). Groups and components can have their own **local coordinate system**, including origin and axial alignment, so that drawing, snapping, and placement inside the container (or when placing instances) use that frame.

### Flows

**Flows** are rules or actions attached to a Shape. For example, a planar face might have a Flow that “splits into panels and extrudes those panels.” When the face’s geometry changes, that Flow can be re-run (by the user or automatically) to regenerate the panels. Flows are the main mechanism for parametric, repeatable behavior.

### FlowChart

The **FlowChart** is an interface element that is always present alongside the 3D model view. It is a 2D (or possibly 3D) abstracted, node-based representation of the model hierarchy. It:

- Shows instance and geometry relationships.
- Shows which Flows are applied to each node.
- Stays in sync with the model—when the model updates, the FlowChart updates.
- Shares selection with the viewport: selecting a node in the FlowChart selects the corresponding geometry or instance in the 3D view.
- Drives the **Properties panel:** the selected node’s Shape determines what Attributes and Flows are shown and editable.

Together, the FlowChart, viewport, and Properties panel form a triad: one selection state, multiple views.

### Flow Creator (authoring flows)

- A **Flow Creator** is the mechanism by which users define or edit the rules and actions that make up a Flow applied to a Shape. Two main approaches:

  - **Script interface:** Users write code (e.g. a small DSL or JavaScript/TypeScript) that calls the application API. Pros: full expressiveness, easy to version-control and share as text, fast to implement. Cons: steeper for non-programmers (many architects and trades prefer not to code), less discoverable.

  - **Node-based interface:** Users build flows in a visual node graph (similar to the FlowChart, but for authoring logic rather than browsing hierarchy). Pros: discoverable, approachable for non-coders, aligns with the existing node metaphor in the app. Cons: more work to build and maintain, can get messy for complex logic, version control is trickier (serialized graphs).

- **Recommendation:** Define the API and a small script/DSL surface first so that every flow has a clear, executable representation. Ship a script-based Flow Creator (or even “paste script” per flow) to get real use and feedback. Then add a **node-based Flow Creator** that targets the same API—nodes become a visual wrapper that generates or configures the same script/API calls. That way one execution model supports both: power users can write script, others can wire nodes, and the system stays consistent. If the audience is heavily non-technical, prioritizing the node-based creator earlier may be worth it.

---

## 3. Technical architecture

### Languages

- **TypeScript** is used for the UI layer: Svelte components, shell logic, and the code that calls the application API. This keeps the interface fast to iterate on and type-safe where it matters most for product behavior.
- **Rust** is used for the engine: flow execution, scene graph, persistence, and the integration with OpenCascade (via C API or a Rust crate). The same Rust engine compiles to **WASM** for the web app and to **native** for the Electron app, so one codebase serves both platforms. The UI (TypeScript/Svelte) is a thin client that calls into this engine.

### Interface

- **Svelte** powers the user interface for both the web app and the Electron app. The viewport, FlowChart, Properties panel, and all shell UI are built with Svelte (and TypeScript), giving a single UI stack across platforms and keeping the front end reactive and maintainable.

### Application API

- ShapeFlow exposes its **own API** that all tools, buttons, and automation use. The API is built incrementally as features are added: every UI action (create shape, run flow, set attribute, etc.) goes through this API. The UI is a thin client that calls the API; the API is the single integration point for the engine, for future scripting, and for any Flow Creator (see below). This keeps behavior consistent, testable, and reusable.

### Modeling kernel

- **OpenCascade (C++)** is the intended modeling kernel for geometry and topology. If the engine is later split into a separate (e.g. private) repo, a stable API/ABI boundary should be defined so the rest of the app depends on that contract rather than kernel internals.

### Web app

- Runs in the browser. OpenCascade would need to be used via a WASM build; there is no native C++ in the browser.
- Performance is more limited (WASM, main thread vs workers, memory). This is an explicit tradeoff for reach and ease of use.
- Feature set may be a subset of the Electron app (e.g. fewer or lighter Flow types, smaller models).

### Electron app

- Full access to native code or to the same WASM build with more CPU and memory. Enables higher performance and potentially more features (e.g. more Flow types, larger models, native kernel).
- Target for “super users” and, optionally, paid/freemium.
- **Multiple windows** — Desktop users can have multiple ShapeFlow windows (or instances) open at once, each with its own model, so several projects or views can be worked on side by side.

### Local-first

- All core modeling, Flow execution, and persistence are designed to run locally. No requirement for cloud compute or cloud storage for the core product. This keeps cost and latency low and avoids vendor lock-in for user data.

### Test suite

- A **full test suite** covers all public APIs and critical functions. Tests are run on every change (e.g. in CI). New tests are added as new APIs or behavior are developed so that coverage and confidence stay high. This applies to both the engine (Rust) and the UI/API layer (TypeScript) where appropriate.

### Journaling

- **Journaling** is an internal mechanism that records fine-grained API calls and events as the user takes actions (tool use, flow execution, selection, etc.). The journal serves debugging, auditing, and as the underlying data for user-facing **Flow Recording** (see Planned features): Flow Recording may be built on the same technology, exposing a higher-level or exportable view of the journal. One implementation can support both—fine-grained logging for developers and a simplified “record / replay” experience for users.

---

## 4. Planned features

The following features are intended for ShapeFlow. Order and grouping are for clarity; implementation priority is tracked in section 6.

### General UX

- **Commit with Enter or Escape** — In any tool with live preview or in-progress geometry, **Enter** or **Escape** commits the current changes (accept/cancel or confirm, as appropriate). Consistent across tools so the designer always knows how to finish or bail.
- **Type dimensions anytime** — Whenever a dimension is visible on screen (length, distance, angle, etc.), the user can type a value to set it. No need to click a field first; focus can stay on the canvas.
- **Temporary preview graphics** — Tools that create or transform geometry (e.g. drawing a line, offsetting, copy, array) show **temporary** preview graphics before execution. The designer sees the result in place; committing (e.g. Enter) applies it, cancel (Escape) discards it. Same pattern across tools for consistency.
- **Hide rest of model (H)** — When editing inside a group or component, tapping **H** toggles hiding or showing the context outside that container (the rest of the model). Reduces clutter while working on the contents; tap H again to bring the rest back.
- **Double-click to edit** — Double-click a group or component to enter it for editing. Consistent way to step into a container.

### Selection

- **Direct selection** — Single-click to select an edge, face, or instance. Modifiers:
  - **Ctrl** — Add to selection.
  - **Shift** — Add or remove from selection (toggle).
  - **Alt** — Remove from selection.
- **All connected** — Double-click any raw geometry (edge, face) to select all connected geometry (e.g. the whole body).
- **Area selection (window)** — Click and drag to draw a selection window:
  - **Left to right** — Only objects fully inside the box are selected.
  - **Right to left** — Anything touching the box is selected.
  - The same **Ctrl**, **Shift**, and **Alt** modifiers apply: add to selection, toggle in/out, or remove from selection.

### 3D canvas and visual styles

- **3D canvas** with controls for visual styles:
  - **Surfaces:** Ambient brightness, ambient contrast, shadow intensity, monochrome mode, default surface colors, section cut line width and color, section poche color.
  - **Edges:** Visibility, color, contrast.
  - **Environment:** Grid, axes, north arrow, sky color(s), ground plane transparency and color.
  - **Diagnostics:** Watertight issues, back faces.
  - **Highlights:** Option for **hover highlights** (faces, edges, vertices, containers) when the cursor is over them; and **selection highlights** for the current selection. Both can be toggled or adjusted so the designer can choose how much feedback they want.

### FlowChart canvas

- **FlowChart canvas** — Interactive, always mirroring the 3D scene. Selection in the FlowChart selects the corresponding elements in the 3D canvas (and vice versa).

### Location

- **Set Location** — Gets flat satellite image and 3D terrain with satellite image applied using UVs.
- **Update Location** — Refresh or change location data.

### Undo and redo

- **Undo and Redo** — Full support for reverting and replaying user actions.

### Properties

- **Properties** — Examine and change object properties, including applying Flows to shapes.

### Drawing tools

- **Drawing tools:** Vertex, Line, Offset Line, Rectangle, Arc, Polygon, Circle. When a closed loop is detected, ShapeFlow automatically creates a face that can then be extruded. Drawing on an existing face can split that face (e.g. a line on a cube face yields two faces); the designer can then extrude or otherwise operate on each resulting face—part of the 3D sketching workflow (see vision).
- **3D Text** tool.

### Move, copy, and array

- **Move tool** — Interactive move: first point (from), second point (to). Uses inferencing and snapping to keep placement accurate. Dimensions on screen; designer can type a distance.
- **Quick Copy** — While using Move, tapping **Ctrl** creates a copy instead of moving the original (Move under the hood; Ctrl toggles copy mode). Preview shows the copy before commit.
- **Array tool** — Builds an array of copies along a vector. Same interaction as Move (first point, second point; inferencing and snapping). Produces multiple copies along that vector; count and spacing can be set (e.g. on-screen or typed). Temporary preview shows the full array before execution.

### Clipboard: cut, copy, paste

- **Cut** and **Copy** — Standard clipboard: selection is cut or copied to the clipboard.
- **Paste** — Pasted geometry appears at the cursor for placement (same flow as Move: position with inferencing/snapping, then commit).
- **Paste-In-Place** — Pasted geometry is placed exactly where it was in world space. Use when moving elements out of a container, or when pasting from another model and wanting to preserve position. No cursor placement step.

### Snapping and inferencing

- **Snapping and inferencing engine:**
  - **Snap targets:** Vertices, midpoints, centroids, edges, and faces. The user can snap to these when drawing, moving, or placing geometry.
  - **Hover inferences:** While hovering over geometry, the engine can generate **hover inferences** (temporary guides) using the **local coordinate system** of the selection or container. This supports precise input and alignment, especially when working inside groups or components that have their own origin and axes.
  - **Axis lock (Shift):** During a **modification** (e.g. Move) or **drawing** (e.g. Line) tool, holding **Shift** locks to the current axis. Moving or drawing off that axis is not possible regardless of cursor position, so the designer can stay on one axis without drift.

### On-screen dimensions

- **Every tool** shows **on-screen dimensions** (lengths, distances, angles, etc.) as the designer works. The designer can read these values and can type values directly to set lengths, distances, and other numeric parameters, so that precise input is always available without leaving the current tool.

### Solid operations

- **Boolean operations:** Solid Join, Solid Intersect, Solid Cut.
- **Solid creation:** Sweep, Loft.
- **Solid modification:** Shell, Fillet.

### Face and edge tools

- **Flatten** — Force a planar face to align to another planar face; clean up edges where they formerly intersected.
- **Extrude Face** — Extrude 2D faces into a solid, and extend or trim existing solids.
- **Extrude Edge** — Extrude selected edges into one or more faces.
- **Offset Face** — Offset the perimeter edge loop on a face (creating a new loop and optionally new geometry).
- **Offset Edge** — Offset a series of edges in their plane.

### BRep and mesh conversion

- **Convert BRep to mesh** — Ability to tessellate BRep (boundary representation) geometry into meshes for lighter display and better performance, or for export/import with mesh-based formats (e.g. STL, OBJ). OpenCascade supports this via its meshing APIs (e.g. `BRepMesh_IncrementalMesh` and related), so ShapeFlow can offer explicit conversion and optionally cache or store mesh representations alongside BRep where useful.
- **Convert mesh to BRep** — Mesh-to-BRep conversion is desirable (e.g. to make imported mesh geometry editable as solid bodies). OpenCascade does not provide this out of the box; it would require implementing or integrating a solution (e.g. surface reconstruction or a dedicated pipeline) if ShapeFlow is to support it.

### Container tools

- **Groups and Components:** Create group or component, edit (enter) group or component, unlink component (break instance link), explode group or component. **Double-click** a group or component to enter it for editing.
- **Edit Container tool** — Select a nested container (group or component instance) and enter it for editing immediately, without double-clicking repeatedly through the hierarchy. Lets the designer jump directly to a deeply nested container instead of stepping in one level at a time.

### Materials

- **Material tools:** Paintbrush, Edit Placement.
- Create and edit **Materials**.
- Ability to point to a **local directory** to pull materials from.

### Layers

- **Layers:** Create, delete, hide, apply geometry to layers.

### Scenes

- **Scenes:** Remember camera state, visual styles, and layer state. Add, remove, update scenes.

### Animations

- **Animations:** Combine scenes into animations. Control order and set durations.

### Content library

- **Content library:** View all objects in the model (groups, components, and other geometry). Point to local directories for external 3D content (e.g. STL, OBJ, FBX, SKP, 3DM).

### Extensions

- **Extensions:** A way for third parties to extend the program using its API. Details TBD.

### Flow Recording

- **Flow Recording** — User-facing feature to record flows and user actions for replay (reproducibility, automation) and debugging. The recorded flow becomes a **Flow** that the user can run once as recorded or save and reuse in other contexts (e.g. apply to other shapes or projects). Recordings can be inspected, re-run, or exported. This may be built on the same technology as journaling (see Technical architecture).

### Key bindings

- **Customizable key bindings** from the start. ShapeFlow ships with a recommended set of shortcuts (tools, views, undo/redo, etc.), but the user can always change any binding to suit their workflow. No shortcuts are locked; customization is a first-class feature.

---

## 5. Out of scope / open questions

### Auth and monetization

- Auth strategy is not yet defined. Requirements may include: taking payments, enforcing a paywall for the Electron app, or identifying users for licensing.
- Options to consider later: license keys, OAuth plus a minimal backend, or deferring auth and shipping local-only first.

### Persistence

- Local storage only for the current scope. File format, schema versioning, and upgrade path are to be defined in future work.

---

## 6. Proposed future work and repo strategy

### Repo strategy

- **Public repo (this repo):** Used for visibility and community. Can hold documentation (this file), high-level specs, and optionally a CLI, demos, or a thin public SDK. No proprietary engine implementation.
- **Private repo (engine):** Holds the OpenCascade integration, Flow execution, core data structures, and any IP that should stay closed. The public repo would depend on the engine only via a clear boundary—e.g. consumed as a binary or package, or specified by an API contract only. This keeps the product visible while protecting the core implementation.

### Future work (prioritized)

1. **Engine API and file format** — Define the public engine API and the persistence format so web and Electron can share the same semantics.
2. **Minimal Shape and Properties** — Implement Shapes with Properties; start with Attributes only, then add one Flow type (e.g. “panelize and extrude” on a face).
3. **FlowChart MVP** — Implement the FlowChart as a hierarchy view first; then show Flows on nodes and keep selection in sync with the viewport and Properties panel.
4. **Web proof-of-concept** — Validate the web path (e.g. OpenCascade via WASM or a simplified kernel) and identify performance limits.
5. **Electron shell** — Build the Electron app and integrate the native (or same WASM) engine; align feature set and UX with the web app where possible.
6. **Auth and paywall** — When needed, introduce auth and a paywall for the Electron app (or other monetization) without breaking local-first for core modeling.
