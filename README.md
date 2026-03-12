# Battle Blaster — Learning Log

Project built while following an Unreal Engine 5 course. It's a top-down shooter where the player controls a tank, aims with the mouse and fires projectiles at enemy towers spread across the levels.

**Engine:** Unreal Engine 5.6  
**Language:** C++ + Blueprint  
**Additional UE Modules:** UMG (UI), EnhancedInput, Niagara

---

## The Game

The player controls a tank (separate base and turret). The base moves with WASD, the turret rotates following the mouse cursor. Enemy towers shoot automatically when the tank enters their range. Destroying all towers loads the next level; dying reloads the current one.

### Controls

| Input | Action |
|---|---|
| W / S | Move forward / backward |
| A / D | Rotate the tank |
| Mouse | Aim the turret |
| Left Click | Shoot |

---

## Class Architecture

```
APawn
└── ABasePawn         ← shared logic (mesh, turret, shooting, death)
    ├── ATank         ← player-controlled, input + camera
    └── ATower        ← AI enemy, shoots at intervals using a Timer

AActor
└── AProjectile       ← projectile with movement, collisions, effects

UActorComponent
└── UHealthComponent  ← manages HP and delegates death to the GameMode

UGameModeBase
└── ABattleBlasterGameMode  ← central referee: countdown, win/lose, levels

UGameInstance
└── UBattleBlasterGameInstance  ← persists across levels, handles navigation

UUserWidget
└── UScreenMessage    ← HUD widget with dynamic text (countdown, victory, defeat)
```

---

## Concepts Learned

### Inheritance and Shared Base Class (BasePawn)

`ABasePawn` contains everything Tank and Tower have in common: the meshes (base + turret), the projectile spawn point, the turret rotation logic (`RotateTurret`) and the death logic with effects. Tank and Tower inherit from BasePawn and only add their specific behavior. This is the fundamental inheritance pattern applied to UE5.

### Enhanced Input System

The Tank uses UE5's new input system (`EnhancedInputComponent`) instead of the legacy one. In `BeginPlay` an `InputMappingContext` is registered on the `EnhancedInputLocalPlayerSubsystem`. Actions are `UInputAction` objects bound via `BindAction` with typed events (`ETriggerEvent::Triggered`, `ETriggerEvent::Started`). This system is more flexible than the legacy one because it separates input definition from its logic.

### Smooth Rotation with RInterpTo

The turret doesn't snap instantly to the target but uses `FMath::RInterpTo` to interpolate the rotation every frame, resulting in smooth movement. The `10.0f` parameter is the interpolation speed: the higher the value, the snappier the rotation.

```cpp
FRotator InterpolatedRotation = FMath::RInterpTo(
    TurretMesh->GetComponentRotation(),
    LookAtRotation,
    GetWorld()->GetDeltaSeconds(),
    10.0f
);
```

### Projectile with ProjectileMovementComponent

`AProjectile` uses `UProjectileMovementComponent` to handle physical movement without writing manual logic. Just setting `InitialSpeed` and `MaxSpeed` in the constructor is enough — the component handles the rest. Collision is managed by the `OnComponentHit` delegate dynamically bound with `AddDynamic` in `BeginPlay`.

### UE Damage System

The project uses UE's built-in damage system. The projectile calls `UGameplayStatics::ApplyDamage` on the hit actor. `UHealthComponent` registers on the owner's `OnTakeAnyDamage` delegate via `AddDynamic` in `BeginPlay` and scales HP accordingly. When HP reaches zero, it notifies the GameMode via `ActorDied`.

### GameMode as Central Referee

`ABattleBlasterGameMode` is the control point for the entire game session:
- In `BeginPlay` it finds all towers with `GetAllActorsOfClass` and assigns each one a reference to the Tank
- It manages the initial countdown with a repeating `FTimerHandle`
- It receives actor death notifications via `ActorDied` and decides victory or defeat
- After game over, it delegates navigation to the GameInstance

### GameInstance for Level Persistence

`UBattleBlasterGameInstance` survives level changes (unlike the GameMode which is destroyed on every load). It tracks `CurrentLevelIndex` and `LastLevelIndex` and handles `LoadNextLevel`, `RestartCurrentLevel` and `RestartGame`. Navigation happens via `UGameplayStatics::OpenLevel` with dynamically built names (`Level_1`, `Level_2`, etc.).

### Timers with FTimerHandle

Used in two different contexts in the project:
- In `ATower::BeginPlay`: repeating timer (`true`) to call `CheckFireCondition` every `FireRate` seconds
- In `ABattleBlasterGameMode`: repeating timer for the countdown, one-shot timer (`false`) for the post game over delay

```cpp
// Repeating timer — the Tower checks the fire condition every FireRate seconds
GetWorldTimerManager().SetTimer(FireTimerHandle, this, &ATower::CheckFireCondition, FireRate, true);
```

### UMG Widget in C++

`UScreenMessage` extends `UUserWidget` and uses `meta = (BindWidget)` to automatically bind a `UTextBlock` defined in the widget Blueprint. The `SetMessageText` method converts an `FString` to `FText` (the correct type for UE UI) and updates the text at runtime. The widget is created by the GameMode with `CreateWidget` and added to the screen with `AddToPlayerScreen`.

### Camera Shake

Both the projectile impact (`HitCameraShakeClass` in Projectile) and actor death (`DeathCameraShakeClass` in BasePawn) trigger a camera shake via `PlayerController->ClientStartCameraShake`. The shake type is a subclass of `UCameraShakeBase` configurable from Blueprint without touching C++.

---

## Assets

| Category | Files |
|---|---|
| Meshes | SM_TankBase, SM_TankTurret, SM_TowerBase, SM_TowerTurret, SM_Projectile, SM_Sphere |
| Effects (Niagara) | P_ProjectileTrail, P_HitEffect, P_DeathEffect |
| Audio | Explode_Audio, Thud_Audio |
| Materials | M_EmissiveBase, M_EmissiveGreen_Inst, M_EmissiveRed_Inst, M_ParticleBase, M_TextureBase, M_Texture_Inst, M_Particle_Inst |
| Textures | T_BasicGrid_M, T_BasicGrid_N, T_ColourGrid |
