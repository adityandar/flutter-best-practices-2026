# Flutter Architecture: Clean Architecture Best Practices 2025–2026

> Comprehensive notes from research + discussion. Covers the current architecture consensus **and** clarifications on pain points that often come up during implementation (BLoC vs MVVM, where the local-vs-cloud decision lives, when to use a use case, who converts the model, etc.).

**Primary sources:**

1. [Flutter Docs – Guide to App Architecture](https://docs.flutter.dev/app-architecture/guide) (official, MVVM)
2. [Flutter Docs – Case Study: UI Layer](https://docs.flutter.dev/app-architecture/case-study/ui-layer) & [Data Layer](https://docs.flutter.dev/app-architecture/case-study/data-layer) (Compass app)
3. [Very Good Ventures – Feature-First Clean Architecture (FFCA)](https://verygood.ventures/blog/feature-first-clean-architecture/) + [official specification](https://engineering.verygood.ventures/architecture/ffca/overview/)
4. [Tech With Sam – Clean Architecture in Flutter 2026](https://techwithsam.dev/articles/clean-architecture-in-flutter-2026-practical-implementation-guide)

---

## 1. The 2025–2026 Consensus

The trend has moved away from "classic" clean architecture (Reso Coder style: `entities`/`use_cases`/`repositories`/`datasources` for **every** feature). The direction now:

1. **Flutter's official MVVM** — the components are simply: **View, ViewModel, Repository, Service**.
2. **The domain layer (use cases) is optional** — added only when genuinely needed, not by default. The biggest criticism of classic clean architecture: excessive boilerplate.
3. **Feature-first beats layer-first** — for any project beyond a few screens.
4. For monorepos / multi-app / many teams / AI-assisted development: **Feature-First Clean Architecture (FFCA)** — each feature becomes a separate package, layer boundaries enforced by the compiler.

---

## 2. Layering & Dependency Rules

```
View → ViewModel → (Use Case?) → Repository → Service
──────────────────────────────────────────────────────
  UI layer            Domain layer       Data layer
```

| Layer | Contents | Responsibility | Dependencies |
|---|---|---|---|
| **Presentation (UI)** | Views (widgets), ViewModels, state management | Render UI, capture user input, manage UI state | Knows domain & repository; doesn't know data impl |
| **Domain (optional)** | Entities, repository contracts, use cases | Core business rules, pure Dart | Knows nothing (innermost) |
| **Data** | Models/DTOs, Services, repository implementations | Fetch data from API/DB/file, transform into domain models | Knows domain (implements its contracts) |

**Golden rules:**

- Domain = pure Dart, no Flutter imports, knows nothing about the data layer. Data depends on domain (inversion), never the reverse.
- **Repositories must never know each other.** Need data from 2 repositories? Combine it in the ViewModel or a use case.
- Views must contain no business logic — only `if` to show/hide widgets, animation, layout, simple routing.
- Dependencies always point inward.

---

## 3. Folder Structure

### 3.1 Single app — feature-first + MVVM (most common)

```
lib/
├── core/                  # Never depends on features
│   ├── di/                # Dependency injection (Riverpod/GetIt)
│   ├── network/           # Dio client, interceptors, error handler
│   ├── theme/
│   ├── router/
│   └── utils/
│
├── features/
│   ├── auth/
│   │   ├── presentation/
│   │   │   ├── screens/       # Views (widgets)
│   │   │   ├── widgets/
│   │   │   └── view_models/   # ViewModels (logic + UI state)
│   │   ├── domain/
│   │   │   ├── models/            # entities, pure Dart
│   │   │   ├── repositories/      # abstract class (contract)
│   │   │   └── use_cases/         # optional
│   │   └── data/
│   │       ├── models/            # DTOs per data source
│   │       ├── mappers/           # DTO ⇄ domain converters
│   │       ├── services/          # 1 per data source
│   │       └── repositories/      # contract implementations
│   └── notes/
│       └── ...
│
└── shared/                # Widgets/models shared across features
```

Principle: **feature at the top level, layers inside the feature**. Advantage over layer-first (`lib/presentation`, `lib/domain`, `lib/data` global): features are self-contained, easy to delete/move, fewer cross-team merge conflicts, and every file has one clear "home" (important for AI coding agents).

### 3.2 Monorepo / multi-app / many teams — FFCA (Very Good Ventures)

```
features/
  cart/
    cart_domain/         # pure Dart: models, repository interfaces, use cases
    cart_data/           # DTOs, services, repository impls
    cart_presentation/   # screens, blocs/viewmodels, Module
  auth/                  # "headless" feature: only domain + data
apps/
  mobile_app/            # composes features + shared into an app
  admin_app/
shared/
  ui_kit/
  api_client/
```

- `cart_presentation` may only depend on `cart_domain` — **never** on `cart_data`. A cross-layer import is a compile error.
- Cross-feature navigation via **callback injection**: a feature exposes a `Module` with callbacks; the app layer wires them to routes (`go_router`). A feature knows neither the router nor other features.
- Bonus: a feature can be deferred-loaded per screen (web chunking / Android dynamic feature modules).
- **When to use FFCA:** features need to be composed/shared/removed across apps, or coding agents are part of the workflow. A regular single app → feature-first folders are enough.

---

## 4. Responsibilities of Each Component

### View
- A collection of widgets for one feature (usually one screen, but not always).
- Receives all data from the ViewModel via state, forwards user events as *commands*.
- View ↔ ViewModel relationship = **1:1**.
- May only contain: `if` to show/hide widgets, animation, layout logic, simple routing.

### ViewModel
Where most app logic lives:
- Fetch data from repositories → transform (filter, sort, aggregate) into UI state.
- Hold screen state (loading flags, active section, etc.).
- Expose **commands** (functions the view calls on button taps).

### Repository
- *Single source of truth* for data. **One repository per data type.**
- Caching, error handling, retry, polling, refresh.
- Outputs **domain models** (or `Stream<Model>`).
- Repository ↔ ViewModel relationship = **many-to-many**.
- The ideal home for **app-wide session state** (user login, in-memory cache) since the instance is shared via DI.
- **Must never know each other**.

### Service
- The lowest layer. **One class per data source** (REST, local storage, platform API).
- Wraps endpoints, returns `Future`/`Stream`, **holds no state**.
- Called "datasource" in older tutorials.

### Use Case (optional)
Added ONLY if it meets one of the criteria (see §5.6):
1. Needs to merge data from multiple repositories
2. Very complex logic
3. Logic reused by many ViewModels

> Forcing a use case for every feature = over-engineering. Start without a domain layer, refactor later when you find duplication.

---

## 5. Clarifications & Pain Points (Q&A)

### 5.1 From BLoC's point of view, a ViewModel = Event + State?

Partly true, but incomplete. The mapping:

| MVVM | BLoC |
|---|---|
| **ViewModel** | **The Bloc/Cubit class itself** (not just events & state) |
| Command | **Event** via `add()` — in Cubit: a method directly |
| UI state | **State** that is emitted |
| Data transformation logic | **Event handler** inside the Bloc (`on<Event>` / Cubit method) |

More precisely: **ViewModel = Bloc + Event + State as one unit.**

- **Event** = input side (equivalent to a command). In **Cubit** this is literally a method — the most literal match to an MVVM command.
- **State** = output side (equivalent to UI state).
- What's often forgotten: **the logic lives in the Bloc itself** (event handler), not inside the Event or State classes. Events & States are just DTOs/contracts.

```dart
// Cubit = the most literal ViewModel
class NotesCubit extends Cubit<NotesState> {
  Future<void> deleteNote(String id) async {   // ← command = method
    emit(NotesLoading());
    await _repository.deleteNote(id);
    emit(NotesLoaded(notes));
  }
}
```

**Practical takeaways:**
- **Cubit** = the most literal ViewModel (command = method).
- **Bloc** = commands formalized into Event classes; more verbose but stricter per-event traceability & testability.
- The rule stays the same: widgets only render state + trigger events/commands.

### 5.2 What state management does MVVM use?

**MVVM is a pattern, not a library.** Each reference uses a different library:

| Source | State management |
|---|---|
| Official Flutter Docs (Compass app) | **`ChangeNotifier` + `ListenableBuilder`** — built into the SDK, no third-party package. Immutability via `freezed` |
| Tech With Sam (2026) | **Riverpod 3** — `@riverpod` classes as ViewModels |
| Very Good Ventures | **flutter_bloc** — Bloc/Cubit as the ViewModel |

The official guide is explicitly neutral: *"You can also use a robust third-party state management solution, such as `riverpod`, `flutter_bloc`, or `signals`."*

| MVVM concept | ChangeNotifier (official) | Riverpod | BLoC |
|---|---|---|---|
| ViewModel | `class HomeViewModel extends ChangeNotifier` | `@riverpod class HomeViewModel extends _$HomeViewModel` | `class HomeBloc extends Bloc<Event, State>` |
| UI state | public field + getter | `build()` return value | `State` class |
| Command | method + `notifyListeners()` | method + `ref.invalidateSelf()` | `add(Event)` / Cubit method |
| View listens via | `ListenableBuilder` | `ref.watch(provider)` | `BlocBuilder` |

### 5.3 If I use BLoC/Cubit, am I using MVVM?

**Yes, architecturally equivalent** — Bloc/Cubit plays the ViewModel role. The official Flutter guide names flutter_bloc as one way to implement MVVM.

But two caveats:

1. **Equivalent ≠ identical terminology.** BLoC is its own named pattern (Business Logic Component) with an Event → State vocabulary. MVVM is a more general pattern with a Command → UI State vocabulary. The position is the same: between View and Repository.
2. **"Using flutter_bloc" ≠ automatically "doing MVVM".** What matters is how you use it. It's still not MVVM if:
   - Widgets call repositories/services directly (Bloc is just bolted on)
   - Business logic still piles up in widgets
   - The Bloc is too thin — just a pass-through without UI-state transformation

> **Formula:** Bloc/Cubit library + correct placement (View only renders & triggers, all logic in the Bloc, data through the Repository) = MVVM in BLoC terminology.

### 5.4 Data can come from local OR cloud — where does the decision live?

**In the Repository** (its implementation, in the data layer). Not in the Service, not in the ViewModel, not in the View.

Reason (from the official guide): the Repository is the *single source of truth*, and its explicit responsibilities include: **caching, error handling, retry logic, refreshing data, polling**.

```dart
// data/repositories/notes_repository_impl.dart
class NotesRepositoryImpl implements NotesRepository {
  final NotesRemoteService _remote;
  final NotesLocalService _local;

  @override
  Stream<List<Note>> watchNotes() async* {
    // 1. Serve cache first (fast, offline-friendly)
    final cached = await _local.getCachedNotes();
    yield cached.map(Note.fromMap).toList();

    // 2. Fetch fresh data from the cloud
    final fresh = await _remote.fetchNotes();

    // 3. Update the cache, then emit the latest data
    await _local.cacheNotes(fresh);
    yield fresh.map(Note.fromMap).toList();
  }
}
```

| Strategy | Logic in the Repository |
|---|---|
| Cache-first (offline-first) | Check local → serve → sync cloud in background |
| Network-first | Fetch cloud → write local → serve; fallback to local on failure |
| Cache-then-network | Serve cache → refresh cloud → update cache |
| Remote only | Call `_remote` directly, never touch `_local` |

**Who must NOT know:**
- **Service** must not decide — it only knows how to fetch from one source, stateless.
- **ViewModel** uses the abstract contract — doesn't know or care whether data came from local or cloud.
- **Domain layer** stays pure — strategy decisions are data-layer implementation details.

> The local-vs-cloud decision is **not** a trigger for a use case. It's an inherent repository responsibility.

### 5.5 Does a Service need an abstraction (interface)?

**No.** The references are consistent: Service = the lowest concrete class, no service interface.

- The abstraction boundary is the **Repository interface** (domain), not the Service.
- The Service is an internal implementation detail of the data layer — never used directly by the ViewModel/domain, so no external consumer needs a contract.
- Testing: concrete classes can be mocked with `mockito`/`mocktail`, so testability isn't a reason.

| Component | Interface? |
|---|---|
| Repository (domain) | **Yes** — required, the dependency-inversion point |
| Service | No — concrete, internal to the data layer |
| Use case | No — a concrete class |

> **Practical rule:** create abstractions only at points crossed across layers. The Service is not crossed across layers (Repository → Service, both in the data layer).

### 5.6 When is a use case used?

**Optional**, added only if it meets **one or more** of 3 criteria (from the official docs):

1. **Needs to merge data from multiple repositories** — the most common trigger.
2. **Very complex logic** — layered business rules, multi-step validation.
3. **Logic reused by many ViewModels** — the strongest signal: you've found duplication.

**Two philosophies, one correct:**

| Approach | Description | Official verdict |
|---|---|---|
| **Add-as-needed** (recommended) | Start without use cases; add when you see duplication/complexity | ✅ *"A good approach"* |
| Use case for every data access | Every ViewModel needing data must go through a use case, even pass-throughs | ❌ *"Adds a significant amount of unnecessary overhead"* |

In the official case study, some features use use cases while some ViewModels access repositories directly — mixing is legitimate.

**Pros & cons (official table):**

| Pros | Cons |
|---|---|
| ✅ Avoids duplication in ViewModels | ❌ Adds complexity & cognitive load |
| ✅ Better testability | ❌ Testing requires extra mocks |
| ✅ More readable ViewModels | ❌ Adds boilerplate |

**Rules when using use cases:**
- Use cases depend on **repositories** (never the reverse)
- Use case ↔ repository = many-to-many
- A ViewModel may depend on a use case **and** a repository at the same time

**Practical "it's time" signals:**
1. Two ViewModels have similar methods → extract into a use case
2. A ViewModel starts filling with business calculations unrelated to UI
3. One ViewModel injects 3+ repositories and orchestrates them all
4. The same logic is copy-pasted into a new feature

**Concrete example (trigger #1 + #3 — merges 3 repositories, used by 2 screens):**

```dart
// BEFORE — logic piles up & is duplicated across 2 ViewModels ❌
class DashboardViewModel {
  // 40 lines merging EmployeeRepository + AttendanceRepository + LeaveRepository
}

// AFTER — use case, both ViewModels become thin ✅
// domain/use_cases/get_today_team_status.dart
class GetTodayTeamStatus {
  GetTodayTeamStatus(this._employees, this._attendance, this._leaves);
  final EmployeeRepository _employees;      // abstract
  final AttendanceRepository _attendance;   // abstract
  final LeaveRepository _leaves;            // abstract

  Future<List<TeamMemberStatus>> call(String managerId) async {
    final team = await _employees.getTeam(managerId);
    final ids = team.map((e) => e.id).toList();
    final today = await _attendance.getTodayRecords(ids);
    final leaves = await _leaves.getTodayLeaves(ids);
    // ... mapping & status rules here
  }
}
```

**Example of trigger #2 (complex logic):**

```dart
// domain/use_cases/calculate_overtime_pay.dart
class CalculateOvertimePay {
  // Rules: overtime after 17:00, min 1 hour, 1.5x, >3 hours & weekend 2x,
  // needs salary + attendance data — too heavy for a ViewModel
}
```

**When NOT to use a use case** — a ViewModel reading one repository directly, no merge/complexity/reuse:

```dart
class AttendanceViewModel {
  AttendanceViewModel(this._attendance);
  final AttendanceRepository _attendance;   // ← direct, WITHOUT a use case ✅
}
```

### 5.7 A use case is domain; is the injected repository also abstract?

**Yes.** Use cases live in the domain layer (pure), so the injected repository is the **abstract contract** (`domain/repositories/`), not the impl.

```dart
final EmployeeRepository _employees;   // ✅ abstract
// ❌ Not EmployeeRepositoryImpl — that's in the data layer
```

- The use case itself is a **concrete class** (no interface of its own) — what's abstract is its *dependencies*.
- The concrete impl enters via **DI** — the only place that knows the concrete impl:

```dart
// core/di/
@Riverpod(keepAlive: true)
GetTodayTeamStatus getTodayTeamStatus(Ref ref) => GetTodayTeamStatus(
      ref.watch(employeeRepository),
      ref.watch(attendanceRepository),
      ref.watch(leaveRepository),
    );
```

> **Full abstraction chain:** ViewModel → Use Case → Repository contract → (DI connects) → RepositoryImpl → Services. The domain stays pure.

### 5.8 Can the Repository be skipped if it just pass-throughs to the Service?

**Not recommended — the references actually suggest keeping the repository, even for methods that look like pass-throughs.**

An "empty repository" isn't truly empty. Official Compass app example:

```dart
Future<Result<void>> delete(int id) async {
  try {
    return _apiClient.deleteBooking(id);   // ← indeed just calls the service
  } on Exception catch (e) {
    return Result.error(e);                // ← but this is the repository's job
  }
}
```

At least 2 jobs **always** live in the repository:

1. **API model → domain model transformation** — *"raw data that often needs to be filtered, combined, or deleted... The repo refines the raw data and outputs it as domain models."* If it feels like "just calling the service", the transformation usually hasn't been done in the right place yet.
2. **Error mapping** — without a repository, try/catch spreads into ViewModels.

Explicit rule violated when skipped (official case study):

> *"It's important that the service is a private member, so that the UI layer can't bypass the repository and call a service directly."*

If the ViewModel goes straight to the Service:
- The UI layer binds to the raw API data format — when the backend changes, widgets/ViewModels change too
- You lose the only legitimate place to mutate that data type
- Tomorrow, needing cache/retry/offline sync → refactor from scratch

> **What the references allow: a THIN repository.** The anti-pattern isn't "a repository with one line", it's "no repository at all". Honest note: the official guide says *"guidelines, not steadfast rules"* — for a demo app, ViewModel → Service can work, but it deviates from the recommendation and loses the data boundary. **What's optional is the use case, not the repository.**

### 5.9 Model / Entity / Mapper — best practices

**Core concept — 2 (sometimes 3) kinds of data objects:**

| Object | Where | Contents | Who may see it |
|---|---|---|---|
| **Domain model** (entity) | `domain/models/` | Pure data in business language | All layers |
| **DTO** | `data/models/` | Raw shape from the data source (JSON API, Drift row) | ONLY the data layer |
| **UI state object** (optional) | `presentation/` | Snapshot to render one view | Its view + viewmodel |

**Domain model (entity):**
- **Do NOT add a `Model`/`Entity` suffix** (official VGV convention): *"Do not add the word 'Model' or 'Entity' to this class."* → just `Product`, `Cart`, `Employee`.
- **Immutable** — the official guide uses `freezed` (deep immutability + copyWith + toJson).
- **No serialization on the entity** — `fromJson`/`toJson`/`fromMap` belong to the DTO.

**DTO:**
- Owned by the data layer, **must not be exported/leaked** to domain/presentation. The data-layer barrel may export data sources, **not** DTOs & mappers.
- `ApiProduct` and `DbProduct` are two separate classes, each representing its source.

**Don't create a shared abstract DTO** (in-memory/local/remote):

> *"For now, we do not believe it makes sense to define an abstract DTO for in-memory, local, and remote storage options. In general, it adds extra, unnecessary mapping."*

Meaning: if each data source returns a different shape, **use each source's own DTO**, with no shared interface/base class. The unification point is the **domain model**, not the DTO:

```
ApiProduct ──[ApiToDomainConverter]──┐
                                      ├──> Product  ← the single shared shape
DbProduct ──[DbToDomainConverter]────┘
```

Why: DTOs are usually code-generated (Drift/Swagger/json_serializable) and tightly coupled to their source; an abstract DTO = an extra mapping layer with no benefit; the only shared contract that needs abstraction is the repository interface.

**Mapper — 3 options VGV recognizes:**

```dart
// Option A — extension method (simplest)
extension ApiProductX on ApiProduct {
  Product toDomain() => Product(id: id, title: title, price: priceInCents / 100);
}

// Option B — Converter class (favored by VGV, testable)
class DbToDomainProductConverter extends Converter<DbProduct, Product> {
  const DbToDomainProductConverter();
  @override
  Product convert(DbProduct dbProduct) => Product(
        id: dbProduct.id,
        price: dbProduct.priceInCents / 100,  // DB cents → domain currency
      );
}

// Option C — code generation: auto_mappr + build_runner
```

**Mapper rules:**
1. **One converter per data source per direction** — a feature reading DB + API has `DbToDomainX` **and** `ApiToDomainX`.
2. **Unit/format conversion lives in the mapper, not the Bloc** — *"The database stores cents as an integer... the conversion belongs here, not in a Bloc."*
3. The repository applies the converter as data enters/leaves the source.

### 5.10 The "Summary" pattern (VGV) is a use case, right?

**Yes, exactly.** VGV just uses different terminology:

| Official Flutter guide | VGV FFCA | Role |
|---|---|---|
| Use case (interactor) | **Command** | Action modifying data (`UpdateProductTitleCommand`) |
| Use case (interactor) | **Query** | Reading data (`GetProductByIdQuery`, `WatchProductByIdQuery`) |

> *"Combined, these are often referred to as 'use cases.' For the purposes of our codebase, we recommend avoiding the term 'use case' and instead prefer Command and Query classes for clarity."*

The creation condition is the same: *"If a Command or Query only serves to call out to the repository, there is no need to introduce the class at all."*

**Summary pattern for cross-feature composition:** don't store a full `Product` object inside `Cart` — store its IDs, then a Query that combines them:

```dart
// domain: CartSummary { id, productIds } — not Cart { List<Product> }

class GetCartByIdQuery {
  Future<Cart> get(String cartId) async {
    final summary = await _cartsRepository.getCartById(cartId);
    return Cart(
      id: summary.id,
      products: [
        for (final pid in summary.productIds)
          await _productsRepository.getProductById(pid),
      ],
    );
  }
}
```

The cart feature never learns how Products are stored; the product feature never learns carts exist. This is use-case trigger #1 (merge multi-repository) in real form.

### 5.11 Repository: are inputs & outputs domain models? Who converts?

**Both are correct:**

1. **Repository contract (domain): inputs & outputs = domain models.**
2. **Domain → DTO conversion happens in the repository implementation, NOT the service.**

Evidence from the official case study:

```dart
// Service (ApiClient) — speaks the RAW language of the data source:
Future<Result<BookingApiModel>> postBooking(BookingApiModel booking) async {...}

// Repository contract (domain) — domain language:
abstract class BookingRepository {
  Future<Result<void>> createBooking(Booking booking);
}

// Repository impl — the conversion point:
class BookingRepositoryRemote implements BookingRepository {
  @override
  Future<Result<void>> createBooking(Booking booking) async {
    final apiModel = booking.toApiModel();   // ← domain → DTO HERE
    return _apiClient.postBooking(apiModel);
  }
}
```

**Why not in the Service?**
1. The Service is a dumb endpoint wrapper (*"its only job is to wrap an external API"*). If it knew domain models, it would no longer be pure and would be contaminated by business logic every time the backend changes.
2. Mapping is owned by the data layer, and **the repository applies it**. Converters go both directions:

```dart
// data/mappers/domain_to_db_product_converter.dart
class DomainToDbProductConverter extends Converter<Product, DbProduct> {
  @override
  DbProduct convert(Product product) => DbProduct(
        id: product.id,
        priceInCents: (product.price * 100).round(),   // currency → cents
      );
}
```

**Full flow:**

```
READ:  API/DB → Service (DTO) → Repo impl [DTO → Domain] → ViewModel
WRITE: ViewModel → Repo impl [Domain → DTO] → Service (DTO) → API/DB
                        └──────── translator boundary ────────┘
```

> Domain models never leave domain+presentation; DTOs never enter them. All two-way translation is concentrated in the repository impl + mappers.

### 5.12 Can layer boundaries (domain ≠ import data) be enforced without separate packages?

**Yes** — via a linter/analyzer plugin. Violations are caught live in the IDE and fail `dart analyze` in CI, without splitting the code into separate packages.

**The tools:**

| Tool | How | Notes |
|---|---|---|
| **[`dart_boundaries`](https://pub.dev/packages/dart_boundaries)** | YAML config in `analysis_options.yaml`, built on `custom_lint` | The best fit; has `layer_boundaries` & `no_banned_imports` rules |
| **[`import_lint`](https://pub.dev/packages/import_lint)** | Similar, define layers via YAML | Older, popular |
| **DCM (`dart_code_metrics`)** | `avoid-banned-imports` rule | If you already use DCM |
| **Write your own** with `custom_lint` | Custom rule | Flexible but more work |

**Quick approach — `no_banned_imports`:**

```yaml
# analysis_options.yaml
dart_boundaries:
  no_banned_imports:
    entries:
      - paths: ['lib/features/**/domain/**']
        deny: ['lib/features/**/data/**']
        message: 'Domain layer must not import the data layer.'
```

**Fuller approach — `layer_boundaries` (define each layer's dependency direction):**

```yaml
dart_boundaries:
  layer_boundaries:
    elements:
      - type: domain
        pattern: 'lib/features/{{ feature }}/domain/**'
      - type: data
        pattern: 'lib/features/{{ feature }}/data/**'
      - type: presentation
        pattern: 'lib/features/{{ feature }}/presentation/**'
    rules:
      - from: domain
        disallow: [data]          # ← the core rule
      - from: presentation
        allow: [domain]           # presentation may only see domain
      - from: data
        allow: [domain]           # data may see domain (to implement the contract)
```

`{{ feature }}` is a capture group — files with the same feature name are treated as intra-feature (still allowed), different names as cross-feature (which can also be banned if you want).

**How it works:**
- **Live in the IDE** — violations appear as warnings/errors as you type.
- **CI** — `dart analyze` fails → PR is blocked.
- Can be `// ignore:`d when an exception is genuinely needed.

**Important limitation (honest):** a linter is a **soft boundary**, not a hard boundary.

| | Folder + linter | Separate package |
|---|---|---|
| Mechanism | `dart analyze` / CI gate | **Compile error** |
| Can a dev "write" violating code? | Can write, but fails analyze/CI | Cannot, import errors immediately |
| Relies on | Folder naming convention (glob patterns) | `pubspec.yaml` declaration |
| Scope for AI agents | Not automatic (one package) | Dependency graph limits context |

Two key points:

1. **The rule is path/glob-based**, so it's only as strong as your folder convention (`features/x/domain`, `features/x/data`). If a file lands in the wrong folder, the rule won't help.
2. **It doesn't stop compilation** — only stops merges via CI. If CI is mandatory, it's effective.

This is exactly what VGV does: they use a **custom analyzer for finer-grained rules** (naming, cross-feature data leaks, dependency cycles), **but still chose packages for the hard boundary** — because *"a layer violation is a compile error, not a review comment"*.

**Recommendation:**
- **Single app, small team, mandatory CI** → `dart_boundaries` (or `import_lint`) is enough, with no package overhead.
- **Multi-app / many teams / AI-assisted / need the strongest guarantee** → move up to separate packages (FFCA), then add the linter on top for fine-grained rules.

### 5.13 A repository needed by 2 features — how to handle it?

Don't duplicate it, don't put it in `shared/`. Make the repository its own **feature** (usually *headless*), then other features depend on its **domain**.

Two complementary VGV rules:

1. *"Feature domains may depend on other feature domains."*
2. *"Repositories for each feature must be independent. The Cart domain should not define an IProductsRepository and vice-versa."*

```
features/
├── destinations/           # headless (owns DestinationRepository)
│   ├── domain/  (contract + entity)
│   └── data/    (impl)
├── trips/                  # consumer → imports destinations/domain only
└── profile/                # second consumer → imports destinations/domain only
```

The DI wiring (impl → contract) stays in the composition root, so `trips` never learns where destinations are stored.

> One repository = one owner (feature). Cross-feature use always goes through the domain contract.

### 5.14 Cross-feature sharing — domain only

What may be imported across features = **domain** (contract + entity). `data` and `presentation` stay private. This is **specific to VGV FFCA** (the official Flutter docs don't cover the feature-package level).

**"Transitive" explained:** if `cart_data → cart_domain → product_domain`, then `product_domain` is a transitive dependency of `cart_data`. But in Dart, **imports are not transitive** — a file in `cart_data` must write its own import to use `Product`. So: *technically possible*, but **the architecture recommends against** the data layer touching another feature's domain.

> What matters isn't headless or not — it's that **only the domain is exposed across features**.

### 5.15 Backend already populates data — safe? + DTOs across features

What's forbidden = **the repository re-fetching/joining across features on the client**. Backend-populated data is **safe**, because the mapper only maps, it doesn't fetch.

```dart
// ✅ backend returns nested; mapper maps both levels, no re-fetch
class PostDto {
  final String id;
  final String title;
  final PostAuthorDto author;   // LOCAL DTO, not UserDto from users
}
```

But **cross-feature DTOs are not allowed**: `PostDto` (posts data) **must not** use `UserDto` (users data) — that's cross-feature `data → data`, violating VGV's *"must not export DTOs or mappers"*. Define a local DTO (`PostAuthorDto`), then the mapper maps it to the domain:

```dart
Post toDomain(PostDto dto) => Post(
  id: dto.id, title: dto.title,
  author: User(id: dto.author.id, name: dto.author.name), // → User (users_domain) — domain→domain ✅
);
```

**Design choice** when `Post` holds a `User`:
- **A:** `Post` directly holds `User` from `users_domain` (domain→domain).
- **B:** `Post` holds a local projection (`PostAuthor { id, name, avatarUrl }`) — looser coupling, no dependency on `users_domain`.

> Principle: a DTO is the data source's language, private per feature, never shared. Only the domain is shared across features.

### 5.16 DI placement (composition root)

Separate the **container** from the **wiring**:

| | What | Where | May depend on features? |
|---|---|---|---|
| Container | the GetIt instance | `core/di/` | ❌ |
| Wiring (registration) | bind interface → impl | **composition root** | ✅ |

`core/` and `shared/` both **must not depend on features**. Registration (which needs feature imports) lives in the **composition root**:

```dart
// lib/app/register_di.dart — the only place that knows the impls
class RegisterDi {
  Future<void> init() async {   // async because SharedPreferences.getInstance()
    final getIt = GetIt.instance;
    // core-level (generic) → can stay here
    // feature-level (wiring interface → impl) → also here
  }
}
```

```dart
// main.dart — thin bootstrap
await RegisterDi().init();
runApp(const App());
```

For larger projects: per-feature registration modules (`features/<x>/di/`) composed from `main.dart`.

> Container in `core/di`, wiring in the composition root. `app/` sits above all layers, so it's free to import.

---

## 6. Cheat Sheet — Golden Rules

| # | Rule |
|---|---|
| 1 | Feature-first: feature at the top level, layers inside the feature |
| 2 | Dependencies always point inward: View → ViewModel → Repository → Service |
| 3 | Domain is pure Dart, knows no data impl; data implements the domain contract |
| 4 | View only renders + triggers; logic in the ViewModel/Bloc/Cubit |
| 5 | Repository = local-vs-cloud decision, caching, retry, error mapping — don't skip it |
| 6 | Service is dumb, concrete, 1 per data source, stateless, no interface |
| 7 | Use case/Command/Query optional — only for multi-repo merge, complex logic, or reuse |
| 8 | Entity without suffix, immutable (freezed), no JSON knowledge |
| 9 | DTO per data source, private to the data layer, don't leak, don't abstract |
| 10 | Mapper 1 per source per direction; unit conversion in the mapper, not the Bloc |
| 11 | Repository interface = the only cross-layer contract (`I` prefix per VGV is optional) |
| 12 | Domain ⇄ DTO conversion in the repository impl, not the service |
| 13 | Repositories never know each other; combine in the ViewModel/use case |
| 14 | DI for everything: components are injected, not instantiated in widgets |
| 15 | App-wide session state (login, cache) lives in the repository via DI |
| 16 | Cross-feature sharing is domain-only (contract + entity); data & presentation stay private |
| 17 | DI: container in core, wiring in the composition root (app/register_di) |

---

## 7. References

These references back every section above — each part of this document points to them, but the content is intentionally written concisely so it isn't weighed down with citations in every paragraph.

| Reference | What it contributed |
|---|---|
| [Flutter Docs – Guide to App Architecture](https://docs.flutter.dev/app-architecture/guide) | Official layering (View/ViewModel/Repository/Service), dependency rules, 3 use-case triggers, repository & service roles, MVVM |
| [Flutter Docs – Case Study: UI Layer](https://docs.flutter.dev/app-architecture/case-study/ui-layer) | ViewModel & UI state examples (`ChangeNotifier` + `freezed`), commands, 1:1 view-viewmodel |
| [Flutter Docs – Case Study: Data Layer](https://docs.flutter.dev/app-architecture/case-study/data-layer) | Repository & service examples, private-service rule, API model vs domain model, `Result` pattern |
| [Very Good Ventures – Feature-First Clean Architecture](https://verygood.ventures/blog/feature-first-clean-architecture/) | FFCA monorepo structure, layer boundaries via packages, callback injection for navigation |
| [VGV Engineering – FFCA (Domain & Data Layer)](https://engineering.verygood.ventures/architecture/ffca/overview/) | Entity naming convention (no suffix), Command/Query, Summary pattern, DTO/mapper rules, one converter per source per direction |
| [Very Good Ventures – Very Good Layered Architecture](https://verygood.ventures/blog/very-good-flutter-architecture/) | Layer-first vs feature-first comparison, when each fits |
| [Tech With Sam – Clean Architecture in Flutter 2026](https://techwithsam.dev/articles/clean-architecture-in-flutter-2026-practical-implementation-guide) | Practical MVVM + Riverpod 3 implementation, cache-then-network example, common mistakes |
| [Mealify (FFCA reference app)](https://github.com/VGVentures/mealify_feature_first) · [Compass app (official reference)](https://github.com/flutter/samples/tree/main/compass_app) | Real-world codebases for both approaches |
