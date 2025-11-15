# URDF Loader Analysis

## Repository Overview

The repository contains two independent URDF loaders:

- A JavaScript implementation that targets Three.js scenes and lives under `javascript/src`. It builds articulated robots by parsing URDF XML, instantiating Three.js objects, and wiring up joint hierarchies.
- A Unity implementation in `unity/Assets/URDFLoader` that constructs equivalent robots as Unity `GameObject` hierarchies while handling coordinate frame conversions between ROS and Unity.

Sample URDF packages for testing live in the `urdf/` directory, and unit tests exist for the JavaScript loader under `javascript/test`.

## JavaScript Loader Architecture

### Construction and configuration

`URDFLoader` exposes configuration points through its constructor. Instances keep a reference to the active loading manager, expose a mesh loading callback, and expose flags that govern whether visual or collision elements are parsed. The loader also tracks package resolution hints, a working directory, and fetch options that are forwarded to every URDF download.【F:javascript/src/URDFLoader.js†L59-L68】

### File loading pipeline

`loadAsync` wraps the callback-based `load` helper in a promise. `load` resolves the URDF URL through Three.js’ loading manager, starts the request bookkeeping, fetches the XML text with `fetch`, and hands the resulting document to `parse`. Errors are reported to both the caller and the loading manager’s bookkeeping hooks.【F:javascript/src/URDFLoader.js†L71-L136】

### Mesh path resolution

`parse` closes over a `resolvePath` helper that normalizes `<mesh filename="…">` URIs. Non-package paths are resolved relative to `workingPath`. `package://` URIs are resolved against either a single package path, a callback that returns a package root, or a map of package names to directories. Missing mappings log a descriptive error and result in a `null` path so downstream loaders can skip the mesh.【F:javascript/src/URDFLoader.js†L138-L196】

### Robot parsing

URDF content (string, `Document`, or `Element`) is normalized and the `<robot>` root is passed to `processRobot`. That routine builds shared maps of materials, links, joints, visuals, colliders, and frames so downstream consumers can access any URDF element by name.【F:javascript/src/URDFLoader.js†L198-L310】

- Materials are processed once and cached per URDF material name. Colors, transparency, and textures defined on `<material>` nodes are applied to `MeshPhongMaterial` instances. Textures respect the loader’s package resolution and adopt sRGB color space when present.【F:javascript/src/URDFLoader.js†L235-L492】
- Links are converted into `URDFLink` objects. Depending on the configuration flags, their `<visual>` and `<collision>` child nodes are parsed into `URDFVisual` or `URDFCollider` groups, registered by name, and attached under the link.【F:javascript/src/URDFLoader.js†L392-L448】
- Joints are instantiated as `URDFJoint` or `URDFMimicJoint` depending on whether a `<mimic>` tag is present. Each joint is positioned relative to its parent link, inherits axis information, and enforces any `<limit>` bounds declared in the URDF.【F:javascript/src/URDFLoader.js†L314-L389】
- Mimic joints are wired after all joints are created. The loader also walks each joint’s mimic tree to detect cycles and throws an explicit error if an infinite loop is discovered.【F:javascript/src/URDFLoader.js†L267-L301】
- A combined `frames` dictionary is exposed on the resulting `URDFRobot` so any joint, link, visual, or collider can be retrieved by URDF name.【F:javascript/src/URDFLoader.js†L303-L308】

### Visual and collision geometry

`processLinkElement` handles both `<visual>` and `<collision>` nodes. It creates a `URDFVisual` or `URDFCollider`, attaches the resolved material (either a shared named material or an inline definition), and populates geometry. Mesh references are loaded asynchronously through the `loadMeshCb` callback, primitives (`<box>`, `<sphere>`, `<cylinder>`) are created procedurally, and `<origin>` offsets are applied using URDF’s roll-pitch-yaw order. Mesh scale attributes are forwarded to the group prior to loading.【F:javascript/src/URDFLoader.js†L495-L625】

### Default mesh loader

If consumers do not provide a `loadMeshCb`, the loader can fetch STL and Collada meshes out of the box. STL files are converted into `MeshPhongMaterial` meshes, and Collada files provide entire scenes. Unsupported extensions simply emit a warning so callers can install custom handlers.【F:javascript/src/URDFLoader.js†L633-L655】

### Coordinate conventions

The loader assumes ROS and Three.js share a right-handed coordinate system, so URDF data can be instantiated without frame remapping. Consumers remain responsible for any global transforms needed to align the model with their application’s axes.【F:javascript/src/URDFLoader.js†L6-L26】

## JavaScript URDF Data Model

The loader relies on extended Three.js classes defined in `URDFClasses.js` to represent URDF entities.

