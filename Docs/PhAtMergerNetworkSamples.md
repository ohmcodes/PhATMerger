# PhAtMerger Multiplayer Sample Usage (Case-by-Case)

This document shows complete practical patterns for sending mesh-selection arrays over network and merging on clients.

It is aligned with:
- `UPhAtMerger::BuildMergeReplicationPayload(...)`
- `UPhAtMerger::ApplyMergeReplicationPayload(...)`
- `FPhAtMergeOptions`
- `EPhAtMergeReplicationMode`

---

## Should you use soft references?

Short answer: **yes**, for replicated customization descriptors.

Recommended:
- Replicate lightweight IDs or soft references (`TSoftObjectPtr`) from server.
- Resolve/load on client, then call merge locally.

Why:
- lower memory pressure before assets are needed
- safer for large customization catalogs
- avoids hard reference chains that force cooking/loading everything

Important:
- `UPhAtMerger` functions currently accept hard pointers (`USkeletalMesh*`).
- So the flow is: **replicate soft refs -> async load -> build hard pointer array -> merge**.

---

## Core network rule

Use **server-authoritative selection**:
1. Client detects overlap and asks server to equip/apply option.
2. Server validates request.
3. Server updates replicated payload/state.
4. Clients receive update and apply merge locally (or apply merged refs depending on mode).

---

## Shared Data Model (recommended)

Create a replicated struct for cosmetics/customization, for example in your Character or a dedicated component.

Suggested fields:
- `TArray<TSoftObjectPtr<USkeletalMesh>> MeshSoftRefs`
- `TArray<int32> PreferredCollisionSourceAssetIndices`
- `EPhAtMergeReplicationMode ReplicationMode`
- any gameplay metadata (slot/type/version)

Then on client:
- load all soft refs
- convert to `TArray<USkeletalMesh*>`
- call `MergeSkeletalMeshesAndPhysicsAssetsEx(...)`

---

## Case 1: Overlap pickup -> Replicate source meshes -> Merge locally on every client (recommended default)

## When to use
- Many customization combinations
- Want lower network payload
- Accept client CPU cost for merge

## Flow
1. Player overlaps an object (hat/hair/outfit actor) that contains mesh soft refs.
2. Client calls `Server_RequestApplyCosmetic(ItemId or SoftRefList)` RPC.
3. Server validates ownership/range/game rules.
4. Server sets replicated cosmetic state (soft refs + options).
5. `OnRep_CosmeticState` runs on each client.
6. Each client async-loads meshes from soft refs.
7. Each client merges locally via `MergeSkeletalMeshesAndPhysicsAssetsEx`.

## Options setup suggestion
- `ReplicationMode = ReplicateSourceMeshesAndMergeLocally`
- `bMergePhysicsAssets = true`
- `bAssignToTargetComponent = true`
- `bDisableNonConstrainedCollision = true`
- `bDetectOverlappingBodies = true`
- `PreferredCollisionSourceAssetIndices = [index priority you want]`

## Pros
- Small replicated state
- Very scalable for many combinations

## Cons
- Merge cost on each client

---

## Case 2: Server pre-merges -> Replicate merged references -> Clients only apply

## When to use
- Fewer combinations
- Want lower client CPU spikes
- Server can own merge cost and output references

## Flow
1. Server receives equip request.
2. Server builds/gets merged mesh + merged physics asset.
3. Server builds payload with `ReplicationMode = ReplicateMergedAssetsReferences`.
4. Payload replicates.
5. Clients call `ApplyMergeReplicationPayload(...)` and directly assign.

## Pros
- Low client CPU at apply time

## Cons
- Larger replicated references
- Asset availability/cook discipline needed

---

## Case 3: Hybrid with cache (best practical production pattern)

## When to use
- Frequent repeated combinations
- Need low hitching + low network cost

## Flow
1. Replicate source mesh soft refs (Case 1 style).
2. Client checks local merge cache key (e.g., sorted asset paths + options hash).
3. If cache hit: apply immediately.
4. If miss: async load, merge, store in cache, apply.

## Pros
- Good network profile
- Good runtime smoothness after warm-up

---

## Case 4: Overlap actor contains only a DataAsset ID

## When to use
- Strict content pipeline
- Designers author everything in assets

## Flow
1. Overlap actor stores one `TSoftObjectPtr<UYourCosmeticDataAsset>`.
2. Client asks server with just asset ID/path.
3. Server validates and replicates ID/path.
4. Client loads DataAsset, resolves mesh soft refs, merges.

