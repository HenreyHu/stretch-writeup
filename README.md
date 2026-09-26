# STRETCH — Enemy AI, Persistence, and Editor Tooling

2D platformer built by a team of eight (six programmers, two artists) on a custom C++ engine written for this project. My role was Programmer, credited on the team as AI and UI Champion. I built seven systems for the game, enemy AI, the full persistence layer, checkpoint handling, editor undo and redo, the UI system, scene lifecycle, and the splash screen. This document walks through each one, with real code pulled from my own source files, and explains the reasoning behind it.

## Contents

- [Where this sits in the engine](#where-this-sits-in-the-engine)
- [Enemy AI](#enemy-ai)
- [The persistence layer](#the-persistence-layer)
- [Checkpoint interaction](#checkpoint-interaction)
- [Editor undo and redo](#editor-undo-and-redo)
- [UI system](#ui-system)
- [Scene lifecycle](#scene-lifecycle)
- [Splash screen](#splash-screen)
- [What I would do differently](#what-i-would-do-differently)
- [File map](#file-map)
- [Credits](#credits)

## Where this sits in the engine

| System | File(s) | What it owns |
|---|---|---|
| Enemy AI | `EnemyAI.cpp` / `.h` | Four-state FSM driving patrol, detection, and charge attacks |
| Persistence | `Serialisation.cpp` / `.h` | Save and load for levels, entities, prefabs, and checkpoints |
| Checkpoint trigger | `CheckpointSystem.cpp` / `.h` | Detects checkpoint interaction and fires the save |
| Editor undo/redo | `EditorHistory.cpp` / `.h` | Undo and redo for the level editor, generic across component types |
| UI | `UI.cpp` / `.h` | Menus, health bar, hit testing, controller navigation |
| Scene lifecycle | `SceneManager.h` | Thin load, restart, and unload interface used by the rest of the game |
| Splash screen | `SplashScreen.cpp` / `.h` | Fade-in, hold, fade-out intro sequence before the main menu |

## Enemy AI

The enemy behaviour is a four-state finite state machine, Idle, Patrol, Spot, and Charge. The FSM framework itself, the templated `StateMachine<T>` and `State<T>` classes, was written by a teammate. What I built on top of it is the actual enemy logic, the four state classes, the per-entity context, and the transition rules between them.

```cpp
// EnemyAI.cpp
namespace EnemyPatrolFSM {

    struct Context {
        Entity e{ 0 };
        float  idleTimer{ 0.0f };
        float  spotTimer{ 0.0f };
        float  spotCooldown{ 0.0f };
        int    facing{ -1 };        // -1 = left, +1 = right
        int    vfxSpawned{ -1 };
        Entity activeVFX{ 0 };
    };
```

Each enemy entity gets its own `Context`, so the same four state objects can be reused across every enemy in the level rather than allocating a new FSM per enemy. The state classes only ever touch data through `Context`, never through member variables of their own, which is what makes that sharing safe.

The transition rules are plain predicates over `Context`, registered once when the FSM is built for an entity.

```cpp
// EnemyAI.cpp, MachineBundle::Build
// 1. Idle -> Spot: player detected AND cooldown expired
sm.AddTransition(stateIdle, stateSpot, [](Context& c) {
    return c.spotCooldown <= 0.0f && PlayerInRadius(c.e);
    });

// 4. Patrol -> Idle: hit wall or ledge (flip facing)
sm.AddTransition(statePatrol, stateIdle, [](Context& c) {
    if (HitWall(c.e, c.facing)) {
        c.facing = -c.facing;
        return true;
    }
    return false;
    });

// 6. Charge -> Idle: hit wall/ledge OR physics stopped us
sm.AddTransition(stateCharge, stateIdle, [](Context& c) {
    if (HitWall(c.e, c.facing)) return true;

    // Fallback: physics collision stopped velocity
    float vx = Rb(c.e).velocity.x;
    if (c.facing > 0 && vx <= 0.01f) return true;
    if (c.facing < 0 && vx >= -0.01f) return true;

    return false;
    });
```

I wrote the transitions as small lambdas rather than a big switch statement in `Update`, so each transition reads as one rule in one place instead of being scattered across the state classes it connects. The Charge to Idle transition also has a fallback check on raw velocity, because a charging enemy that gets physically stopped by a collision (rather than detecting the wall itself first) still needs to leave the Charge state, and I did not want that edge case silently keeping the enemy stuck mid-charge.

The Charge state is the most involved one, because it has to spawn a one-shot VFX at a specific animation frame, and keep that VFX following the enemy without spawning it twice.

```cpp
// EnemyAI.cpp, Charge::OnUpdate
auto clipIt = animator.AnimationsMap.find("charge");
if (clipIt != animator.AnimationsMap.end() && !clipIt->second.FramesList().empty()) {
    int frameIdx = clipIt->second.GetCurrentFrame();
    if (frameIdx >= 0 && frameIdx < static_cast<int>(clipIt->second.FramesList().size())) {
        int spriteIdx = clipIt->second.FramesList()[frameIdx].sprite();

        // Snap frames: 28 (first snap) and 34 (second snap)
        bool isSnapFrame = (spriteIdx == 28);
        bool wasSnapFrame = (ctx.vfxSpawned == 28);

        // Trigger VFX on entering a snap frame (not already on one)
        if (isSnapFrame && !wasSnapFrame) {
            Entity player = FindPlayerEntity();
            if (player != 0 && HasTr(player)) {
                float enemyX = Tr(ctx.e).Position.x;
                float playerX = Tr(player).Position.x;

                bool stillApproaching = (ctx.facing > 0) ? (enemyX < playerX) : (enemyX > playerX);

                if (stillApproaching) {
                    ctx.activeVFX = SpawnChargeVFX(ctx.e, ctx.facing);
                }
            }
        }

        ctx.vfxSpawned = spriteIdx;
    }
}
```

Rather than driving the VFX off a timer, I tied it to the actual sprite index the animation was showing, comparing the current frame against the previous frame so the spawn only fires on the transition into the snap frame, not on every frame the enemy happens to be on it. The `stillApproaching` check also stops the VFX from spawning if the enemy already passed the player, which was a real bug during testing where a fast enemy would spawn the effect behind the player after already colliding with them.

Finally, the behaviour is registered with the engine's logic system rather than hardcoded into the ECS core, so the ECS itself has no idea "enemy AI" exists.

```cpp
// EnemyAI.cpp
void RegisterBehaviours() {
    using namespace ECS;
    auto coordinator = Coordinator::GetInstance();
    auto logicSystem = coordinator->GetSystem<LogicSystem>();

    if (logicSystem) {
        Behaviour noneBehaviour;
        noneBehaviour.init = nullptr;
        noneBehaviour.update = nullptr;
        noneBehaviour.end = nullptr;
        logicSystem->Register("None", noneBehaviour);

        Behaviour enemyPatrol;
        enemyPatrol.init = EnemyPatrol::Init;
        enemyPatrol.update = EnemyPatrol::Update;
        enemyPatrol.end = EnemyPatrol::End;
        logicSystem->Register("EnemyPatrol", enemyPatrol);
    }
}
```

This is the same registration pattern the persistence layer uses for components (see below), function pointers registered by name against a shared system, rather than the system knowing about every behaviour type up front. It means adding a new enemy behaviour later does not require touching the ECS or the logic system at all, only writing the new behaviour and registering it here.

## The persistence layer

This is the largest single system I own on the project, and it undersells itself if you call it "save and load." It is the engine's entire data persistence layer, full level save and load, entity snapshots used by the editor's undo system, prefab persistence, and a separate checkpoint save file for player progress. All of it goes through the same RapidJSON-based component registry.

The registry itself is small. A component type registers a load function and a save function against its name, and everything else in the file works off those two maps.

```cpp
// Serialisation.cpp
std::unordered_map<std::string, LoadFn> g_loaders;
std::unordered_map<std::string, SaveFn> g_savers;

void RegisterComponent(const char* componentName, LoadFn loadFn, SaveFn saveFn) {
    g_loaders[componentName] = loadFn;
    g_savers[componentName] = saveFn;
}
```

Saving an entity is then just walking whatever component names are registered and asking each one to serialize itself if the entity has it.

```cpp
// Serialisation.cpp
static bool SaveEntityInternal(
    ECS::Entity e,
    const std::vector<std::string>& componentNames,
    rapidjson::Value& outEntity,
    rapidjson::Document::AllocatorType& a)
{
    rapidjson::Value comps(rapidjson::kObjectType);

    for (const std::string& name : componentNames)
    {
        auto it = g_savers.find(name);
        if (it == g_savers.end())
            continue;

        rapidjson::Value c(rapidjson::kObjectType);

        if (it->second(e, c, a))
        {
            comps.AddMember(rapidjson::Value(name.c_str(), a), c, a);
        }
    }

    outEntity.SetObject();
    outEntity.AddMember("components", comps, a);
    return true;
}
```

The save function returning `false` for a component the entity does not actually have is what lets one loop handle every entity regardless of which components it carries, instead of writing a separate save path per entity archetype.

Every entity tracked by the engine also carries a lifetime category, which decides what survives a level change.

```cpp
// Serialisation.h
enum class EntityLifetime {
    LEVEL = 0,       // Tied to current level (cleared on level change)
    PERSISTENT = 1,  // Survives level changes (inventory, quest items)
    TEMPORARY = 2    // Not saved (bullets, particles, VFX)
};
```

```cpp
// Serialisation.cpp, ClearSceneRuntime
for (Entity e : allLiveEntities)
{
    EntityLifetime lifetime = GetEntityLifetime(e);
    bool shouldDestroy = false;

    if (lifetime == EntityLifetime::LEVEL || lifetime == EntityLifetime::TEMPORARY) {
        shouldDestroy = true;
    }
    else if (lifetime == EntityLifetime::PERSISTENT && clearPersistent) {
        shouldDestroy = true;
    }

    if (shouldDestroy) {
        entitiesToDestroy.push_back(e);
    }
    else {
        survivingEntities.insert(e);
        survivingLifetimes[e] = lifetime;
        if (g_entityComponents.count(e)) {
            survivingComponents[e] = g_entityComponents[e];
        }
    }
}
```

I split this into three categories instead of a plain boolean "persistent or not" because temporary entities like particles and VFX needed to disappear on a level change the same way level entities do, but for a different reason, they were never meant to be saved at all, not even within the same level. Folding them into "not persistent" would have been correct behaviour by accident rather than by design.

The same component registry backs a second, lighter-weight path, snapshotting a single entity to a JSON string rather than a whole level to a file. This is what the editor's undo system runs on.

```cpp
// Serialisation.cpp
std::string SaveEntitySnapshot(ECS::Entity e)
{
    std::vector<std::string> compNames;
    for (const auto& kv : g_savers)
        compNames.push_back(kv.first);

    rapidjson::Document doc;
    doc.SetObject();
    auto& a = doc.GetAllocator();

    doc.AddMember("fileType", "entitySnapshot", a);
    doc.AddMember("version", 1, a);

    int lifetimeInt = static_cast<int>(GetEntityLifetime(e));
    doc.AddMember("lifetime", lifetimeInt, a);

    rapidjson::Value entities(rapidjson::kArrayType);
    rapidjson::Value eobj(rapidjson::kObjectType);

    SaveEntityInternal(e, compNames, eobj, a);
    entities.PushBack(eobj, a);

    doc.AddMember("entities", entities, a);

    rapidjson::StringBuffer buffer;
    rapidjson::Writer<rapidjson::StringBuffer> writer(buffer);
    doc.Accept(writer);

    return buffer.GetString();
}
```

I deliberately reused `SaveEntityInternal`, the same function full-level saving uses, instead of writing a second serializer for single entities. Keeping one save path for "how a component turns into JSON" meant the undo system and the level save file could never silently drift apart in what they consider an entity's full state.

Checkpoint progress is a third, separate concern again, and gets its own small save file rather than reusing the level format, since a checkpoint only needs a level path, a spawn position, and the player's health, not an entire entity list.

```cpp
// Serialisation.cpp
bool SaveCheckpointProgress(const std::string& file,
    const std::string& levelPath,
    const MathLibrary::Vector2D& spawnPos,
    int playerHealth,
    bool startOfLevel)
{
    try {
        rapidjson::Document doc;
        doc.SetObject();
        auto& a = doc.GetAllocator();

        doc.AddMember("fileType", "checkpointSave", a);
        doc.AddMember("version", 1, a);
        doc.AddMember("levelPath", rapidjson::Value(levelPath.c_str(), a), a);

        rapidjson::Value spawn(rapidjson::kObjectType);
        spawn.AddMember("x", static_cast<double>(spawnPos.x), a);
        spawn.AddMember("y", static_cast<double>(spawnPos.y), a);
        doc.AddMember("playerSpawn", spawn, a);
        doc.AddMember("playerHealth", static_cast<double>(playerHealth), a);
        doc.AddMember("startOfLevel", startOfLevel, a);

        WriteJson(file, doc, true);
        return true;
    }
    catch (const std::exception& e) {
        std::cerr << "[Save] Checkpoint save failed: " << e.what() << "\n";
        return false;
    }
}
```

`LoadCheckpointProgress` is the mirror of this, and `HasCheckpointSave` just tries to parse the file and validate its header, which is what the main menu uses to decide whether to show a "Continue" button at all.

All three paths, full level, single-entity snapshot, and checkpoint progress, meet in `LoadScene`, which is the one function everything else in the game calls to actually change what is on screen.

```cpp
// Serialisation.cpp
bool LoadScene(const std::string& filename, bool isGameplayLevel, bool isCheckpoint, bool enteringEditMode)
{
    try
    {
        ClearSceneRuntime();
        SetActiveLevelPath(filename);

        bool ok = LoadLevel(filename);
        if (!ok)
            return false;

        auto& C = *ECS::Coordinator::GetInstance();
        C.GetSystem<ECS::PlatformSystem>()->SetupPlatforms(ImGuiSystem::Instance()->GetSceneState());
        C.GetSystem<LevelTransitionSystem>()->OnLevelLoad(isGameplayLevel, isCheckpoint, enteringEditMode);

        return true;
    }
    catch (const std::exception& e)
    {
        std::cerr << "[Scene] Load failed: " << e.what() << "\n";
        return false;
    }
}
```

Clearing the runtime before loading, rather than loading on top of whatever is already there, is what makes `LoadScene` safe to call from anywhere, the main menu starting a new game, a level transition, or a checkpoint respawn, without any of those callers needing to know what state the world was in beforehand.

## Checkpoint interaction

`CheckpointSystem` is the gameplay-facing piece that decides when a checkpoint should actually fire a save, sitting between the trigger volume in the level and the persistence layer above. It follows the same self-wiring pattern as the engine's `TriggerSystem`, creating one hidden observer entity the first time it runs, rather than requiring a "game manager" entity to exist in every level.

```cpp
// CheckpointSystem.cpp
void CheckpointSystem::EnsureObserverWired()
{
    if (_observerWired) return;

    ECS::Coordinator& C = *ECS::Coordinator::GetInstance();

    _observerEntity = C.CreateEntity();
    C.AddComponent<ECS::C_Entity>(_observerEntity, ECS::C_Entity{ "MSG_OBSERVER_CHECKPOINT" });
    C.AddComponent<C_Observer>(_observerEntity, C_Observer{ _observerEntity });

    C.GetComponent<C_Observer>(_observerEntity)
        .AttachHandler(MSG_CHECKPOINT_SAVE,
            reinterpret_cast<MESSAGE_HANDLER>(ProcessCheckpointSave));

    Serialisation::TrackEntity(_observerEntity, Serialisation::EntityLifetime::PERSISTENT);
    Serialisation::TrackComponent(_observerEntity, "Entity");

    _observerWired = true;
}
```

The actual save is fired through the engine's message system rather than called directly, which keeps `CheckpointSystem` from needing to know anything about how a save happens, only that it should broadcast one.

```cpp
// CheckpointSystem.cpp, Update
if (!alreadySubscribed) {
    observable.Subscribe(MSG_CHECKPOINT_SAVE,
        &coordinator.GetComponent<C_Observer>(_observerEntity));
}

CheckpointSaveMsg msg(entity);
observable.ProcessMessage(&msg);

checkpoint.hasSaved = true;
checkpoint.feedbackTimer = ECS::C_Checkpoint::FEEDBACK_DURATION;
```

The `oneTimeUse` and `hasSaved` guards on the checkpoint component (checked earlier in `Update`) exist because a save-on-touch checkpoint sitting in a doorway the player walks through repeatedly would otherwise rewrite the save file every single frame the player stands in the trigger zone.

## Editor undo and redo

The undo system is generic across every component type in the engine, rather than hardcoding "undo a transform edit" and "undo a component edit" as separate mechanisms. It does this by reusing the same JSON entity snapshots the persistence layer already produces, capturing a before and after snapshot around an edit and replaying them on undo or redo.

```cpp
// EditorHistory.cpp, BeginEdit / EndEdit
void BeginEdit(ECS::Entity entity)
{
    if (!EntityIsValid(entity)) return;

    if (g_state.isComponentEditing && g_state.componentEditEntity == entity)
        return;

    g_state.isComponentEditing = true;
    g_state.componentEditEntity = entity;
    g_state.componentEditBeforeJson = Serialisation::SaveEntitySnapshot(entity);
}

void EndEdit(ECS::Entity entity, const char* description)
{
    if (!g_state.isComponentEditing || g_state.componentEditEntity != entity)
        return;

    std::string afterJson = Serialisation::SaveEntitySnapshot(entity);

    if (afterJson == g_state.componentEditBeforeJson) {
        g_state.isComponentEditing = false;
        g_state.componentEditEntity = 0;
        g_state.componentEditBeforeJson.clear();
        return;
    }

    Action action{};
    action.type = ActionType::ComponentEdit;
    action.entity = entity;
    action.beforeJson = std::move(g_state.componentEditBeforeJson);
    action.afterJson = std::move(afterJson);
    action.description = description ? description : "Edit";

    PushAction(std::move(action));
}
```

Comparing the before and after JSON as plain strings before pushing an undo action means a widget that gets clicked but not actually changed does not clutter the undo stack with a no-op entry. `BeginEdit` is also written to be idempotent, safe to call every frame the inspector is drawing a widget, since it only captures a fresh snapshot the first time it sees a given entity being edited.

Undoing a component edit destroys the entity and recreates it from the stored JSON, which meant entity IDs could change out from under the rest of the editor mid-undo, so I added an explicit remap step.

```cpp
// EditorHistory.cpp
void RemapEntity(ECS::Entity oldEntity, ECS::Entity newEntity)
{
    if (oldEntity == newEntity) return;

    for (auto& action : g_state.undoStack)
        if (action.entity == oldEntity) action.entity = newEntity;

    for (auto& action : g_state.redoStack)
        if (action.entity == oldEntity) action.entity = newEntity;

    if (g_state.isComponentEditing && g_state.componentEditEntity == oldEntity)
        g_state.componentEditEntity = newEntity;

    if (g_state.remapCallback)
        g_state.remapCallback(oldEntity, newEntity);
}
```

The callback at the end is what lets `ImGuiSystem` keep the correct entity selected in the inspector after an undo recreates it under a new ID, without `EditorHistory` needing to know that `ImGuiSystem` or a "currently selected entity" concept exists at all.

A plain transform edit (dragging a gizmo) does not need the full destroy-and-recreate path, so it gets a lighter, dedicated snapshot type instead of going through JSON.

```cpp
// EditorHistory.cpp
void EndTransformEdit(ECS::Entity entity)
{
    if (!g_state.isTransformEditing || g_state.editingEntity != entity) return;

    g_state.isTransformEditing = false;
    if (!EntityIsValid(entity)) return;

    TransformSnapshot after = CaptureInternal(entity);
    if (IsEqual(g_state.transformBefore, after)) return;

    Action action{};
    action.type = ActionType::Transform;
    action.entity = entity;
    action.transformBefore = g_state.transformBefore;
    action.transformAfter = after;
    action.hasTransform = true;
    action.description = "Transform";

    PushAction(action);
}
```

I kept this path separate from the generic component-edit path because a transform drag happens every frame during a gizmo interaction, and reserializing an entire entity to JSON on every one of those frames would have been wasted work for something as small as a position change.

## UI system

The UI layer owns every menu in the game, the health bar, hit testing for buttons and sliders, and controller navigation, all built on top of the ECS as a plain `System` rather than a separate framework.

Buttons can live in either world space or normalized device coordinates depending on the menu, so hit testing has two versions of the same rotated-rectangle check.

```cpp
// UI.cpp
bool UISystem::PointInRectNDC(const MathLibrary::Vector2D& p,
    const ECS::C_Transform& t,
    float aspectRatio)
{
    float dx = p.x - t.Position.x;
    float dy = p.y - t.Position.y;

    // Inverse of rendering matrix for hit detection:
    // [cos,           sin/aspect]
    // [-sin*aspect,   cos       ]

    float rad = t.Angle * 3.14159265f / 180.0f;
    float c = std::cos(rad);
    float s = std::sin(rad);

    float localX = c * dx + s * dy / aspectRatio;
    float localY = -s * dx * aspectRatio + c * dy;

    float halfW = t.Scale.x * 0.5f;
    float halfH = t.Scale.y * 0.5f;

    return (localX >= -halfW && localX <= halfW &&
        localY >= -halfH && localY <= halfH);
}
```

The NDC version divides and multiplies by `aspectRatio` on opposite axes because NDC space is stretched horizontally relative to world space, so a rotated button hitbox that looked correct in world coordinates would otherwise test against the wrong region once the same math ran in NDC.

Sub-menus need to block interaction with whatever menu is behind them, a settings popup should not let clicks fall through to the main menu underneath it, so that check lives in one place rather than being repeated per menu.

```cpp
// UI.cpp
bool UISystem::IsParentMenuBlocked(const ECS::C_UIElement& ui, bool fullBlock) const
{
    if (ui.menuId == "MainMenu" &&
        (IsMenuActive("QuitConfirm") || IsMenuActive("NewGameConfirm") || (fullBlock && IsMenuActive("MainSettingsMenu"))))
    {
        return true;
    }

    if (ui.menuId == "PauseMenu" &&
        (IsMenuActive("ReturnToMainConfirm") || (fullBlock && (IsMenuActive("PauseSettingsMenu") || IsMenuActive("HowToPlay")))))
    {
        return true;
    }

    return false;
}
```

Controller navigation was added after the mouse-driven menus already worked, so rather than rewriting button interaction around a "selected index," I kept the mouse path as-is and added a second path that drives the same click and hover callbacks from a virtual cursor position.

```cpp
// UI.cpp, UpdateControllerMenuNavigation
if (moveNext && _stickWasCentered) {
    _currentSelection++;
    if (_currentSelection >= activeButtons.size()) _currentSelection = 0;
    _stickWasCentered = false;
    C.GetSystem<AudioManager>()->PlayEventInstance("event:/SFX/UI/SFX_UI_Hover", true, "SFX");
}
else if (movePrev && _stickWasCentered) {
    _currentSelection--;
    if (_currentSelection < 0) _currentSelection = static_cast<int>(activeButtons.size()) - 1;
    _stickWasCentered = false;
    C.GetSystem<AudioManager>()->PlayEventInstance("event:/SFX/UI/SFX_UI_Hover", true, "SFX");
}
```

`_stickWasCentered` exists to stop the stick registering as ten separate "move next" presses while the player holds it in one direction, the selection only advances again once the stick passes back through the dead zone.

Health and checkpoint respawn share one path, `LoadCheckpoint`, whether the player is loading a save from the main menu or respawning after death.

```cpp
// UI.cpp
bool UISystem::LoadCheckpoint(bool useSavedHealth, int forcedHealth) {
    std::string savePath = Serialisation::GetSavePath();

    if (!Serialisation::HasCheckpointSave(savePath)) {
        return false;
    }

    std::string levelPath;
    MathLibrary::Vector2D spawnPos;
    int savedHealth = 3;
    bool startOfLevel{ false };

    if (!Serialisation::LoadCheckpointProgress(savePath, levelPath, spawnPos, savedHealth, startOfLevel)) {
        return false;
    }

    const int targetHealth = useSavedHealth ? savedHealth : forcedHealth;

    SceneManager::GetInstance()->LoadScene(levelPath, true, !startOfLevel);

    ECS::Coordinator& C = *ECS::Coordinator::GetInstance();
    for (auto& e : C.GetAllEntities()) {
        if (!C.HasComponent<ECS::C_PlayerController>(e)) continue;
        if (!C.HasComponent<ECS::C_Transform>(e)) continue;

        auto& transform = C.GetComponent<ECS::C_Transform>(e);
        transform.Position.x = spawnPos.x;
        transform.Position.y = spawnPos.y;

        if (C.HasComponent<ECS::C_Health>(e)) {
            auto& health = C.GetComponent<ECS::C_Health>(e);
            health.currentHealth = static_cast<float>(targetHealth);
            health.isDead = false;
            health.invincibilityTimer = 0.0f;
        }
        break;
    }

    _respawnFromDeath = false;
    return true;
}
```

The `useSavedHealth` flag is why one function covers both cases, loading a save from the main menu restores the health you had when you saved, while respawning after dying resets you to full health at the same checkpoint, and the caller decides which behaviour it wants rather than `LoadCheckpoint` guessing from context.

## Scene lifecycle

`SceneManager` is a thin façade in front of the persistence layer, giving the rest of the game a small, stable interface, load, restart, unload, rather than every caller reaching into `Serialisation` directly.

```cpp
// SceneManager.h
bool RestartScene(bool enteringEditMode) {
    std::string checkpointPath{};
    MathLibrary::Vector2D spawnPos{};
    int savedHealth = 3;
    bool startOfLevel{ false };
    bool loadedCheckpoint{ Serialisation::LoadCheckpointProgress(
        Serialisation::GetSavePath(),
        checkpointPath,
        spawnPos,
        savedHealth,
        startOfLevel
    ) };

    const std::string& path = Serialisation::GetActiveLevelPath();
    if (path.empty()) return false;

    bool result{ Serialisation::LoadScene(
        (loadedCheckpoint && !enteringEditMode ? checkpointPath : path),
        !enteringEditMode,
        (loadedCheckpoint ? !startOfLevel : false),
        enteringEditMode) };

    return result;
}
```

I wrote this as a header-only singleton, small enough that a separate `.cpp` file would have added nothing but a longer include chain for every system that just wants to call `LoadScene`.

## Splash screen

The splash screen is a small fade-in, hold, fade-out state machine that runs before the main menu, reusing the engine's existing cutscene shader uniforms instead of adding a new rendering path just for still images.

```cpp
// SplashScreen.cpp
void SplashScreen::Update(float dt, InputManager& input) {
    if (!_active) return;
    if (_currentIndex >= static_cast<int>(_entries.size())) {
        _active = false;
        return;
    }

    const SplashEntry& entry = _entries[_currentIndex];

    if (input.WasKeyPressed(GLFW_KEY_ENTER) || input.WasKeyPressed(GLFW_KEY_SPACE)) {
        AdvanceToNext();
        return;
    }

    _timer += dt;

    switch (_phase) {
    case Phase::FadeIn:
        _alpha = (entry.fadeInTime > 0.0f) ? (std::min)(_timer / entry.fadeInTime, 1.0f) : 1.0f;
        if (_timer >= entry.fadeInTime) { _phase = Phase::Hold; _timer = 0.0f; }
        break;

    case Phase::Hold:
        _alpha = 1.0f;
        if (_timer >= entry.holdTime) { _phase = Phase::FadeOut; _timer = 0.0f; }
        break;

    case Phase::FadeOut:
        _alpha = (entry.fadeOutTime > 0.0f) ? 1.0f - (std::min)(_timer / entry.fadeOutTime, 1.0f) : 0.0f;
        if (_timer >= entry.fadeOutTime) AdvanceToNext();
        break;
    }
}
```

Letting the player skip with Enter or Space before checking the phase logic at all was a deliberate ordering choice, so a skip always works immediately regardless of which fade phase the current splash entry happens to be in.

## What I would do differently

The controller navigation code in `UpdateControllerMenuNavigation` grew organically on top of the mouse-driven path rather than being designed alongside it from the start, and it shows, the function is long and has several booleans tracking state that a proper input-mapping layer would have handled more cleanly.

The persistence layer's component registry uses raw function pointers (`LoadFn`, `SaveFn`) rather than `std::function`, which was fine because every registered component's load and save logic is a free function with no captured state, but it does mean a component that needed to close over any state at registration time would not fit the current design without changes.

`EditorHistory`'s generic component-edit path snapshots and diffs an entity's full JSON on every edit, which is simple and correct but not cheap for entities with many components, a dirty-flag system that only serialized the components that actually changed would scale better for larger scenes.

## File map

| File | What it contains |
|---|---|
| `EnemyAI.h` / `.cpp` | The four-state enemy FSM and its registration with the logic system |
| `Serialisation.h` / `.cpp` | The component registry, level save/load, entity snapshots, checkpoint save/load |
| `CheckpointSystem.h` / `.cpp` | Checkpoint trigger detection and the message-based save handoff |
| `EditorHistory.h` / `.cpp` | Generic and transform-specific undo/redo for the level editor |
| `UI.h` / `.cpp` | Menus, hit testing, health bar, controller navigation |
| `SceneManager.h` | Load, restart, and unload façade over the persistence layer |
| `SplashScreen.h` / `.cpp` | Fade-in/hold/fade-out intro sequence |

## Credits

STRETCH was built by a team of eight, six programmers and two artists. The systems documented in this write-up, enemy AI, the persistence layer, checkpoint handling, editor undo and redo, the UI system, scene lifecycle, and the splash screen, are mine.
