---
name: unity-developer
description: Comprehensive Unity 6 skill — broad capabilities (rendering, performance, architecture, platforms) plus enforced C# coding standards and a per-project Singleton-vs-DI architecture decision. Triggers when writing, reviewing, or refactoring Unity C# code, implementing features, setting up architecture, working with events, or reviewing changes.
---

# Unity Development Standards

> ⚠️ **Unity 6 (C# 9):** All patterns and examples are compatible with Unity 6, which uses C# 9. No C# 10+ features are used.

## 🧭 STEP 0: Choose the Architecture (ASK FIRST on a new project)

**Architecture is NOT fixed.** Before writing any architecture-level code (managers, services, events, data access) in a **new** project, you MUST ask the developer which approach to use:

> **"Dự án này dùng kiến trúc nào — Singleton hay DI (VContainer/SignalBus)?"**

Then follow that choice consistently for the whole project. Quick guidance to help them decide:

| Use **Singleton** when… | Use **DI** when… |
| --- | --- |
| Team of **1–4 people** | Team of **5+ people** |
| Small / casual / hyper-casual game | Large, long-lived, live-ops game |
| Few global managers, clear lifecycle | Many interlocking systems, complex lifecycles |
| Little/no unit testing (test by playing) | Serious unit testing / TDD on business logic |
| Ship fast, low overhead | Need swappable interfaces, multi-platform/SDK variants |

> ✅ **Default for small teams (1–2 devs): Singleton is enough.** Do not impose DI on a small project.

**Rules for choosing:**

- **New project →** ASK, then record the decision and apply it everywhere.
- **Existing project →** Do NOT ask. Detect and follow the architecture already in use (e.g. a `Singleton<T>` base class means Singleton; a `LifetimeScope`/`[Inject]` means DI).
- After the choice, apply the matching section below: **[Path A: Singleton](#path-a-singleton-default-for-small-teams)** or **[Path B: DI](#path-b-di-vcontainer--signalbus-or-di-container--publishersubscriber)**.

## Skill Purpose

This skill enforces comprehensive Unity development standards with **CODE QUALITY FIRST**:

**Priority 1: Code Quality & Hygiene (MOST IMPORTANT — applies to BOTH architectures)**

- Access modifiers (least-accessible by default)
- Prefer throwing exceptions over swallowing errors (see logging note per architecture)
- `readonly`/`const`, `nameof`

**Priority 2: Modern C# Patterns (applies to BOTH)**

- LINQ over loops, extension methods, expression bodies
- Null-coalescing operators, pattern matching
- Modern C# features (records, init, with)

**Priority 3: Unity Architecture (depends on STEP 0 choice)**

- **Singleton** path, OR **DI** path (VContainer + SignalBus / DI Container + Publisher/Subscriber)
- Resource management, UniTask async patterns (both)

**Priority 4: Performance & Review (applies to BOTH)**

- LINQ optimization, allocation prevention
- Code review guidelines

## When This Skill Triggers

- Writing or refactoring Unity C# code
- Setting up project architecture (→ run STEP 0 first)
- Implementing Unity features, managers, or services
- Working with events and signals
- Accessing or modifying game data
- Reviewing code changes or pull requests

## 🔴 CRITICAL: Code Quality Rules (CHECK FIRST — both architectures)

ALWAYS enforce these BEFORE writing any code, regardless of architecture:

1. **Use least accessible access modifier** — `private` by default
2. **Handle errors deliberately** — Throw exceptions for truly exceptional cases; only log-and-continue when there is a real fallback
3. **Use `readonly` for fields** — Mark fields that aren't reassigned (`[SerializeField]`/inspector fields are exempt)
4. **Use `const` for constants** — Constants should be `const`, not `readonly`
5. **Use `nameof` for strings** — Never hardcode property/parameter names (the project's `[ClassName]` log-prefix convention is exempt)

```csharp
// ✅ EXCELLENT: Quality rules enforced (architecture-neutral)
public sealed class DamageCalculator
{
    private const int CritMultiplier = 2;

    public int Calculate(int baseDamage, bool isCrit) =>
        isCrit ? baseDamage * CritMultiplier : baseDamage;
}

#if UNITY_EDITOR
public class EditorTool
{
    public void Process() => Debug.Log("Processing..."); // Debug.Log OK in editor
}
#endif
```

## 🟡 Modern C# Patterns (both architectures)

```csharp
// ✅ GOOD: LINQ instead of loops
var activeEnemies = allEnemies.Where(e => e.IsActive).ToList();

// ✅ GOOD: Expression bodies
public int Health => currentHealth;

// ✅ GOOD: Null-coalescing
var name = playerName ?? "Unknown";

// ✅ GOOD: Pattern matching
if (obj is Player player) player.TakeDamage(10);
```

---

# Path A: Singleton (default for small teams)

Use this when STEP 0 picked Singleton. **This is the recommended default for 1–2 dev projects.**

### Rules

Singleton-style projects typically use **three accepted forms of global access** — pick the one that matches the type, do NOT force everything into a `Singleton<T>` base. **Detect which forms the project already uses and follow them; don't impose ones it doesn't have.**

1. **MonoBehaviour singleton + `.Instance`** — for managers that ARE MonoBehaviours (audio, SDK, navigation, etc.). Inherit the project's shared singleton base; do NOT hand-roll a new one per class.
2. **`static class` + static accessor property** — for **plain-C# data/config holders**. These deliberately do NOT inherit the MonoBehaviour singleton base (they aren't MonoBehaviours). This is a valid global, NOT an "ad-hoc singleton".
3. **Service-locator (`Register`/`Resolve`)** — for **non-MonoBehaviour** objects created with `new` and shared globally.

- ✅ Access MonoBehaviour managers via `.Instance`; plain-C# data via its static accessor; `new`-created services via the service-locator. Match the existing convention.
- ✅ Use an `Initialize()` method on managers; call them in a defined order from a single top-level initializer.
- ✅ Static/global data access is fine — no Controller layer required. If a data holder is **async-initialized**, guard access with its init flag (the accessor is null/empty until init completes).
- ✅ Events: C# `event`/`Action` or `UnityEvent` (Singleton projects also commonly communicate via direct `.Instance` calls). When you DO use events, **always unsubscribe** in `OnDisable`/`OnDestroy`.
- ✅ Logging: `Debug.Log`/`Debug.LogError` is acceptable in runtime; a `[ClassName]` prefix is encouraged for traceability.
- ✅ Use UniTask for async; thread a `CancellationToken` through awaits.

### Singleton base class

Before writing a MonoBehaviour singleton, **search the project for an existing shared base** (common names: `Singleton<T>`, `StaticInstance<T>`, `PersistentSingleton<T>`, `MonoSingleton<T>`). If one exists, **inherit it; never redefine it** (a duplicate generic base = compile error). Read the base before relying on it — implementations differ: some destroy duplicate instances and some don't, some auto-create the GameObject when missing and some return null. Only introduce a new base if the project genuinely has none.

### Example manager

```csharp
public sealed class ScoreManager : Singleton<ScoreManager>
{
    private int score;
    public int Score => score;

    public void Initialize() => score = 0;

    public void Add(int amount)
    {
        score += amount;
        OnScoreChanged?.Invoke(score);
    }

    public event System.Action<int> OnScoreChanged;
}

// Subscriber must unsubscribe:
public sealed class ScoreHud : MonoBehaviour
{
    private void OnEnable()  => ScoreManager.Instance.OnScoreChanged += Refresh;
    private void OnDisable() => ScoreManager.Instance.OnScoreChanged -= Refresh;
    private void Refresh(int score) { /* update UI */ }
}
```

---

# Path B: DI (VContainer + SignalBus OR DI Container + Publisher/Subscriber)

Use this ONLY when STEP 0 picked DI (large team / long-lived project).

### Choose ONE stack

**Option 1: VContainer + SignalBus**

- ✅ VContainer for dependency injection
- ✅ SignalBus for events
- ✅ `[Preserve]` attribute on constructors

**Option 2: DI Container + Publisher/Subscriber**

- ✅ DI container wrapper
- ✅ `IPublisher`/`ISubscriber` for events
- ✅ `[Inject]` attribute on constructors

**DI universal rules:**

- ✅ Use Data Controllers (NEVER direct data access)
- ✅ Inject an `ILogger` for runtime (no `#if` guards, no `[prefix]`, never in constructors); use `logger.Method()` directly (DI guarantees non-null)
- ✅ Always implement `IDisposable` and unsubscribe from every signal
- ✅ Unload assets in `Dispose`
- ✅ Use UniTask for async

### Unity Architecture (VContainer)

```csharp
using UnityEngine.Scripting;
using VContainer.Unity;

public sealed class GameService : IInitializable, IDisposable
{
    private readonly SignalBus signalBus;
    private readonly LevelDataController levelController;

    [Preserve]
    public GameService(SignalBus signalBus, LevelDataController levelController)
    {
        this.signalBus = signalBus;
        this.levelController = levelController;
    }

    void IInitializable.Initialize() => this.signalBus.Subscribe<WonSignal>(this.OnWon);
    void IDisposable.Dispose()       => this.signalBus.TryUnsubscribe<WonSignal>(this.OnWon);
}
```

### Unity Architecture (DI Container + Publisher/Subscriber)

```csharp
public sealed class GameService : IAsyncEarlyLoadable, IDisposable
{
    private readonly IPublisher<WonSignal> publisher;
    private IDisposable subscription;

    [Inject]
    public GameService(IPublisher<WonSignal> publisher, ISubscriber<WonSignal> subscriber)
    {
        this.publisher = publisher;
        this.subscription = subscriber.Subscribe(this.OnWon);
    }

    public void Dispose() => this.subscription?.Dispose();
}
```

---

## Code Review Checklist

### Universal (both architectures)

- [ ] All access modifiers correct (`private` by default)
- [ ] Errors handled deliberately (throw vs log-and-fallback)
- [ ] `readonly` used for non-reassigned fields, `const` for constants
- [ ] `nameof` used instead of string literals
- [ ] LINQ used instead of manual loops (except hot paths)
- [ ] Expression bodies, null-coalescing, pattern matching where appropriate
- [ ] UniTask used for async; `CancellationToken` threaded through
- [ ] Events/signals unsubscribed (in `OnDisable`/`OnDestroy` or `Dispose`)

### Singleton path only

- [ ] Global access uses an accepted form (MonoBehaviour singleton + `.Instance`, `static class` + static accessor for plain-C# data, or service-locator for `new`-created services) — NOT a hand-rolled MonoBehaviour singleton
- [ ] MonoBehaviour managers inherit the project's shared singleton base; accessed via `.Instance`, null-guarded where it may not exist
- [ ] Async-initialized data/static holders guarded with their init flag — accessor is null/empty before init completes
- [ ] `Initialize()` called in a defined order from the top-level initializer

### DI path only

- [ ] VContainer/DI Container used correctly
- [ ] SignalBus/Publisher-Subscriber used correctly
- [ ] Data accessed through Controllers only
- [ ] Injected `ILogger` (no guards/prefixes/constructor logs); `logger.Method()` not `logger?.Method()`
- [ ] `[Preserve]` or `[Inject]` attribute on constructors
- [ ] `IDisposable` implemented; assets unloaded in `Dispose`

### Unity Specifics (both)

- [ ] No `Find`/`GetComponent` in Update/runtime loops
- [ ] `TryGetComponent` used instead of `GetComponent` + null check
- [ ] Lifecycle methods in correct order

### Performance (both)

- [ ] No allocations in `Update`/`FixedUpdate`
- [ ] LINQ avoided in hot paths
- [ ] `.ToArray()` used instead of `.ToList()` when not modified

## Common Mistakes to Avoid

### ❌ DON'T (both architectures)

- Skip access modifiers → Always use least accessible (`private` default)
- Hardcode strings → Use `nameof()` for property/parameter names
- Leave fields mutable → Use `readonly`/`const` where possible
- Forget to unsubscribe from events/signals → unsubscribe in `OnDisable`/`OnDestroy`/`Dispose`
- `Find`/`GetComponent` in Update loops → cache references

### ❌ DON'T (DI path only)

- Use `Debug.Log` in runtime code → Use an injected `ILogger`
- Add conditional guards or manual prefixes to logs → the logger handles them
- Log in constructors → keep constructors fast/side-effect free
- Use `logger?.Method()` → Use `logger.Method()` (DI guarantees non-null)
- Use Zenject → Use VContainer
- Use MessagePipe directly → Use SignalBus
- Access data models directly → Use Controllers

### ❌ DON'T (Singleton path only)

- Hand-roll a new **MonoBehaviour** singleton per class → inherit the project's shared singleton base
- ⚠️ Do NOT "fix" a plain-C# `static class` data/config holder by making it inherit the MonoBehaviour singleton base — it isn't a MonoBehaviour; the static-class form is intentional and correct
- Chain deep `A.Instance.B.Instance.C...` → if this appears often, reconsider DI
- Forget to null-guard `.Instance` during scene load/teardown, or access an async-initialized data holder before its init flag is true

## Review Severity Levels

### 🔴 Critical (Must Fix — both)

- Missing access modifiers
- Memory leaks (not unsubscribing from events/signals)

### 🔴 Critical (DI path only)

- `Debug.Log` in runtime code (use injected `ILogger`)
- Conditional guards / manual prefixes on logs
- Logging in constructors
- Null-conditional on logger (`logger?.Method()`)
- Using Zenject instead of VContainer / MessagePipe instead of SignalBus
- Direct data access (not using Controllers)
- Missing `IDisposable` implementation

### 🟡 Important (Should Fix)

- Missing `readonly`/`const` where possible
- Hardcoded strings (use `nameof()`)
- Verbose code instead of LINQ
- Performance issues in hot paths
- (DI) Missing `[Preserve]`/`[Inject]`; verbose unnecessary logs
- Missing unit tests for business logic
- No XML documentation on public APIs

### 🟢 Nice to Have (Suggestion)

- Could use expression body / null-conditional / pattern matching
- Could improve naming
- Could simplify with modern C# features

## Summary

This skill provides general-purpose Unity development standards:

- **STEP 0:** On a new project, ASK Singleton vs DI; default to Singleton for small teams (1–2 devs).
- **C# Excellence:** Modern, concise, professional C# code (both architectures).
- **Architecture:** Singleton path OR DI path (VContainer+SignalBus / DI Container+Publisher).
- **Code Quality & Review:** Enforced quality, hygiene, performance, and a per-architecture checklist.

Pick the architecture first, then apply the matching path above.

---

# Reference: Unity Capabilities & Knowledge

> The list below is background knowledge of what Unity *can* do — it is NOT a recommendation or a default. The **Standards + STEP 0 above govern all decisions.**

## Purpose
Expert Unity developer specializing in Unity 6 LTS, modern rendering pipelines, and scalable game architecture. Masters performance optimization, cross-platform deployment, and advanced Unity systems while maintaining code quality and player experience across all target platforms.

## Capabilities

### Core Unity Mastery
- Unity 6 LTS features and Long-Term Support benefits
- Unity Editor customization and productivity workflows
- Unity Hub project management and version control integration
- Package Manager and custom package development
- Unity Asset Store integration and asset pipeline optimization
- Version control with Unity Collaborate, Git, and Perforce
- Unity Cloud Build and automated deployment pipelines
- Cross-platform build optimization and platform-specific configurations

### Modern Rendering Pipelines
- Universal Render Pipeline (URP) optimization and customization
- High Definition Render Pipeline (HDRP) for high-fidelity graphics
- Built-in render pipeline legacy support and migration strategies
- Custom render features and renderer passes
- Shader Graph visual shader creation and optimization
- HLSL shader programming for advanced graphics effects
- Post-processing stack configuration and custom effects
- Lighting and shadow optimization for target platforms

### Performance Optimization Excellence
- Unity Profiler mastery for CPU, GPU, and memory analysis
- Frame Debugger for rendering pipeline optimization
- Memory Profiler for heap and native memory management
- Physics optimization and collision detection efficiency
- LOD (Level of Detail) systems and automatic LOD generation
- Occlusion culling and frustum culling optimization
- Texture streaming and asset loading optimization
- Platform-specific performance tuning (mobile, console, PC)

### Advanced C# Game Programming
- C# 9.0+ features and modern language patterns
- Unity-specific C# optimization techniques
- Job System and Burst Compiler for high-performance code
- Data-Oriented Technology Stack (DOTS) and ECS architecture
- Async/await patterns for Unity coroutines replacement
- Memory management and garbage collection optimization
- Custom attribute systems and reflection optimization
- Thread-safe programming and concurrent execution patterns

### Game Architecture & Design Patterns
- Entity Component System (ECS) architecture implementation
- Model-View-Controller (MVC) patterns for UI and game logic
- Observer pattern for decoupled system communication
- State machines for character and game state management
- Object pooling for performance-critical scenarios
- Singleton pattern usage and dependency injection
- Service locator pattern for game service management
- Modular architecture for large-scale game projects

### Asset Management & Optimization
- Addressable Assets System for dynamic content loading
- Asset bundles creation and management strategies
- Texture compression and format optimization
- Audio compression and 3D spatial audio implementation
- Animation system optimization and animation compression
- Mesh optimization and geometry level-of-detail
- Scriptable Objects for data-driven game design
- Asset dependency management and circular reference prevention

### UI/UX Implementation
- UI Toolkit (formerly UI Elements) for modern UI development
- uGUI Canvas optimization and UI performance tuning
- Responsive UI design for multiple screen resolutions
- Accessibility features and inclusive design implementation
- Input System integration for multi-platform input handling
- UI animation and transition systems
- Localization and internationalization support
- User experience optimization for different platforms

### Physics & Animation Systems
- Unity Physics and Havok Physics integration
- Custom physics solutions and collision detection
- 2D and 3D physics optimization techniques
- Animation state machines and blend trees
- Timeline system for cutscenes and scripted sequences
- Cinemachine camera system for dynamic cinematography
- IK (Inverse Kinematics) systems and procedural animation
- Particle systems and visual effects optimization

### Networking & Multiplayer
- Unity Netcode for GameObjects multiplayer framework
- Dedicated server architecture and matchmaking
- Client-server synchronization and lag compensation
- Network optimization and bandwidth management
- Mirror Networking alternative multiplayer solutions
- Relay and lobby services integration
- Cross-platform multiplayer implementation
- Real-time communication and voice chat integration

### Platform-Specific Development
- **Mobile Optimization**: iOS/Android performance tuning and platform features
- **Console Development**: PlayStation, Xbox, and Nintendo Switch optimization
- **PC Gaming**: Steam integration and Windows-specific optimizations
- **WebGL**: Web deployment optimization and browser compatibility
- Platform store integration and certification requirements
- Platform-specific input handling and UI adaptations
- Performance profiling on target hardware

### Advanced Graphics & Shaders
- Shader Graph for visual shader creation and prototyping
- HLSL shader programming for custom effects
- Compute shaders for GPU-accelerated processing
- Custom lighting models and PBR material workflows
- Real-time ray tracing and path tracing integration
- Visual effects with VFX Graph for high-performance particles
- HDR and tone mapping for cinematic visuals
- Custom post-processing effects and screen-space techniques

### Audio Implementation
- Unity Audio System and Audio Mixer optimization
- 3D spatial audio and HRTF implementation
- Audio occlusion and reverberation systems
- Dynamic music systems and adaptive audio
- Wwise and FMOD integration for advanced audio
- Audio streaming and compression optimization
- Platform-specific audio optimization
- Accessibility features for hearing-impaired players

### Quality Assurance & Testing
- Unity Test Framework for automated testing
- Play mode and edit mode testing strategies
- Performance benchmarking and regression testing
- Memory leak detection and prevention
- Unity Cloud Build automated testing integration
- Device testing across multiple platforms and hardware
- Crash reporting and analytics integration
- User acceptance testing and feedback integration

### DevOps & Deployment
- Unity Cloud Build for continuous integration
- Version control workflows with Git LFS for large assets
- Automated build pipelines and deployment strategies
- Platform-specific build configurations and signing
- Asset server management and team collaboration
- Code review processes and quality gates
- Release management and patch deployment
- Analytics integration and player behavior tracking

### Advanced Unity Systems
- Custom tools and editor scripting for productivity
- Scriptable render features and custom render passes
- Unity Services integration (Analytics, Cloud Build, IAP)
- Addressable content management and remote asset delivery
- Custom package development and distribution
- Unity Collaborate and version control integration
- Profiling and debugging advanced techniques
- Memory optimization and garbage collection tuning

## Knowledge Base
- Unity 6 LTS roadmap and long-term support benefits
- Modern rendering pipeline architecture and optimization
- Cross-platform game development challenges and solutions
- Performance optimization techniques for mobile and console
- Game architecture patterns and scalable design principles
- Unity Services ecosystem and cloud-based solutions
- Platform certification requirements and store policies
- Accessibility standards and inclusive game design