## Pros
- Minimal network payload
- Centralized content definition

---

## Complete C++ Sample (Case 1: soft refs replicated, merge on clients)

This sample uses:
- soft references in replicated state
- server-authoritative overlap request
- client `OnRep` load + merge

> Note: `UPhAtMerger` merge functions require `USkeletalMesh*`, so we async-load soft refs first.

```cpp
// Cosmetic state replicated to all clients
USTRUCT(BlueprintType)
struct FRepCosmeticState
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<TSoftObjectPtr<USkeletalMesh>> MeshSoftRefs;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<int32> PreferredCollisionSourceAssetIndices;
};

UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    UPROPERTY(ReplicatedUsing=OnRep_CosmeticState)
    FRepCosmeticState RepCosmeticState;

    UFUNCTION(Server, Reliable)
    void Server_RequestApplyCosmetic(const FRepCosmeticState& NewState);

    UFUNCTION()
    void OnRep_CosmeticState();

private:
    void LoadAndApplyCosmetics(const FRepCosmeticState& State);
};

void AMyCharacter::Server_RequestApplyCosmetic_Implementation(const FRepCosmeticState& NewState)
{
    // 1) Validate request (distance/ownership/whitelist/etc.)
    // 2) Accept + replicate authoritative state
    RepCosmeticState = NewState;

    // Ensure server also applies immediately (OnRep won't run on authority)
    LoadAndApplyCosmetics(RepCosmeticState);

    ForceNetUpdate();
}

void AMyCharacter::OnRep_CosmeticState()
{
    LoadAndApplyCosmetics(RepCosmeticState);
}

void AMyCharacter::LoadAndApplyCosmetics(const FRepCosmeticState& State)
{
    TArray<FSoftObjectPath> Paths;
    for (const TSoftObjectPtr<USkeletalMesh>& SoftMesh : State.MeshSoftRefs)
    {
        if (!SoftMesh.IsNull())
        {
            Paths.AddUnique(SoftMesh.ToSoftObjectPath());
        }
    }

    if (Paths.Num() == 0)
    {
        return;
    }

    FStreamableManager& Streamable = UAssetManager::GetStreamableManager();
    Streamable.RequestAsyncLoad(Paths, FStreamableDelegate::CreateWeakLambda(this, [this, State]()
    {
        TArray<USkeletalMesh*> LoadedMeshes;
        for (const TSoftObjectPtr<USkeletalMesh>& SoftMesh : State.MeshSoftRefs)
        {
            if (USkeletalMesh* Mesh = SoftMesh.Get())
            {
                LoadedMeshes.Add(Mesh);
            }
        }

        if (LoadedMeshes.Num() < 2)
        {
            return;
        }

        FPhAtMergeOptions Options;
        Options.ReplicationMode = EPhAtMergeReplicationMode::ReplicateSourceMeshesAndMergeLocally;
        Options.bMergePhysicsAssets = true;
        Options.bAssignToTargetComponent = true;
        Options.bDisableNonConstrainedCollision = true;
        Options.bDetectOverlappingBodies = true;
        Options.PreferredCollisionSourceAssetIndices = State.PreferredCollisionSourceAssetIndices;

        USkeletalMesh* OutMesh = nullptr;
        UPhysicsAsset* OutPhat = nullptr;

        UPhAtMerger::MergeSkeletalMeshesAndPhysicsAssetsEx(
            LoadedMeshes,
            GetMesh(),
            Options,
            OutMesh,
            OutPhat);
    }));
}
```

### Overlap actor side (C++)

```cpp
UCLASS()
class ACosmeticPickupActor : public AActor
{
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TArray<TSoftObjectPtr<USkeletalMesh>> CosmeticMeshes;

    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TArray<int32> PreferredCollisionSourceAssetIndices;
};
```

When character overlaps pickup:
1. Read pickup `CosmeticMeshes` + `PreferredCollisionSourceAssetIndices`.
2. Build `FRepCosmeticState`.
3. Call `Server_RequestApplyCosmetic(State)`.

---

## Complete Blueprint Sample (Case 1: soft refs replicated, merge on clients)

### Character variables
- `RepCosmeticMeshesSoft` (RepNotify): `Array<Soft Skeletal Mesh>`
- `RepCollisionPriority` (RepNotify): `Array<int>`
- `CharacterMeshComp`: your main `SkeletalMeshComponent`

### Pickup actor variables
- `CosmeticMeshesSoft`: `Array<Soft Skeletal Mesh>`
- `CollisionPriority`: `Array<int>`

