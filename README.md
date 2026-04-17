# spine-csharp

A self-contained 2D skeletal animation runtime for .NET, implementing the [Spine](http://esotericsoftware.com/) animation format. This library loads, manages, and plays back skeletal animations created in the Spine editor, providing the core runtime logic needed to integrate Spine animations into any C#/.NET application or game engine.

> Based on [Spine Runtimes](https://github.com/EsotericSoftware/spine-runtimes)

## Target Frameworks

- .NET 10.0 (primary)
- .NET 8.0

No external NuGet dependencies — the library is entirely self-contained.

---

## Architecture

The runtime follows a **setup pose / current pose** pattern that cleanly separates static data from mutable runtime state:

- **Setup Pose** — Stateless skeleton data loaded from JSON or binary files (`SkeletonData`, `BoneData`, `SlotData`, etc.). This data is immutable and can be shared across multiple skeleton instances.
- **Current Pose** — A live `Skeleton` instance whose bones, slots, and constraints are updated each frame by an `AnimationState`.

### Loading Pipeline

```
Atlas (.atlas file)
  └─▸ AtlasAttachmentLoader
        └─▸ SkeletonJson / SkeletonBinary  (reads .json or .skel)
              └─▸ SkeletonData             (shared setup pose)
                    ├─▸ Skeleton           (per-instance runtime pose)
                    └─▸ AnimationStateData  (mix durations)
                          └─▸ AnimationState (playback & mixing)
```

1. Parse the `.atlas` file into an `Atlas` containing texture page and region metadata.
2. Create an `AtlasAttachmentLoader` (or a custom `AttachmentLoader`) backed by that atlas.
3. Use `SkeletonJson` or `SkeletonBinary` to load a `SkeletonData` from file.
4. Instantiate a `Skeleton` from the shared `SkeletonData`.
5. Create an `AnimationStateData` to define crossfade durations between animations.
6. Create an `AnimationState` and set/queue animations.
7. Each frame: advance the animation state, apply it to the skeleton, then update the skeleton's world transforms.

---

## Core Subsystems

### Animation System

| Class | Responsibility |
|---|---|
| `Animation` | A named collection of timelines that animate skeleton properties over time. |
| `AnimationState` | Drives animation playback across multiple tracks with mixing, layering, queuing, and event callbacks (`Start`, `Interrupt`, `End`, `Dispose`, `Complete`). |
| `AnimationStateData` | Stores per-animation-pair mix (crossfade) durations and a default mix time. |

Key capabilities:

- **Multi-track playback** — independent animations on separate tracks (e.g. walk on track 0, shoot on track 1).
- **Animation mixing** — smooth crossfading between animations with configurable durations.
- **Mix modes & blend directions** — fine control over how animations combine.
- **Event callbacks** — delegate-based hooks for reacting to animation lifecycle events.

### Skeleton System

| Class | Responsibility |
|---|---|
| `SkeletonData` | Immutable setup pose: bone hierarchy, slots, skins, animations, constraints, events, reference scale, FPS, and asset paths. |
| `Skeleton` | Mutable runtime instance: current bone transforms, slot states, draw order, active skin, color tint, and physics state. Maintains an update cache for correct evaluation order. |
| `SkeletonLoader` | Abstract base for format-specific loaders. |
| `SkeletonJson` | Deserializes skeleton data from Spine's JSON export format. |
| `SkeletonBinary` | Deserializes skeleton data from Spine's compact binary format (faster loading, smaller files). |

### Bone & Transform System

| Class | Responsibility |
|---|---|
| `Bone` | Runtime bone with local transform (x, y, rotation, scaleX, scaleY, shearX, shearY), applied transform, and computed world transform matrix (a, b, c, d, worldX, worldY). Implements `IUpdatable`. |
| `BoneData` | Setup pose defaults and hierarchy information for a bone. |

Bones support multiple inheritance modes via the `Inherit` enum: `Normal`, `OnlyTranslation`, `NoRotationOrReflection`, and `NoScale`.

### Slot & Rendering System

| Class | Responsibility |
|---|---|
| `Slot` | Runtime slot: references a parent bone, holds RGBA color (with optional secondary dark color for two-color tinting), the current attachment, deform vertices, sequence index, and blend mode. |
| `SlotData` | Setup pose slot defaults. |
| `BlendMode` | Enum: `Normal`, `Additive`, `Multiply`, `Screen`. |

Draw order can differ from hierarchy order, allowing flexible rendering.

### Constraint System

Four constraint types provide advanced rigging capabilities:

#### IK Constraints
`IkConstraint` / `IkConstraintData` — Inverse kinematics for 1–2 bone chains. Supports bend direction, softness, compression, and stretch.

#### Transform Constraints
`TransformConstraint` / `TransformConstraintData` — Matches constrained bones to a target bone's world transform with independent mix percentages for rotation, translation, scale, and shear. Supports relative and local modes.

#### Path Constraints
`PathConstraint` / `PathConstraintData` — Constrains bones to follow a `PathAttachment` curve. Configurable position modes (Fixed, Percent), spacing modes (Length, Fixed, Percent, Proportional), and rotation modes (Tangent, Chain, ChainScale).

#### Physics Constraints
`PhysicsConstraint` / `PhysicsConstraintData` — Applies realistic secondary motion to bones with inertia, damping, mass, gravity, and wind. Useful for hair, capes, tails, and other soft-body effects.

All constraints implement the `IUpdatable` interface and are evaluated in dependency order via the skeleton's update cache.

### Attachment System

Attachments are the visual and structural elements attached to slots.

| Class | Description |
|---|---|
| `RegionAttachment` | A textured quad (4 vertices) with color, offset, and UV arrays. |
| `MeshAttachment` | A deformable mesh with vertices, triangles, UVs, hull, and optional parent mesh linking for shared vertex data. Supports bone weights. |
| `BoundingBoxAttachment` | A polygon used for hit testing and bounds computation. |
| `PathAttachment` | A Bézier path curve used by path constraints. Supports open/closed paths. |
| `PointAttachment` | A single point with rotation, useful for particle/effect spawn locations. |
| `ClippingAttachment` | A polygon that clips rendering of subsequent slots. |

Base classes and interfaces:

- `Attachment` — abstract base with name and copy support.
- `VertexAttachment` — abstract base for attachments with bone-weighted, deformable vertices.
- `IHasTextureRegion` — interface for attachments backed by a texture region.
- `AttachmentLoader` — interface for pluggable attachment creation.
- `AtlasAttachmentLoader` — default loader that resolves attachments from an `Atlas`.
- `Sequence` — drives frame-by-frame texture region switching for animated attachments.

### Atlas & Texture System

| Class | Responsibility |
|---|---|
| `Atlas` | Parses `.atlas` files and manages texture pages and regions. Handles texture padding, rotation, and filter settings. |
| `AtlasRegion` | A named rectangular region within an atlas texture. |
| `TextureRegion` | Base class providing width, height, and UV coordinates (u, v, u2, v2). |

### Skin System

`Skin` is a named collection of attachments keyed by `(slotIndex, attachmentName)`. Skins can also reference specific bones and constraints. Multiple skins can be combined at runtime (e.g. mixing equipment pieces), and mesh attachments are deep-copied to avoid shared state.

### Event System

| Class | Responsibility |
|---|---|
| `EventData` | Defines an event template with default int, float, string, volume, and balance values. |
| `Event` | A runtime event instance fired at a specific time during animation playback. |

### Geometry & Bounds

| Class | Responsibility |
|---|---|
| `SkeletonBounds` | Collects bounding box attachments from visible slots, computes an axis-aligned bounding box (AABB), and provides polygon hit testing. Uses an object pool for polygons. |
| `SkeletonClipping` | Clips mesh rendering against `ClippingAttachment` polygons. Produces clipped vertex, UV, and triangle arrays. |
| `Triangulator` | Triangulates concave polygons via ear-clipping and decomposes them into convex sub-polygons. Uses internal pools for performance. |

### Utilities

| Class | Responsibility |
|---|---|
| `MathUtils` | Trigonometry (sin/cos/atan2 with optional lookup tables), degree/radian conversion, clamping, and interpolation. |
| `ExposedList<T>` | A `List<T>`-like collection that exposes its backing `Items` array publicly for direct, bounds-check-free iteration — critical for performance in tight update loops. |
| `Json` | Lightweight JSON deserializer for skeleton data parsing. |

---

## Design Patterns

- **Data / Instance separation** — every runtime class (`Bone`, `Slot`, `IkConstraint`, …) has a corresponding `*Data` class holding the immutable setup pose. This allows sharing one `SkeletonData` across many `Skeleton` instances.
- **IUpdatable interface** — bones and constraints share a polymorphic `Update()` contract, evaluated in topologically-sorted order.
- **Strategy pattern** — `AttachmentLoader` lets consumers swap in custom attachment creation logic.
- **Object pooling** — `SkeletonBounds`, `SkeletonClipping`, and `Triangulator` pool intermediate arrays to minimise GC pressure.
- **ExposedList** — avoids virtual dispatch and bounds checking on hot paths by exposing the raw array.

---

## Project Structure

```
spine-csharp/
├── spine-csharp.csproj          # Project file (net10.0 + net8.0)
└── src/
    ├── Animation.cs             # Animation & timeline definitions
    ├── AnimationState.cs        # Multi-track playback & mixing
    ├── AnimationStateData.cs    # Mix duration configuration
    ├── Atlas.cs                 # Texture atlas loading
    ├── BlendMode.cs             # Blend mode enum
    ├── Bone.cs                  # Runtime bone
    ├── BoneData.cs              # Setup pose bone data
    ├── ConstraintData.cs        # Base constraint data
    ├── Event.cs                 # Runtime event
    ├── EventData.cs             # Event template
    ├── ExposedList.cs           # High-performance list
    ├── IkConstraint[Data].cs    # Inverse kinematics
    ├── IUpdatable.cs            # Update interface
    ├── Json.cs                  # JSON parser
    ├── MathUtils.cs             # Math helpers
    ├── PathConstraint[Data].cs  # Path following
    ├── PhysicsConstraint[Data].cs # Physics simulation
    ├── Skeleton.cs              # Runtime skeleton
    ├── SkeletonBinary.cs        # Binary format loader
    ├── SkeletonBounds.cs        # Bounding box & hit testing
    ├── SkeletonClipping.cs      # Mesh clipping
    ├── SkeletonData.cs          # Setup pose data
    ├── SkeletonJson.cs          # JSON format loader
    ├── SkeletonLoader.cs        # Abstract loader base
    ├── Skin.cs                  # Skin (attachment collection)
    ├── Slot.cs                  # Runtime slot
    ├── SlotData.cs              # Setup pose slot data
    ├── TextureRegion.cs         # UV region base class
    ├── TransformConstraint[Data].cs # Transform matching
    ├── Triangulator.cs          # Polygon triangulation
    └── Attachments/
        ├── AtlasAttachmentLoader.cs
        ├── Attachment.cs
        ├── AttachmentLoader.cs
        ├── AttachmentType.cs
        ├── BoundingBoxAttachment.cs
        ├── ClippingAttachment.cs
        ├── IHasTextureRegion.cs
        ├── MeshAttachment.cs
        ├── PathAttachment.cs
        ├── PointAttachment.cs
        ├── RegionAttachment.cs
        ├── Sequence.cs
        └── VertexAttachment.cs
```

---

## License

See the [Spine Runtimes License](https://github.com/EsotericSoftware/spine-runtimes) for terms of use.
