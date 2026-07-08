# Unturned Code Insights

Personal notes from an initial source dive into the recently opened Unturned 3.x Unity project.

## High-Level Shape

- Main runtime code lives under `Assets/Runtime/Assembly-CSharp`.
- Game-specific code is mostly under `Assets/Runtime/Assembly-CSharp/Unturned`.
- The project uses Unity, Steamworks.NET, and its own networking layers:
  - `NetMessaging`
  - `NetInvokable`
  - `NetPak`
  - transport implementations for Steam Networking, Steam Networking Sockets, System Sockets, loopback, and legacy UNet.
- Many legacy `ask*` / `tell*` methods remain as obsolete compatibility wrappers, while newer code tends to use typed `ClientStaticMethod`, `ServerStaticMethod`, `ClientInstanceMethod`, and `ServerInstanceMethod` handles.

## Server Authority

In gameplay code, "server" usually means `Provider.isServer`, not only dedicated server. A dedicated server is a more specific case checked with `Dedicator.IsDedicatedServer`.

This matters because Unturned can run as:

- Dedicated server: headless authoritative server.
- Listen/local host: one player's client is also the authoritative server.
- Client: connected to one of the above.

Steam lobby/matchmaking is more of a pre-game or connection coordination layer. Once gameplay is running, one peer owns server authority.

## Zombie Simulation

Relevant files:

- `Assets/Runtime/Assembly-CSharp/Unturned/Managers/ZombieManager.cs`
- `Assets/Runtime/Assembly-CSharp/Unturned/Managers/ZombieRegion.cs`
- `Assets/Runtime/Assembly-CSharp/Unturned/Zombies/Zombie.cs`

Zombie updates are split into two broad paths.

### AI Tick

`Zombie.tick()` handles the expensive decision-making:

- Is the target player still valid?
- Has the target died, entered water, or moved out of nav bounds?
- Is the zombie stuck?
- Should it attack a barricade, structure, vehicle, object, or player?
- Should a special zombie use a special ability?
- What target point should pathfinding move toward?

The actual movement is delegated to a pathfinding movement component through `seeker.Move(delta)`.

Zombies are not all fully simulated all the time. `Zombie.isHunting` adds/removes the zombie from `ZombieManager.tickingZombies`. This means idle zombies are comparatively cheap, and only active/hunting/pulled zombies spend CPU on AI tick logic.

Dedicated servers further spread work over frames:

- `ZombieManager.Update()` processes a slice of `tickingZombies`.
- The dedicated-server slice is capped around 50 zombies per frame.
- Non-dedicated modes can tick the whole list.

There is also a cap for wandering behavior:

- `ZombieManager.canSpareWanderer => wanderingCount < 8 && tickingZombies.Count < 50`

So the game intentionally limits background wandering and avoids turning the whole population into active agents.

### State / Network / Animation Update

`Zombie.OnUpdate()` used to be Unity's `Update()` message, but a 2026 comment says maps with 2500+ zombies made that expensive. The current approach is that `ZombieManager` manually calls `OnUpdate()` only for zombies in regions with players.

On the server, `OnUpdate()` checks whether position or yaw changed enough to send a state update:

- If position changed more than `Provider.UPDATE_DISTANCE`, or yaw changed more than 1 degree, the zombie marks itself updated.
- It increments `zombieRegion.updates`.
- Later `ZombieManager.updateRegionsAndSendZombieStates()` serializes updated zombie states and sends them to clients in that nav region.

On clients, `OnUpdate()` interpolates toward the latest server target:

- `interpPositionTarget`
- `interpYawTarget`
- `Provider.INTERP_SPEED`

This is the classic split:

- Server owns truth.
- Clients smooth remote entities.

## Player Input, Prediction, And Reconciliation

Relevant files:

- `Assets/Runtime/Assembly-CSharp/Unturned/Player/PlayerInput.cs`
- `Assets/Runtime/Assembly-CSharp/Unturned/Player/PlayerMovement.cs`
- `Assets/Runtime/Assembly-CSharp/Unturned/Player/CapsuleHistory.cs`

The input model is a typical multiplayer action-game design:

- Client predicts its own movement immediately.
- Server re-simulates the same input authoritatively.
- Server acknowledges correct prediction or sends a correction.
- Client rewinds to the corrected state and replays unacknowledged input.

Important constants:

- `PlayerInput.RATE = 0.08f`
- `PlayerInput.SAMPLES = 4`
- `PlayerInput.TOCK_PER_SECOND = 50`

This means movement input packets are effectively around 12.5 Hz, while equipment/gun `tock` processing has a 50 Hz rhythm.

### Client Flow

In `PlayerInput.FixedUpdate()`, the local player:

1. Captures movement, stance, jump, sprint, lean, aim steady, plugin keys, and attack inputs.
2. Simulates life, stance, movement, equipment, and animator locally.
3. Stores the predicted movement in `clientInputHistory`.
4. Builds a `WalkingPlayerInputPacket` or `DrivingPlayerInputPacket`.
5. Sends the packet to the server with `SendInputs`.

For walking movement, the packet includes the client's post-simulation position:

- `WalkingPlayerInputPacket.clientPosition`

### Server Flow