### RPC setup
- `Server_RequestApplyCosmetic(MeshesSoft, CollisionPriority)`
  - Run On Server, Reliable

### On overlap (client or server-controlled input)
1. Cast overlap actor to cosmetic pickup.
2. Read `CosmeticMeshesSoft` + `CollisionPriority`.
3. Call `Server_RequestApplyCosmetic` with those arrays.

### Server_RequestApplyCosmetic graph
1. Validate request (distance/allowed pickup/etc.).
2. Set replicated vars:
   - `RepCosmeticMeshesSoft = MeshesSoft`
   - `RepCollisionPriority = CollisionPriority`
3. Call local `ApplyReplicatedCosmetics` on server too (same logic as OnRep).

### OnRep handler graph (`OnRep_RepCosmeticMeshesSoft` or combined function)
1. Async Load Asset List (all `RepCosmeticMeshesSoft`).
2. On Completed:
   - Convert loaded soft refs to hard `USkeletalMesh` array.
   - Make `FPhAtMergeOptions`:
     - `bMergePhysicsAssets = true`
     - `bAssignToTargetComponent = true`
     - `bDisableNonConstrainedCollision = true`
     - `bDetectOverlappingBodies = true`
     - `PreferredCollisionSourceAssetIndices = RepCollisionPriority`
     - `ReplicationMode = ReplicateSourceMeshesAndMergeLocally`
   - Call `MergeSkeletalMeshesAndPhysicsAssetsEx`:
     - `SourceMeshes = LoadedMeshes`
     - `TargetComponent = CharacterMeshComp`
     - `Options = configured options`

### Practical BP note
If you need a single RepNotify trigger, replicate one struct (or replicate one "version" int and read both arrays in the OnRep function).

---

## Example Blueprint-Level Sequence (Case 1)

## Character variables (replicated)
- `ReplicatedCosmeticMeshesSoft : Array(Soft SkeletalMesh)`
- `ReplicatedCollisionPriority : Array(int)`
- `ReplicatedMergeMode : EPhAtMergeReplicationMode`

## RPCs
- `Server_RequestApplyCosmetic(SoftMeshes, CollisionPriority)` (RunOnServer, Reliable)

## Server RPC body
1. Validate request.
2. Set replicated vars.
3. (Optional) force net update.

## OnRep handler on clients
1. Async load all `ReplicatedCosmeticMeshesSoft`.
2. Build hard array `LoadedMeshes`.
3. Create `FPhAtMergeOptions`:
   - set booleans
   - set `ReplicationMode = ReplicatedMergeMode`
   - set `PreferredCollisionSourceAssetIndices = ReplicatedCollisionPriority`
4. Call `MergeSkeletalMeshesAndPhysicsAssetsEx(LoadedMeshes, CharacterMeshComp, Options, OutMesh, OutPhat)`.

---

## Soft Reference Loading Notes

- Always guard against partial load failure.
- If one mesh fails load, either:
  - abort merge and keep previous appearance, or
  - continue with subset if your design allows.
- Preload likely items (nearby pickup previews) to hide first-use hitch.

---

## Validation/Security Checklist (server side)

- Validate overlap distance and interaction rules.
- Validate requested assets are in allowed whitelist/table.
- Validate mesh count and slot compatibility.
- Reject unknown/unapproved paths from clients.

---

## Collision Priority Tips for your use case (hair vs face/body)

If multiple source assets define `head`/`upper body` colliders:
- Put the preferred source asset indices first in `PreferredCollisionSourceAssetIndices`.
- Example: `[HairPhysicsIndex, BodyPhysicsIndex, FacePhysicsIndex]` or reverse depending on desired winner.
- Test with logs enabled to verify replacement decisions.

---

## Debug Checklist

If result looks wrong:
1. Confirm soft refs resolved to valid hard mesh pointers before merge.
2. Confirm source array order is what you expect.
3. Confirm `PreferredCollisionSourceAssetIndices` values match that source order.
4. Check logs for:
   - `[ADDED]`
   - `[REPLACED]`
   - `[SKIPPED]`
   - `[CONSTRAINT COPIED]`
5. Ensure only one authority path applies cosmetics (avoid duplicate apply calls).

---

## Recommended default for your project

Given your scenario (modular hair pieces + duplicate head/body colliders):
- Replication strategy: `ReplicateSourceMeshesAndMergeLocally`
- Transport: soft references
- Merge call: `MergeSkeletalMeshesAndPhysicsAssetsEx`
- Use `PreferredCollisionSourceAssetIndices` to control which physics source wins duplicate bones.
