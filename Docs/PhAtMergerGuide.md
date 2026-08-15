# PhAT + Skeletal Merge Utility Guide

This document explains every merge function, struct, enum, parameter, and runtime effect in `UPhAtMerger`.

Source files:
- `Source/PhATMerger/PhATMergerBPLibrary.h`
- `Source/PhATMerger/PhATMergerBPLibrary.cpp`

Related practical sample cookbook:
- `Docs/PhAtMergerNetworkSamples.md`

---

## Purpose

`UPhAtMerger` is a Blueprint function library that can:
1. Merge multiple skeletal meshes into one mesh.
2. Merge multiple physics assets into one physics asset.
3. Assign merged mesh/physics to a target skeletal mesh component.
4. Provide multiplayer-friendly payload build/apply helpers.

The design keeps old APIs intact and adds opt-in behavior through `FPhAtMergeOptions`.

---

## Public API Reference

## 1) `MergePhysicsAssets`

Signature:
- `UPhysicsAsset* MergePhysicsAssets(TArray<UPhysicsAsset*> PhysicsAssets, USkeletalMesh* SkeletalMesh)`

Parameters:
- `PhysicsAssets`: source physics assets to merge.
- `SkeletalMesh`: merged/target skeletal mesh used for preview + overlap checks.

Return:
- New merged `UPhysicsAsset*` when at least 2 valid assets are merged.
- `nullptr` if input count is below 2.

Behavior:
- Uses default internal options:
  - `bDisableNonConstrainedCollision = true`
  - `bDetectOverlappingBodies = true`
  - no preferred source index list (first-wins on duplicate bones)

---

## 2) `MergePhysicsAssetsEx`

Signature:
- `UPhysicsAsset* MergePhysicsAssetsEx(TArray<UPhysicsAsset*> PhysicsAssets, USkeletalMesh* SkeletalMesh, const TArray<int32>& PreferredCollisionSourceAssetIndices)`

Parameters:
- `PhysicsAssets`: source physics assets.
- `SkeletalMesh`: target mesh context.
- `PreferredCollisionSourceAssetIndices`: priority order for duplicate-bone collision bodies.

Return:
- Merged `UPhysicsAsset*` or `nullptr` if invalid input count.

Behavior difference vs default:
- Same merge flow as `MergePhysicsAssets`, but duplicate-bone body selection can be overridden by preferred source index order.

---

## 3) `MergeSkeletalMeshes`

Signature:
- `bool MergeSkeletalMeshes(const TArray<USkeletalMesh*>& SourceMeshes, USkeletalMesh*& OutMergedMesh)`

Parameters:
- `SourceMeshes`: source mesh assets.
- `OutMergedMesh` (out): resulting merged mesh.

Return:
- `true` when merge succeeds.
- `false` if fewer than 2 valid meshes or merge fails.

Internal merge settings used:
- `StripTopLODS = 0`
- `bNeedsCpuAccess = false`
- `bSkeletonBefore = false`
- `Skeleton = nullptr` (mesh merge uses generated/derived setup)

Effect:
- Produces a single merged `USkeletalMesh` asset reference.
- Material slots are merged at mesh-asset level.

---

## 4) `MergeSkeletalMeshesAndPhysicsAssets`

Signature:
- `bool MergeSkeletalMeshesAndPhysicsAssets(const TArray<USkeletalMesh*>& SourceMeshes, USkeletalMeshComponent* TargetComponent, USkeletalMesh*& OutMergedMesh, UPhysicsAsset*& OutMergedPhysicsAsset)`

Parameters:
- `SourceMeshes`: source meshes.
- `TargetComponent`: component to assign merged outputs to.
- `OutMergedMesh` (out): merged mesh.
- `OutMergedPhysicsAsset` (out): merged physics asset (or passthrough single asset).

Return:
- `true` on successful mesh merge path.
- `false` when mesh merge fails.

Behavior:
- Calls `MergeSkeletalMeshesAndPhysicsAssetsEx` with default `FPhAtMergeOptions` values.

---

## 5) `MergeSkeletalMeshesAndPhysicsAssetsEx`