The server receives inputs in `PlayerInput.ReceiveInputs()`:

- It rate-limits input spam.
- It rejects out-of-order simulation frame numbers.
- It enqueues packets in `serversidePackets`.
- It tracks large gaps between inputs as possible fake lag and applies a penalty frame counter.

Later, server `FixedUpdate()` dequeues a packet and runs the same simulation steps:

- `player.life.simulate(...)`
- `player.look.simulate(...)`
- `player.stance.simulate(...)`
- `player.movement.simulate(...)`
- `player.equipment.simulate(...)`
- `player.animator.simulate(...)`

For walking movement, the server compares:

- client-reported position: `walkingPacket.clientPosition`
- server result: `transform.position`

The tolerance is 2 cm in Unity world units:

```csharp
const float errorToleranceDistance = 0.02f; // 2cm
const float sqrErrorToleranceDistance = errorToleranceDistance * errorToleranceDistance;
if ((walkingPacket.clientPosition - serverPosition).sqrMagnitude > sqrErrorToleranceDistance)
{
    SendSimulateMispredictedInputs.Invoke(...);
}
else
{
    SendAckGoodInputs.Invoke(...);
}
```

This uses squared Euclidean distance to avoid a square-root calculation every check. Assuming the usual Unity convention of 1 unit = 1 meter, `0.02f` is 2 cm.

### Client Re-Simulation

If the server sends a correction, `ClientResimulate()`:

1. Removes acknowledged history up to the correction frame.
2. Temporarily sets stance, position, velocity, stamina, and some life timing fields to the server state.
3. Replays all remaining entries in `clientInputHistory`.
4. Restores local camera/aim rotations.

This is client-side prediction plus server reconciliation. It prevents movement from feeling delayed while still keeping the server authoritative.

## Hit Validation

Hit validation follows a similar philosophy: client convenience, server authority.

The client can send raycast hit results, but the server does not blindly trust them. It validates target identity, distance, and whether the hit point could reasonably be inside the target's current or recent bounds.

`BoundsHistory` stores one second of character-controller bounds at 50 tick rate:

- `CAPACITY = 50`
- `AddCharacterControllerBounds(...)`
- `ContainsPoint(...)`

This provides lag compensation without letting the client invent arbitrary hits. It is especially relevant for player hit validation because a remote player's server-current position may already differ from what the shooter saw when firing.

## Damage System

Relevant file:

- `Assets/Runtime/Assembly-CSharp/Unturned/Tools/DamageTool.cs`

`DamageTool` is the central damage gateway for:

- players
- zombies
- animals
- vehicles
- barricades
- structures
- resources
- level objects
- explosions

Interesting details:

- Player damage applies limb and clothing armor.
- Clothing quality can degrade when absorbing damage.
- Explosion armor averages several clothing slots rather than scaling based on the number of equipped pieces.
- PvP and friendly-fire checks are centralized.
- Explosion damage uses physics overlap and line-of-sight checks.
- There are plugin-facing hooks for damage requests.

The design is not tiny, but centralizing damage makes sense for a game with many damageable entity types and plugin/mod compatibility needs.

## Gun Runtime

Relevant files:

- `Assets/Runtime/Assembly-CSharp/Unturned/Useable/UseableGun.cs`
- `Assets/Runtime/Assembly-CSharp/Unturned/Bundles/ItemGunAsset.cs`
- many attachment, magazine, projectile, and item asset classes

`UseableGun.cs` is a very large common runtime for guns. It handles:

- fire modes
- ammo and magazines
- attachments
- recoil, sway, spread, and aiming
- ballistic bullets
- projectile launchers
- explosive bullets/projectiles
- hit processing
- server/client visual effects
- UI and attachment menus
- plugin events like `onBulletSpawned`, `onBulletHit`, and `onProjectileSpawned`

All guns are not separate scripts in the usual OOP sense. Instead, guns are data-driven through `ItemGunAsset`, magazine assets, barrel/sight/grip/tactical attachments, and item state bytes. `UseableGun` is the shared execution engine.

### Is This Bad Design?

It is costly design:

- Very large file.
- Many responsibilities in one class.
- High risk of unrelated regressions.
- Hard to test in isolation.
- Hard to reason about without knowing item data, networking, animation, and inventory.

But it is also understandable in context:

- Unturned is old and heavily live-operated.
- It has many legacy items and mods.
- Plugin compatibility matters.
- Server validation, client feel, visual effects, UI, and inventory state are tightly coupled for weapons.
- A data-driven single runtime can keep old content working without per-gun scripts.

So the best read is not "clean architecture", but "long-lived live game architecture". It has a god-class smell, yet it likely reflects years of compatibility, bug fixes, and practical constraints.

## Overall Takeaway

Unturned's runtime is more server-authoritative and performance-aware than it first appears.

The key pattern is:

- Use client prediction for responsiveness.
- Use server re-simulation and validation for authority.
- Use region/nav bounds to limit world simulation and network traffic.
- Use compatibility wrappers and data-driven assets to preserve old content and mods.
- Apply pragmatic optimizations when real maps or live servers expose bottlenecks.

It feels less like a greenfield engine and more like a codebase that has survived years of multiplayer edge cases.
