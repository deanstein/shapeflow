# ShapeFlow

**ShapeFlow** is an easy-to-use 3D modeler for architectural and industrial designers. It centers on **3D sketching**—drawing on surfaces to split and refine geometry, then extruding or applying flows—combined with **Flows** (parametric rules attached to geometry) and a node-based **FlowChart** that mirrors the model hierarchy. Everything runs **local-first**: no cloud dependency for core modeling, so your data and compute stay on your machine.

Planned as both a **web app** (broader reach, some performance limits) and an **Electron app** (more power and features for heavy users). The stack is **Svelte + TypeScript** for the UI and **Rust** for the engine, with **OpenCascade** as the geometry kernel. All tools and UI call a single **application API**, and the same data model is used in memory and on disk (`.sf` file format).

For full vision, concepts, and planned features, see [docs/architecture.md](docs/architecture.md).