Signature:
- `bool MergeSkeletalMeshesAndPhysicsAssetsEx(const TArray<USkeletalMesh*>& SourceMeshes, USkeletalMeshComponent* TargetComponent, const FPhAtMergeOptions& Options, USkeletalMesh*& OutMergedMesh, UPhysicsAsset*& OutMergedPhysicsAsset)`

Parameters:
- `SourceMeshes`: source meshes.
- `TargetComponent`: optional assignment target.
- `Options`: behavior toggles and replication mode.
- `OutMergedMesh` (out): merged mesh.
- `OutMergedPhysicsAsset` (out): merged/selected physics asset.

Return:
- `true` when mesh merge succeeds and function completes.
- `false` when mesh merge fails.

Detailed effects:
- Always performs mesh merge first.
- If `Options.bMergePhysicsAssets` is `true`:
  - Collects physics assets from source meshes.
  - If collected count `>= 2`, merges physics assets internally.
  - If collected count `== 1`, forwards single physics asset directly (no merge).
  - If `0`, output physics remains null.
- If `Options.bAssignToTargetComponent` and `TargetComponent` valid:
  - Assigns merged mesh.
  - Assigns merged physics if valid.
  - Calls `RecreatePhysicsState()`.

---

## 6) `BuildMergeReplicationPayload`

Signature:
- `bool BuildMergeReplicationPayload(const TArray<USkeletalMesh*>& SourceMeshes, USkeletalMesh* MergedMesh, UPhysicsAsset* MergedPhysicsAsset, const FPhAtMergeOptions& Options, FPhAtMergeReplicationPayload& OutPayload)`

Parameters:
- `SourceMeshes`: source meshes for local merge replication mode.
- `MergedMesh`: merged mesh for reference replication mode.
- `MergedPhysicsAsset`: merged physics for reference replication mode.
- `Options`: decides which replication mode is used.
- `OutPayload` (out): payload to replicate.

Return:
- In `ReplicateSourceMeshesAndMergeLocally`: `true` if at least one valid source mesh copied.
- In `ReplicateMergedAssetsReferences`: `true` if merged mesh is valid.

Effect:
- Writes only the fields required for the selected mode.

---

## 7) `ApplyMergeReplicationPayload`

Signature:
- `bool ApplyMergeReplicationPayload(const FPhAtMergeReplicationPayload& Payload, USkeletalMeshComponent* TargetComponent, const FPhAtMergeOptions& Options, USkeletalMesh*& OutMergedMesh, UPhysicsAsset*& OutMergedPhysicsAsset)`

Parameters:
- `Payload`: replicated payload.
- `TargetComponent`: component to apply final result.
- `Options`: used if local merge mode is chosen.
- `OutMergedMesh` (out): resulting mesh.
- `OutMergedPhysicsAsset` (out): resulting physics asset.

Return:
- `false` if target component invalid.
- Local merge mode: result of `MergeSkeletalMeshesAndPhysicsAssetsEx`.
- Reference mode: `false` if payload merged mesh invalid; otherwise applies and returns `true`.

Effect by mode:
- `ReplicateSourceMeshesAndMergeLocally`: rebuilds locally from source meshes.
- `ReplicateMergedAssetsReferences`: directly assigns payload mesh/physics to component.

---

## Struct and Enum Reference

## `EPhAtMergeReplicationMode`

Values:
1. `ReplicateSourceMeshesAndMergeLocally`
   - Replicate source mesh list.
   - Clients run merge locally.
   - Lower replicated asset-reference requirements, higher client CPU.

2. `ReplicateMergedAssetsReferences`
   - Replicate final merged mesh/physics references.
   - Clients apply directly.
   - Lower client CPU, depends on replicated asset references availability.

---

## `FPhAtMergeOptions`

Fields and effects:
- `bMergePhysicsAssets` (default `true`)
  - `true`: merge/select physics asset output.
  - `false`: skip physics merge entirely.

- `bAssignToTargetComponent` (default `true`)
  - `true`: function auto-applies outputs to `TargetComponent` and rebuilds physics state.
  - `false`: outputs are returned only.

- `bDisableNonConstrainedCollision` (default `true`)
  - In physics merge: disables collision between body pairs that do not share a constraint.
  - Helps reduce internal self-collision jitter.

