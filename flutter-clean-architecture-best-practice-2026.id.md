# Arsitektur Flutter: Clean Architecture Best Practice 2025–2026

> Catatan komprehensif dari riset + diskusi. Berisi konsensus arsitektur terkini **dan** klarifikasi atas pain point yang sering muncul saat implementasi (BLoC vs MVVM, di mana decision lokal-vs-cloud, kapan use case, siapa yang convert model, dll).

**Sumber utama:**

1. [Flutter Docs – Guide to App Architecture](https://docs.flutter.dev/app-architecture/guide) (resmi, MVVM)
2. [Flutter Docs – Case Study: UI Layer](https://docs.flutter.dev/app-architecture/case-study/ui-layer) & [Data Layer](https://docs.flutter.dev/app-architecture/case-study/data-layer) (Compass app)
3. [Very Good Ventures – Feature-First Clean Architecture (FFCA)](https://verygood.ventures/blog/feature-first-clean-architecture/) + [spesifikasi resminya](https://engineering.verygood.ventures/architecture/ffca/overview/)
4. [Tech With Sam – Clean Architecture in Flutter 2026](https://techwithsam.dev/articles/clean-architecture-in-flutter-2026-practical-implementation-guide)

---

## 1. Konsensus 2025–2026

Tren berubah dari clean architecture "klasik" (Reso Coder style: `entities`/`use_cases`/`repositories`/`datasources` untuk **semua** fitur). Sekarang arahnya:

1. **MVVM resmi Flutter** — komponennya cukup: **View, ViewModel, Repository, Service**.
2. **Domain layer (use case) jadi opsional** — hanya ditambah saat benar-benar perlu, bukan default. Kritik terbesar terhadap clean architecture klasik: boilerplate berlebihan.
3. **Feature-first menang atas layer-first** — untuk project yang melebihi beberapa screen.
4. Untuk monorepo multi-app / banyak tim / AI-assisted development: **Feature-First Clean Architecture (FFCA)** — fitur jadi paket terpisah, batas layer dipaksa compiler.

---

## 2. Layering & Aturan Dependensi

```
View → ViewModel → (Use Case?) → Repository → Service
──────────────────────────────────────────────────────
  UI layer            Domain layer       Data layer
```

| Layer | Isi | Tanggung jawab | Dependensi |
|---|---|---|---|
| **Presentation (UI)** | Views (widget), ViewModels, state management | Render UI, tangkap input user, kelola UI state | Tahu domain & repository; tidak tahu data impl |
| **Domain (opsional)** | Entities, repository contract, use cases | Aturan bisnis inti, murni Dart | Tidak tahu siapa pun (paling dalam) |
| **Data** | Models/DTO, Services, implementasi repository | Ambil data dari API/DB/file, transformasi ke domain model | Tahu domain (implementasi kontraknya) |

**Aturan emas:**

- Domain = pure Dart, tidak boleh import Flutter, tidak tahu apa-apa soal data layer. Data bergantung ke domain (inversi), bukan sebaliknya.
- **Repository tidak boleh saling kenal.** Butuh data dari 2 repository? Gabungkan di ViewModel atau use case.
- View tidak boleh berisi business logic — hanya `if` untuk show/hide widget, animasi, layout, routing sederhana.
- Dependensi selalu mengarah ke dalam.

---

## 3. Folder Structure

### 3.1 Single app — feature-first + MVVM (pola paling umum)

```
lib/
├── core/                  # Tidak pernah depend ke fitur
│   ├── di/                # Dependency injection (Riverpod/GetIt)
│   ├── network/           # Dio client, interceptor, error handler
│   ├── theme/
│   ├── router/
│   └── utils/
│
├── features/
│   ├── auth/
│   │   ├── presentation/
│   │   │   ├── screens/       # Views (widget)
│   │   │   ├── widgets/
│   │   │   └── view_models/   # ViewModel (logic + UI state)
│   │   ├── domain/
│   │   │   ├── models/            # entities, murni Dart
│   │   │   ├── repositories/      # abstract class (kontrak)
│   │   │   └── use_cases/         # opsional
│   │   └── data/
│   │       ├── models/            # DTO per sumber data
│   │       ├── mappers/           # DTO ⇄ domain converter
│   │       ├── services/          # 1 per data source
│   │       └── repositories/      # implementasi kontrak
│   └── notes/
│       └── ...
│
└── shared/                # Widget/model bersama antar fitur
```

Prinsip: **fitur di level atas, layer di dalam fitur**. Keunggulan vs layer-first (`lib/presentation`, `lib/domain`, `lib/data` global): fitur mandiri, mudah dihapus/dipindah, minim merge conflict antar tim, dan setiap file punya satu "rumah" yang jelas (penting untuk AI coding agent).

### 3.2 Monorepo / multi-app / banyak tim — FFCA (Very Good Ventures)

```
features/
  cart/
    cart_domain/         # pure Dart: models, repository interface, use cases
    cart_data/           # DTO, services, impl repository
    cart_presentation/   # screens, blocs/viewmodels, Module
  auth/                  # fitur "headless": hanya domain + data
apps/
  mobile_app/            # menyusun fitur + shared jadi app
  admin_app/
shared/
  ui_kit/
  api_client/
```

- `cart_presentation` hanya boleh depend ke `cart_domain` — **tidak pernah** ke `cart_data`. Import lintas layer = compile error.
- Navigasi antar fitur lewat **callback injection**: fitur mengekspos `Module` dengan callback, app layer yang me-wire ke route (`go_router`). Fitur tidak kenal router maupun fitur lain.
- Bonus: fitur bisa di-deferred-load per screen (web chunking / Android dynamic feature module).
- **Kapan pakai FFCA:** fitur perlu di-compose/di-share/di-remove lintas app, atau coding agent jadi bagian workflow. Single app biasa → feature-first folder cukup.

---

## 4. Pembagian Tugas Tiap Komponen

### View
- Kumpulan widget untuk satu fitur (biasanya satu screen, tidak harus).
- Menerima semua data dari ViewModel via state, meneruskan event user sebagai *command*.
- Hubungan View ↔ ViewModel = **1:1**.
- Hanya boleh berisi: `if` show/hide widget, animasi, layout logic, routing sederhana.

### ViewModel
Tempat mayoritas logika app:
- Ambil data dari repository → transformasi (filter, sort, aggregate) ke UI state.
- Simpan state screen (flag loading, section aktif, dll).
- Ekspos **commands** (fungsi yang dipanggil view saat button tap).

### Repository
- *Single source of truth* data. **Satu repository per tipe data.**
- Caching, error handling, retry, polling, refresh.
- Output berupa **domain model** (atau `Stream<Model>`).
- Hubungan Repository ↔ ViewModel = **many-to-many**.
- Tempat ideal **app-wide session state** (user login, cache in-memory) karena instance-nya shared via DI.
- **Tidak boleh saling kenal** satu sama lain.

### Service
- Lapisan terbawah. **Satu class per sumber data** (REST, local storage, platform API).
- Bungkus endpoint, return `Future`/`Stream`, **tidak menyimpan state**.
- Di tutorial lama disebut "datasource".

### Use Case (opsional)
Ditambah HANYA jika memenuhi salah satu (lihat §5.6):
1. Perlu menggabungkan data dari beberapa repository
2. Logika sangat kompleks
3. Logika dipakai ulang oleh banyak ViewModel

> Memaksakan use case untuk semua fitur = over-engineering. Mulai tanpa domain layer, refactor nanti kalau ketemu duplikasi.

---

## 5. Klarifikasi & Pain Point (Q&A)

### 5.1 ViewModel dari POV BLoC = Event + State?

Sebagian benar, tapi kurang lengkap. Mapping-nya:

| MVVM | BLoC |
|---|---|
| **ViewModel** | **Bloc/Cubit class itu sendiri** (bukan cuma event & state) |
| Command | **Event** yang di-`add()` — di Cubit: method langsung |
| UI state | **State** yang di-emit |
| Logika transformasi data | **Event handler** di dalam Bloc (`on<Event>` / method Cubit) |

Lebih tepatnya: **ViewModel = Bloc + Event + State sebagai satu kesatuan.**

- **Event** = sisi input (equivalent command). Di **Cubit** ini benar-benar method — paling literal dengan command MVVM.
- **State** = sisi output (equivalent UI state).
- Yang sering kelupaan: **logikanya ada di Bloc-nya** (event handler), bukan di dalam class Event maupun State. Event & State hanyalah DTO/kontrak.

```dart
// Cubit = ViewModel paling literal
class NotesCubit extends Cubit<NotesState> {
  Future<void> deleteNote(String id) async {   // ← command = method
    emit(NotesLoading());
    await _repository.deleteNote(id);
    emit(NotesLoaded(notes));
  }
}
```

**Kesimpulan praktis:**
- **Cubit** = ViewModel yang paling literal (command = method).
- **Bloc** = command diformalisasi jadi Event class; lebih verbose tapi traceability & testability event-per-event lebih ketat.
- Aturan tetap sama: widget hanya render state + trigger event/command.

### 5.2 MVVM itu pakai state management apa?

**MVVM itu pola, bukan library.** Tiap referensi memakai library berbeda:

| Sumber | State management |
|---|---|
| Flutter Docs resmi (Compass app) | **`ChangeNotifier` + `ListenableBuilder`** — bawaan SDK, tanpa package pihak ketiga. Immutability dibantu `freezed` |
| Tech With Sam (2026) | **Riverpod 3** — class `@riverpod` sebagai ViewModel |
| Very Good Ventures | **flutter_bloc** — Bloc/Cubit sebagai ViewModel |

Panduan resmi eksplisit netral: *"Kamu juga bisa memakai solusi pihak ketiga seperti `riverpod`, `flutter_bloc`, atau `signals`."*

| Konsep MVVM | ChangeNotifier (resmi) | Riverpod | BLoC |
|---|---|---|---|
| ViewModel | `class HomeViewModel extends ChangeNotifier` | `@riverpod class HomeViewModel extends _$HomeViewModel` | `class HomeBloc extends Bloc<Event, State>` |
| UI state | field public + getter | return value `build()` | `State` class |
| Command | method + `notifyListeners()` | method + `ref.invalidateSelf()` | `add(Event)` / method Cubit |
| View mendengarkan | `ListenableBuilder` | `ref.watch(provider)` | `BlocBuilder` |

### 5.3 Kalau aku pakai BLoC/Cubit, berarti aku pakai MVVM?

**Ya, secara arsitektur setara** — Bloc/Cubit memainkan peran ViewModel. Panduan resmi Flutter menyebut flutter_bloc sebagai salah satu cara implementasi MVVM.

Tapi 2 catatan:

1. **Setara ≠ istilah sama persis.** BLoC = pola bernama sendiri (Business Logic Component) dengan vocabulary Event → State. MVVM = pola lebih umum dengan vocabulary Command → UI State. Posisinya sama: di antara View dan Repository.
2. **"Pakai library flutter_bloc" ≠ otomatis "sudah MVVM".** Yang menentukan adalah cara pemakaiannya. Tetap bukan MVVM kalau:
   - Widget langsung memanggil repository/service (Bloc cuma tempelan)
   - Logika bisnis masih numpuk di widget
   - Bloc terlalu tipis, cuma pass-through tanpa transformasi UI state

> **Rumus:** Library Bloc/Cubit + penempatan benar (View hanya render & trigger, semua logika di Bloc, data lewat Repository) = MVVM dengan terminologi BLoC.

### 5.4 Data bisa dari lokal ATAU cloud — decision-nya di mana?

**Di Repository** (implementasinya, di data layer). Bukan di Service, bukan di ViewModel, bukan di View.

Alasan (dari panduan resmi): Repository adalah *single source of truth*, dan tanggung jawab eksplisitnya meliputi: **caching, error handling, retry logic, refreshing data, polling**.

```dart
// data/repositories/notes_repository_impl.dart
class NotesRepositoryImpl implements NotesRepository {
  final NotesRemoteService _remote;
  final NotesLocalService _local;

  @override
  Stream<List<Note>> watchNotes() async* {
    // 1. Serve cache dulu (cepat, offline-friendly)
    final cached = await _local.getCachedNotes();
    yield cached.map(Note.fromMap).toList();

    // 2. Ambil data segar dari cloud
    final fresh = await _remote.fetchNotes();

    // 3. Update cache, lalu emit data terbaru
    await _local.cacheNotes(fresh);
    yield fresh.map(Note.fromMap).toList();
  }
}
```

| Strategi | Logika di Repository |
|---|---|
| Cache-first (offline-first) | Cek lokal → serve → sync cloud di background |
| Network-first | Ambil cloud → tulis lokal → serve; fallback lokal kalau gagal |
| Cache-then-network | Serve cache → refresh cloud → update cache |
| Hanya remote | Langsung `_remote`, tanpa sentuh `_local` |

**Yang TIDAK boleh tahu:**
- **Service** tidak boleh ikut memutuskan — cuma tahu cara ambil dari satu sumber, tanpa state.
- **ViewModel** memakai kontrak abstrak — tidak tahu/peduli data dari lokal atau cloud.
- **Domain layer** tetap murni — decision strategi adalah detail implementasi di data layer.

> Decision lokal-vs-cloud **bukan** trigger untuk use case. Itu tanggung jawab bawaan Repository.

### 5.5 Service butuh abstraksi (interface)?

**Tidak perlu.** Referensi konsisten: Service = class konkret terendah, tidak ada interface service.

- Batas abstraksi ada di **Repository interface** (domain), bukan Service.
- Service adalah detail implementasi internal data layer — tidak pernah dipakai langsung oleh ViewModel/domain, jadi tidak ada konsumen eksternal yang butuh kontrak.
- Testing: mock class konkret bisa dengan `mockito`/`mocktail`, jadi testability bukan alasan.

| Komponen | Interface? |
|---|---|
| Repository (domain) | **Ya** — wajib, titik dependency inversion |
| Service | Tidak — konkret, internal data layer |
| Use case | Tidak — class konkret |

> **Aturan praktis:** buat abstraksi hanya di titik yang dilewati lintas layer. Service tidak dilewati lintas layer (Repository → Service, sama-sama di data layer).

### 5.6 Kapan use case dipakai?

**Opsional**, hanya ditambah kalau memenuhi **salah satu atau lebih** dari 3 syarat (dari docs resmi):

1. **Perlu menggabungkan data dari beberapa repository** — pemicu paling umum.
2. **Logikanya sangat kompleks** — aturan bisnis berlapis, validasi multi-langkah.
3. **Logika dipakai ulang oleh banyak ViewModel** — sinyal terkuat: kamu menemukan duplikasi.

**Dua filosofi, satu yang benar:**

| Pendekatan | Deskripsi | Verdict resmi |
|---|---|---|
| **Add-as-needed** (disarankan) | Mulai tanpa use case; tambah saat lihat duplikasi/kompleksitas | ✅ *"A good approach"* |
| Use-case untuk semua akses data | Setiap ViewModel butuh data → wajib lewat use case, meski pass-through | ❌ *"Adds a significant amount of unnecessary overhead"* |

Di case study resmi, sebagian fitur pakai use case, sebagian ViewModel akses repository langsung — campur itu sah.

**Pros & cons (tabel resmi):**

| Pros | Cons |
|---|---|
| ✅ Hindari duplikasi di ViewModel | ❌ Tambah kompleksitas & cognitive load |
| ✅ Testability lebih baik | ❌ Testing butuh mock tambahan |
| ✅ ViewModel lebih mudah dibaca | ❌ Tambah boilerplate |

**Aturan kalau memakai use case:**
- Use case depend ke **repository** (tidak pernah sebaliknya)
- Use case ↔ repository = many-to-many
- ViewModel boleh depend ke use case **dan** repository sekaligus

**Sinyal praktis "sudah waktunya":**
1. Dua ViewModel punya method yang mirip → ekstrak jadi use case
2. ViewModel mulai berisi kalkulasi bisnis yang tidak ada hubungannya dengan UI
3. Satu ViewModel inject 3+ repository dan mengorkestrasi semuanya
4. Logic sama di-copy-paste ke fitur baru

**Contoh nyata (pemicu #1 + #3 — gabung 3 repository, dipakai 2 layar):**

```dart
// SEBELUM — logika numpuk & terduplikasi di 2 ViewModel ❌
class DashboardViewModel {
  // 40 baris menggabungkan EmployeeRepository + AttendanceRepository + LeaveRepository
}

// SESUDAH — use case, kedua ViewModel jadi tipis ✅
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
    // ... mapping & aturan status di sini
  }
}
```

**Contoh pemicu #2 (logic kompleks):**

```dart
// domain/use_cases/calculate_overtime_pay.dart
class CalculateOvertimePay {
  // Aturan: lembur setelah 17:00, minimal 1 jam, 1.5x, >3 jam & weekend 2x,
  // butuh data gaji + absensi — terlalu berat untuk ViewModel
}
```

**Kapan TIDAK butuh use case** — ViewModel baca satu repository langsung, tanpa merge/kompleksitas/reuse:

```dart
class AttendanceViewModel {
  AttendanceViewModel(this._attendance);
  final AttendanceRepository _attendance;   // ← langsung, TANPA use case ✅
}
```

### 5.7 Use case itu domain; repository yang di-inject juga abstract?

**Ya.** Use case tinggal di domain layer (pure), jadi repository yang di-inject adalah **kontrak abstrak** (`domain/repositories/`), bukan impl.

```dart
final EmployeeRepository _employees;   // ✅ abstract
// ❌ Bukan EmployeeRepositoryImpl — itu di data layer
```

- Use case-nya sendiri **class konkret** (tidak butuh interface sendiri) — yang abstrak adalah *dependensi*-nya.
- Impl konkret masuk lewat **DI** — satu-satunya tempat yang tahu impl konkret:

```dart
// core/di/
@Riverpod(keepAlive: true)
GetTodayTeamStatus getTodayTeamStatus(Ref ref) => GetTodayTeamStatus(
      ref.watch(employeeRepository),
      ref.watch(attendanceRepository),
      ref.watch(leaveRepository),
    );
```

> **Rantai abstraksi lengkap:** ViewModel → Use Case → kontrak Repository → (DI menyambungkan) → RepositoryImpl → Services. Domain tetap murni.

### 5.8 Repository boleh di-skip kalau cuma pass-through ke Service?

**Tidak disarankan — referensi justru menyarankan repository tetap ada, bahkan untuk method yang terlihat pass-through.**

"Repository kosong" sebenarnya tidak benar-benar kosong. Contoh resmi Compass app:

```dart
Future<Result<void>> delete(int id) async {
  try {
    return _apiClient.deleteBooking(id);   // ← memang cuma call service
  } on Exception catch (e) {
    return Result.error(e);                // ← tapi ini pekerjaan repository
  }
}
```

Minimal 2 tugas yang **selalu** ada di repository:

1. **Transformasi API model → domain model** — *"raw data that often needs to be filtered, combined, or deleted... The repo refines the raw data and outputs it as domain models."* Kalau terasa "cuma call service", biasanya transformasi itu belum dikerjakan di tempat yang benar.
2. **Error mapping** — tanpa repository, try/catch tersebar di ViewModel.

Aturan eksplisit yang dilanggar kalau di-skip (case study resmi):

> *"It's important that the service is a private member, so that the UI layer can't bypass the repository and call a service directly."*

Kalau ViewModel langsung ke Service:
- UI layer terikat format data mentah API — backend berubah, widget/ViewModel ikut berubah
- Kehilangan satu-satunya tempat sah untuk memutasi tipe data itu
- Besok butuh cache/retry/offline sync → refactor dari awal

> **Yang diizinkan referensi: repository TIPIS.** Anti-pattern bukan "repository isi satu baris", tapi "tidak ada repository sama sekali". Catatan jujur: panduan resmi bilang *"guidelines, not steadfast rules"* — untuk app demo bisa saja ViewModel → Service, tapi itu menyimpang dari rekomendasi dan boundary data-nya hilang. **Yang opsional itu use case, bukan repository.**

### 5.9 Model / Entity / Mapper — best practice

**Konsep dasar — 2 (kadang 3) jenis objek data:**

| Objek | Di mana | Isinya | Siapa yang boleh lihat |
|---|---|---|---|
| **Domain model** (entity) | `domain/models/` | Data murni sesuai bahasa bisnis | Semua layer |
| **DTO** | `data/models/` | Bentuk mentah sumber data (JSON API, row Drift) | HANYA data layer |
| **UI state object** (opsional) | `presentation/` | Snapshot render satu view | View + viewmodel-nya |

**Domain model (entity):**
- **JANGAN kasih suffix `Model`/`Entity`** (konvensi resmi VGV): *"Do not add the word 'Model' or 'Entity' to this class."* → cukup `Product`, `Cart`, `Employee`.
- **Immutable** — panduan resmi memakai `freezed` (deep immutability + copyWith + toJson).
- **Jangan taruh serialisasi di entity** — `fromJson`/`toJson`/`fromMap` adalah urusan DTO.

**DTO:**
- Milik data layer, **tidak boleh diekspor/dibocorkan** ke domain/presentation. Barrel data layer boleh ekspor data sources, **tidak boleh** ekspor DTO & mappers.
- `ApiProduct` dan `DbProduct` jadi dua class terpisah, masing-masing mewakili sumbernya.

**Jangan buat DTO abstrak gabungan** (in-memory/local/remote):

> *"For now, we do not believe it makes sense to define an abstract DTO for in-memory, local, and remote storage options. In general, it adds extra, unnecessary mapping."*

Artinya: kalau tiap sumber data return bentuk berbeda, **pakai DTO masing-masing saja**, tanpa interface/class induk. Titik penyatuannya adalah **domain model**, bukan DTO:

```
ApiProduct ──[ApiToDomainConverter]──┐
                                      ├──> Product  ← satu-satunya bentuk bersama
DbProduct ──[DbToDomainConverter]────┘
```

Alasannya: DTO biasanya code-generated (Drift/Swagger/json_serializable) dan tightly coupled ke sumbernya; abstract DTO = lapisan mapping ekstra tanpa manfaat; kontrak bersama yang perlu abstraksi hanya repository interface.

**Mapper — 3 opsi yang diakui VGV:**

```dart
// Opsi A — extension method (paling sederhana)
extension ApiProductX on ApiProduct {
  Product toDomain() => Product(id: id, title: title, price: priceInCents / 100);
}

// Opsi B — Converter class (difavoritkan VGV, testable)
class DbToDomainProductConverter extends Converter<DbProduct, Product> {
  const DbToDomainProductConverter();
  @override
  Product convert(DbProduct dbProduct) => Product(
        id: dbProduct.id,
        price: dbProduct.priceInCents / 100,  // DB cents → domain rupiah
      );
}

// Opsi C — code generation: auto_mappr + build_runner
```

**Aturan mapper:**
1. **Satu converter per sumber data per arah** — fitur yang baca DB + API punya `DbToDomainX` **dan** `ApiToDomainX`.
2. **Konversi unit/format hidup di mapper, bukan di Bloc** — *"The database stores cents as an integer... the conversion belongs here, not in a Bloc."*
3. Repository yang menerapkan converter saat data keluar/masuk source.

### 5.10 Pola "Summary" (VGV) itu use case kan?

**Ya, persis.** VGV cuma pakai istilah berbeda:

| Panduan resmi Flutter | VGV FFCA | Peran |
|---|---|---|
| Use case (interactor) | **Command** | Aksi modifikasi data (`UpdateProductTitleCommand`) |
| Use case (interactor) | **Query** | Membaca data (`GetProductByIdQuery`, `WatchProductByIdQuery`) |

> *"Combined, these are often referred to as 'use cases.' For the purposes of our codebase, we recommend avoiding the term 'use case' and instead prefer Command and Query classes for clarity."*

Syarat dibuatnya juga sama: *"If a Command or Query only serves to call out to the repository, there is no need to introduce the class at all."*

**Pola Summary untuk komposisi antar fitur:** jangan simpan objek `Product` utuh di `Cart` — simpan ID-nya, lalu Query yang menggabungkan:

```dart
// domain: CartSummary { id, productIds } — bukan Cart { List<Product> }

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

Fitur cart tidak tahu cara Product disimpan; fitur product tidak tahu cart ada. Inilah trigger use case #1 (merge multi-repository) dalam bentuk nyata.

### 5.11 Repository: input & output-nya domain model? Siapa yang convert?

**Keduanya benar:**

1. **Contract repository (domain): input & output = domain model.**
2. **Konversi domain → DTO terjadi di implementasi repository, BUKAN di service.**

Bukti dari case study resmi:

```dart
// Service (ApiClient) — berbicara bahasa MENTAH sumber data:
Future<Result<BookingApiModel>> postBooking(BookingApiModel booking) async {...}

// Kontrak repository (domain) — bahasa domain:
abstract class BookingRepository {
  Future<Result<void>> createBooking(Booking booking);
}

// Impl repository — titik konversi:
class BookingRepositoryRemote implements BookingRepository {
  @override
  Future<Result<void>> createBooking(Booking booking) async {
    final apiModel = booking.toApiModel();   // ← domain → DTO DI SINI
    return _apiClient.postBooking(apiModel);
  }
}
```

**Kenapa bukan di Service?**
1. Service = pembungkus endpoint yang bodoh (*"its only job is to wrap an external API"*). Kalau tahu domain model, ia tidak lagi murni dan terkontaminasi logika bisnis tiap ganti backend.
2. Kepemilikan mapping ada di data layer, dan **repository yang menerapkannya**. Converter dua arah:

```dart
// data/mappers/domain_to_db_product_converter.dart
class DomainToDbProductConverter extends Converter<Product, DbProduct> {
  @override
  DbProduct convert(Product product) => DbProduct(
        id: product.id,
        priceInCents: (product.price * 100).round(),   // rupiah → cents
      );
}
```

**Alur lengkap:**

```
READ:  API/DB → Service (DTO) → Repo impl [DTO → Domain] → ViewModel
WRITE: ViewModel → Repo impl [Domain → DTO] → Service (DTO) → API/DB
                        └──────── translator boundary ────────┘
```

> Domain model tidak pernah keluar dari domain+presentation; DTO tidak pernah masuk ke dalamnya. Semua penerjemahan dua arah terkonsentrasi di impl repository + mapper.

### 5.12 Bisakah batas layer (domain ≠ import data) di-enforce tanpa separate package?

**Bisa** — lewat linter/analyzer plugin. Pelanggarannya ketahuan langsung di IDE dan menggagalkan `dart analyze` di CI, tanpa perlu memecah kode jadi package terpisah.

**Alatnya:**

| Alat | Cara | Catatan |
|---|---|---|
| **[`dart_boundaries`](https://pub.dev/packages/dart_boundaries)** | Konfigurasi YAML di `analysis_options.yaml`, dibangun di atas `custom_lint` | Paling pas; punya rule `layer_boundaries` & `no_banned_imports` |
| **[`import_lint`](https://pub.dev/packages/import_lint)** | Mirip, definisikan layer via YAML | Lebih dulu populer |
| **DCM (`dart_code_metrics`)** | rule `avoid-banned-imports` | Kalau kamu sudah pakai DCM |
| **Tulis sendiri** dengan `custom_lint` | Rule custom | Fleksibel tapi lebih repot |

**Cara cepat — `no_banned_imports`:**

```yaml
# analysis_options.yaml
dart_boundaries:
  no_banned_imports:
    entries:
      - paths: ['lib/features/**/domain/**']
        deny: ['lib/features/**/data/**']
        message: 'Domain layer tidak boleh mengimpor data layer.'
```

**Cara lebih lengkap — `layer_boundaries` (define arah dependensi tiap layer):**

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
        disallow: [data]          # ← inti aturannya
      - from: presentation
        allow: [domain]           # presentation hanya boleh lihat domain
      - from: data
        allow: [domain]           # data boleh lihat domain (untuk implement kontrak)
```

`{{ feature }}` adalah capture group — file dengan nama feature sama dianggap intra-feature (tetap boleh), beda feature dianggap cross-feature (bisa dilarang juga kalau mau).

**Cara kerjanya:**
- **Live di IDE** — pelanggaran muncul sebagai warning/error saat menulis.
- **CI** — `dart analyze` gagal → PR ter-blokir.
- Bisa di-`// ignore:` kalau memang butuh pengecualian.

**Batasan penting (jujur):** linter = **soft boundary**, bukan hard boundary.

| | Folder + linter | Separate package |
|---|---|---|
| Mekanisme | `dart analyze` / CI gate | **Compile error** |
| Dev bisa "nulis" kode pelanggar? | Bisa nulis, tapi gagal di analyze/CI | Tidak bisa, langsung error saat import |
| Bergantung pada | Konvensi nama folder (glob pattern) | Deklarasi `pubspec.yaml` |
| Scope buat AI agent | Tidak otomatis (semua satu paket) | Dependency graph membatasi konteks |

Dua poin kuncinya:

1. **Rule-nya berbasis path/glob**, jadi sekuat konvensi foldermu (`features/x/domain`, `features/x/data`). Kalau file ditaruh di folder yang salah, rule tidak menolong.
2. **Tidak menghentikan compilation** — hanya menghentikan merge lewat CI. Kalau CI-mu wajib, ini efektif.

Ini persis yang VGV lakukan: mereka pakai **custom analyzer untuk aturan yang lebih halus** (naming, cross-feature data leaks, dependency cycle), **tapi tetap memilih package untuk hard boundary** — karena *"a layer violation is a compile error, not a review comment"*.

**Rekomendasi:**
- **Single app, tim kecil, CI wajib** → `dart_boundaries` (atau `import_lint`) sudah cukup, tanpa overhead package.
- **Multi-app / banyak tim / AI-assisted / butuh jaminan paling kuat** → naik ke separate package (FFCA), baru tambahkan linter di atasnya untuk aturan halus.

---

## 6. Cheat Sheet — Aturan Emas

| # | Aturan |
|---|---|
| 1 | Feature-first: fitur di level atas, layer di dalam fitur |
| 2 | Dependensi selalu ke dalam: View → ViewModel → Repository → Service |
| 3 | Domain murni Dart, tidak tahu data impl; data implementasi kontrak domain |
| 4 | View hanya render + trigger; logika di ViewModel/Bloc/Cubit |
| 5 | Repository = decision lokal-vs-cloud, caching, retry, error mapping — jangan di-skip |
| 6 | Service bodoh, konkret, 1 per sumber data, tanpa state, tanpa interface |
| 7 | Use case/Command/Query opsional — hanya saat merge multi-repo, logic kompleks, atau reuse |
| 8 | Entity tanpa suffix, immutable (freezed), tanpa JSON knowledge |
| 9 | DTO per sumber data, private di data layer, jangan dibocorkan, jangan diabstraksi |
| 10 | Mapper 1 per sumber per arah; konversi unit di mapper, bukan di Bloc |
| 11 | Repository interface = satu-satunya kontrak lintas layer (prefix `I` gaya VGV opsional) |
| 12 | Konversi domain ⇄ DTO di impl repository, bukan di service |
| 13 | Repository tidak saling kenal; gabungkan di ViewModel/use case |
| 14 | DI untuk semua: komponen di-inject, tidak di-instantiate di widget |
| 15 | App-wide session state (login, cache) tinggal di repository via DI |

---

## 7. Referensi

Referensi ini adalah sumber dari seluruh catatan di atas — tiap bagian di dokumen ini mengacu ke sana, tapi kontennya sengaja ditulis ringkas agar tidak terbebani sitasi di tiap paragraf.

| Referensi | Yang diambil darinya |
|---|---|
| [Flutter Docs – Guide to App Architecture](https://docs.flutter.dev/app-architecture/guide) | Layering resmi (View/ViewModel/Repository/Service), aturan dependensi, 3 pemicu use case, peran repository & service, MVVM |
| [Flutter Docs – Case Study: UI Layer](https://docs.flutter.dev/app-architecture/case-study/ui-layer) | Contoh ViewModel & UI state (`ChangeNotifier` + `freezed`), command, pola view 1:1 viewmodel |
| [Flutter Docs – Case Study: Data Layer](https://docs.flutter.dev/app-architecture/case-study/data-layer) | Contoh repository & service, aturan service private, API model vs domain model, `Result` pattern |
| [Very Good Ventures – Feature-First Clean Architecture](https://verygood.ventures/blog/feature-first-clean-architecture/) | Struktur monorepo FFCA, batas layer via paket, callback injection untuk navigasi |
| [VGV Engineering – FFCA (Domain & Data Layer)](https://engineering.verygood.ventures/architecture/ffca/overview/) | Konvensi naming entity (tanpa suffix), Command/Query, pola Summary, aturan DTO/mapper, converter per sumber per arah |
| [Very Good Ventures – Very Good Layered Architecture](https://verygood.ventures/blog/very-good-flutter-architecture/) | Perbandingan layer-first vs feature-first, kapan masing-masing cocok |
| [Tech With Sam – Clean Architecture in Flutter 2026](https://techwithsam.dev/articles/clean-architecture-in-flutter-2026-practical-implementation-guide) | Implementasi praktis MVVM + Riverpod 3, contoh cache-then-network, kesalahan umum |
| [Mealify (FFCA reference app)](https://github.com/VGVentures/mealify_feature_first) · [Compass app (referensi resmi)](https://github.com/flutter/samples/tree/main/compass_app) | Contoh codebase nyata dari kedua pendekatan |
