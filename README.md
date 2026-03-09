# Battle Blaster — Learning Log

Progetto realizzato seguendo un corso di Unreal Engine 5. È un top-down shooter in cui il giocatore controlla un carro armato, mira con il mouse e spara proiettili contro torrette nemiche distribuite nei livelli.

**Engine:** Unreal Engine 5.6  
**Linguaggio:** C++ + Blueprint  
**Moduli UE aggiuntivi:** UMG (UI), EnhancedInput, Niagara

---

## Il Gioco

Il giocatore controlla un tank (base + torretta separata). La base si muove con WASD, la torretta ruota seguendo il cursore del mouse. Le torrette nemiche sparano automaticamente quando il tank entra nel loro raggio. Distruggere tutte le torrette carica il livello successivo; morire ricarica il livello corrente.

### Controlli

| Input | Azione |
|---|---|
| W / S | Avanza / Arretra |
| A / D | Ruota il tank |
| Mouse | Mira la torretta |
| Click Sinistro | Spara |

---

## Architettura delle Classi

```
APawn
└── ABasePawn         ← logica condivisa (mesh, torretta, sparo, morte)
    ├── ATank         ← controllato dal giocatore, input + camera
    └── ATower        ← nemico AI, spara a intervalli con Timer

AActor
└── AProjectile       ← proiettile con movimento, collisioni, effetti

UActorComponent
└── UHealthComponent  ← gestisce HP e delega la morte al GameMode

UGameModeBase
└── ABattleBlasterGameMode  ← arbitro centrale: countdown, win/lose, livelli

UGameInstance
└── UBattleBlasterGameInstance  ← persiste tra i livelli, gestisce la navigazione

UUserWidget
└── UScreenMessage    ← widget HUD con testo dinamico (countdown, vittoria, sconfitta)
```

---

## Concetti Appresi

### Ereditarietà e classe base condivisa (BasePawn)

`ABasePawn` contiene tutto ciò che Tank e Tower hanno in comune: le mesh (base + torretta), il punto di spawn del proiettile, la logica di rotazione della torretta (`RotateTurret`) e la logica di morte con effetti. Tank e Tower ereditano da BasePawn e aggiungono solo il comportamento specifico. Questo è il pattern fondamentale dell'ereditarietà applicato a UE5.

### Enhanced Input System

Il Tank usa il nuovo sistema di input di UE5 (`EnhancedInputComponent`) al posto del legacy. In `BeginPlay` viene registrato un `InputMappingContext` sull'`EnhancedInputLocalPlayerSubsystem`. Le azioni sono oggetti `UInputAction` bindati tramite `BindAction` con eventi tipizzati (`ETriggerEvent::Triggered`, `ETriggerEvent::Started`). Questo sistema è più flessibile del legacy perché separa la definizione dell'input dalla sua logica.

### Rotazione fluida con RInterpTo

La torretta non scatta istantaneamente sul bersaglio ma usa `FMath::RInterpTo` per interpolare la rotazione ogni frame. Il risultato è un movimento fluido. Il parametro `10.0f` è la velocità di interpolazione: più alto, più scattante.

```cpp
FRotator InterpolatedRotation = FMath::RInterpTo(
    TurretMesh->GetComponentRotation(),
    LookAtRotation,
    GetWorld()->GetDeltaSeconds(),
    10.0f
);
```

### Proiettile con ProjectileMovementComponent

`AProjectile` usa `UProjectileMovementComponent` per gestire il movimento fisico senza scrivere logica manuale. Basta impostare `InitialSpeed` e `MaxSpeed` nel costruttore e il componente si occupa di tutto. La collisione è gestita dal delegate `OnComponentHit` bindato dinamicamente con `AddDynamic` in `BeginPlay`.

### Damage System di UE

Il progetto usa il sistema di danno built-in di UE. Il proiettile chiama `UGameplayStatics::ApplyDamage` sull'attore colpito. `UHealthComponent` si registra sul delegate `OnTakeAnyDamage` dell'owner tramite `AddDynamic` in `BeginPlay`, e scala gli HP di conseguenza. Quando gli HP arrivano a zero, notifica il GameMode tramite `ActorDied`.

### GameMode come arbitro centrale

`ABattleBlasterGameMode` è il punto di controllo dell'intera sessione di gioco:
- In `BeginPlay` trova tutte le torrette con `GetAllActorsOfClass` e assegna a ciascuna il riferimento al Tank
- Gestisce il countdown iniziale con un `FTimerHandle` ripetuto
- Riceve la notifica di morte degli attori tramite `ActorDied` e decide se è vittoria o sconfitta
- Dopo il game over, delega la navigazione alla GameInstance

### GameInstance per la persistenza tra livelli

`UBattleBlasterGameInstance` sopravvive al cambio di livello (a differenza del GameMode che viene distrutto ad ogni load). Tiene traccia di `CurrentLevelIndex` e `LastLevelIndex` e gestisce `LoadNextLevel`, `RestartCurrentLevel` e `RestartGame`. La navigazione avviene tramite `UGameplayStatics::OpenLevel` con nomi costruiti dinamicamente (`Level_1`, `Level_2`, ecc.).

### Timer con FTimerHandle

Usati in due contesti diversi nel progetto:
- In `ATower::BeginPlay`: timer ripetuto (`true`) per chiamare `CheckFireCondition` ogni `FireRate` secondi
- In `ABattleBlasterGameMode`: timer ripetuto per il countdown, timer one-shot (`false`) per il delay post game over

```cpp
// Timer ripetuto — la Torre controlla la condizione di fuoco ogni FireRate secondi
GetWorldTimerManager().SetTimer(FireTimerHandle, this, &ATower::CheckFireCondition, FireRate, true);
```

### UMG Widget in C++

`UScreenMessage` estende `UUserWidget` e usa `meta = (BindWidget)` per collegare automaticamente un `UTextBlock` definito nel Blueprint del widget. Il metodo `SetMessageText` converte una `FString` in `FText` (il tipo corretto per la UI in UE) e aggiorna il testo a runtime. Il widget viene creato dal GameMode con `CreateWidget` e aggiunto allo schermo con `AddToPlayerScreen`.

### Camera Shake

Sia l'impatto del proiettile (`HitCameraShakeClass` in Projectile) che la morte di un attore (`DeathCameraShakeClass` in BasePawn) triggerano un camera shake tramite `PlayerController->ClientStartCameraShake`. Il tipo di shake è una sottoclasse di `UCameraShakeBase` configurabile dal Blueprint senza toccare il C++.

---

## Asset

| Categoria | File |
|---|---|
| Meshes | SM_TankBase, SM_TankTurret, SM_TowerBase, SM_TowerTurret, SM_Projectile, SM_Sphere |
| Effects (Niagara) | P_ProjectileTrail, P_HitEffect, P_DeathEffect |
| Audio | Explode_Audio, Thud_Audio |
| Materials | M_EmissiveBase, M_EmissiveGreen_Inst, M_EmissiveRed_Inst, M_ParticleBase, M_TextureBase, M_Texture_Inst, M_Particle_Inst |
| Textures | T_BasicGrid_M, T_BasicGrid_N, T_ColourGrid |