- `bDetectOverlappingBodies` (default `true`)
  - In physics merge: creates temporary scene/component and overlap-tests body pairs.
  - Disables collision for overlapping pairs.

- `ReplicationMode` (default `ReplicateSourceMeshesAndMergeLocally`)
  - Controls payload build/apply behavior for multiplayer helper functions.

- `PreferredCollisionSourceAssetIndices` (default empty)
  - Priority list for duplicate-bone body ownership.
  - Empty list preserves original first-wins behavior.

---

## `FPhAtMergeReplicationPayload`

Fields:
- `ReplicationMode`: payload interpretation mode.
- `SourceMeshes`: used in local merge mode.
- `MergedMesh`: used in merged-reference mode.
- `MergedPhysicsAsset`: used in merged-reference mode.

---

## Physics Merge Internal Pipeline (Exact Behavior)

When internal physics merge runs (`MergePhysicsAssetsInternal`), it executes:

1. Validate count (`>= 2`) else return `nullptr`.
2. Create a new transient `UPhysicsAsset` and set preview mesh.
3. Merge body setups:
   - First body per bone is added.
   - On duplicate bone:
     - default: skip duplicate.
     - with preferred index list: allow replacement when incoming source has higher priority.
   - Uses `DuplicateObject<USkeletalBodySetup>` to preserve authored body settings.
4. Copy constraints:
   - Duplicates full `UPhysicsConstraintTemplate` for constraints whose bodies exist.
   - Skips duplicate joint names.
5. Optional collision prune #1:
   - if enabled, disables all non-constrained body-pair collisions.
6. Optional collision prune #2:
   - if enabled, detects overlap pairs in temp preview scene and disables collisions for those pairs.
7. Logs summary and returns merged physics asset.

---

## Duplicate Bone Collision Selection (Preferred Indices)

Use when multiple source physics assets define collision on the same bone (head/body/face region, etc.).

Rules:
- Indices map to the order in `PhysicsAssets` input array.
- Smaller rank in `PreferredCollisionSourceAssetIndices` means higher priority.
- If both existing/incoming are unranked, existing owner is kept.
- If list is empty, existing owner is always kept (legacy behavior).

Example:
- `PhysicsAssets = [A0, A1, A2]`
- `PreferredCollisionSourceAssetIndices = [2, 0, 1]`
- Duplicate `head` body winner priority: `A2` > `A0` > `A1`.

Recommended use case:
- Keep only one authoritative face/head/body collider source to avoid hair colliding against duplicate nearby colliders.

---

## Blueprint Usage Patterns

## A) Mesh only
1. Prepare `SourceMeshes`.
2. Call `MergeSkeletalMeshes`.
3. If `true`, assign/use `OutMergedMesh`.

## B) Mesh + physics with defaults (non-breaking)
1. Call `MergeSkeletalMeshesAndPhysicsAssets`.
2. Component is auto-assigned when valid.

## C) Mesh + physics with full control
1. Create `FPhAtMergeOptions`.
2. Set toggles and preferred index list.
3. Call `MergeSkeletalMeshesAndPhysicsAssetsEx`.

## D) Multiplayer payload workflow
1. Server determines replication strategy in options.
2. Server builds payload with `BuildMergeReplicationPayload`.
3. Replicate payload.
4. Clients call `ApplyMergeReplicationPayload`.

---

## Benefits

### Runtime
- Reduces number of active skeletal mesh components per character.
- Can reduce per-character physics overhead by consolidating setups.

### Authoring fidelity
- Preserves authored body and constraint tuning by duplicating source PhAT objects.

### Flexibility
- Keeps legacy Blueprint nodes.
- Adds opt-in controls via `Ex` API and options struct.
- Supports two multiplayer replication strategies.

---

## Caveats / Important Notes

- `MergePhysicsAssets` requires at least 2 physics assets; otherwise returns null.
- `MergeSkeletalMeshes` requires at least 2 valid source meshes.
- `bNeedsCpuAccess` is currently false in mesh merge settings.
- Full VS solution rebuild may fail due to unrelated Engine tooling projects; build game editor target for gameplay code validation.
- If C++ build reports Live Coding lock, disable Live Coding (`Ctrl+Alt+F11`) or close editor before build.