- `URDFBase` preserves the originating XML node and URDF name when Three.js objects are cloned.【F:javascript/src/URDFClasses.js†L11-L32】
- `URDFLink`, `URDFVisual`, and `URDFCollider` are lightweight markers that extend `URDFBase` and add identification flags so traversal code can recognize them.【F:javascript/src/URDFClasses.js†L34-L68】
- `URDFJoint` manages joint configuration. Setting `jointType` provisions the expected number of degrees of freedom and ensures Three.js matrices refresh. Joint state updates lazily cache the original transform, update mimic joints first, respect position or rotation limits, and support all URDF joint families (fixed, continuous, revolute, prismatic, floating, planar). Floating and planar joints compose new transforms using intermediate `Matrix4` helpers so multi-axis motion is handled consistently.【F:javascript/src/URDFClasses.js†L70-L314】
- `URDFMimicJoint` applies `multiplier` and `offset` adjustments when syncing with its source joint and ensures those properties survive cloning.【F:javascript/src/URDFClasses.js†L318-L348】
- `URDFRobot` tracks the robot-wide metadata, rebuilds link/joint maps after cloning, offers a `getFrame` helper, and exposes `setJointValue(s)` utilities for driving joints by name. When cloning, it rehydrates mimic joint references so the copy stays functional.【F:javascript/src/URDFClasses.js†L352-L469】

## JavaScript Test Coverage Insights

The test suite exercises the loader’s configuration surface:

- Disabling `parseVisual` and `parseCollision` removes those nodes from the generated graph, while enabling both yields the expected counts for a known NASA R2 URDF. Tests stub `loadMeshCb` to avoid actual asset downloads.【F:javascript/test/URDFLoader.test.js†L107-L175】
- Custom mesh callbacks and `workingPath` overrides are verified to ensure resolved mesh URLs respect caller configuration.【F:javascript/test/URDFLoader.test.js†L177-L233】
- Package resolution supports object and functional lookup tables, guaranteeing that `package://` URIs can be remapped dynamically.【F:javascript/test/URDFLoader.test.js†L236-L313】
- Cloning maintains structural and joint fidelity, including mimic joint networks, by recursively comparing every node in the original and cloned robots.【F:javascript/test/URDFLoader.test.js†L317-L375】

## Unity Loader Highlights

The Unity loader mirrors the JavaScript pipeline while adapting data into Unity’s left-handed coordinate system.

- Public `Load` and `Parse` overloads accept either a single package directory or a package map, and the optional `Options` struct allows callers to supply a mesh loader callback, override the working directory, or target an existing `URDFRobot` component.【F:unity/Assets/URDFLoader/URDFLoader.cs†L32-L90】
- Materials are parsed up front and cached by name just like the JavaScript version, applying colors from `<material>` definitions. Links create `GameObject` instances, attach renderer lists, and process all declared `<visual>` elements.【F:unity/Assets/URDFLoader/URDFLoader.cs†L95-L200】
- Joint nodes become intermediary `GameObject` transforms parented between their source and child links. The loader records URDF joint metadata (type, axis, limits) and positions the joint using converted URDF origin transforms. Axis vectors and motion limits are translated into Unity space so later runtime scripts can drive the joints correctly.【F:unity/Assets/URDFLoader/URDFLoader.cs†L382-L470】
- When the hierarchy is built, the root link is either attached to a supplied `URDFRobot` component or a new component is created. The robot gains dictionaries of links and joints so higher-level systems can query by URDF name.【F:unity/Assets/URDFLoader/URDFLoader.cs†L471-L500】
- Default mesh loading relies on `StlLoader`, but consumers can override `options.loadMeshCb` for other formats. The helper wraps raw meshes into Unity primitives before invoking the callback.【F:unity/Assets/URDFLoader/URDFLoader.cs†L510-L540】
- `ResolveMeshPath` mirrors the JavaScript package resolution logic, supporting single-package and multi-package setups with clear error logging on missing packages.【F:unity/Assets/URDFLoader/URDFLoader.cs†L542-L581】
- `URDFToUnityPos`, `URDFToUnityScale`, and `URDFToUnityRot` perform the ROS→Unity coordinate conversion, handling the left-handed to right-handed axis swap and converting radians to degrees for Unity’s Euler angles.【F:unity/Assets/URDFLoader/URDFLoader.cs†L689-L746】

## Practical Usage Tips

- Always set `loader.packages` (JavaScript) or pass package hints to `Load/Parse` (Unity) when URDF meshes live under `package://` URIs; otherwise `resolvePath` will return `null` and meshes will be skipped.【F:javascript/src/URDFLoader.js†L150-L196】【F:unity/Assets/URDFLoader/URDFLoader.cs†L542-L581】
- Provide a custom `loadMeshCb` when supporting additional mesh formats, applying per-asset materials or physics, or integrating with asset pipelines. Both implementations defer to this callback for all mesh instantiation.【F:javascript/src/URDFLoader.js†L61-L63】【F:javascript/src/URDFLoader.js†L548-L571】【F:unity/Assets/URDFLoader/URDFLoader.cs†L36-L88】【F:unity/Assets/URDFLoader/URDFLoader.cs†L510-L540】
- Use the `frames` map on `URDFRobot` (JavaScript) or the `links/joints` dictionaries on the Unity robot to look up specific nodes by URDF name without traversing the whole tree.【F:javascript/src/URDFLoader.js†L303-L308】【F:unity/Assets/URDFLoader/URDFLoader.cs†L471-L499】
- Drive motion through `URDFRobot.setJointValue(s)` (JavaScript) so all joint types, mimic chains, and limit enforcement are handled automatically. Mimic joints propagate updates through their master joint chain before applying their own multiplier/offset adjustments.【F:javascript/src/URDFClasses.js†L157-L314】【F:javascript/src/URDFClasses.js†L318-L348】
