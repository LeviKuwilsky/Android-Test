# Android — Room · Retrofit · Saubere Architektur

> Kotlin · Jetpack Compose · MVVM · Coroutines/Flow · Room · Retrofit
> Alles, was man für „Mischung Retrofit + Room + saubere Architektur" braucht.
> **Code = Kotlin, Kommentare = Deutsch.** Versionsnummern ggf. auf den aktuellsten Stand prüfen.

---

## 🎯 TEST-TAG SCHNELLSTART — „Der größere Wahlhelfer"

> **Open-Book-Strategie:** Diesen Block von oben nach unten abarbeiten. Code ist schon auf `Party`/`votes` gemünzt — du musst NICHTS mehr umbenennen, nur **Package-Name** anpassen und Lücken füllen.
> **#1-Falle vermeiden:** überall `at.htl.wahlhelfer` durch deinen echten Package-Namen ersetzen (Strg+R „Replace in Files").

### 📁 Genaue Ordner-/Package-Struktur
> Package frei wählbar — nur **nicht** `com.example`. Beispiel: `at.htl.wahlhelfer`.

```
app/src/main/
├── AndroidManifest.xml                 # INTERNET-Permission (falls Retrofit) + android:name=".WahlhelferApp"
└── java/at/htl/wahlhelfer/
    ├── WahlhelferApp.kt                # Application = DI-Container (hält das Repository)
    ├── MainActivity.kt                 # setContent { Theme { AppNavDisplay() } }
    │
    ├── data/                           # ── DATEN-SCHICHT ──
    │   ├── local/
    │   │   ├── Party.kt                # @Entity  (eine Tabelle)
    │   │   ├── PartyDao.kt             # @Dao     (DB-Zugriff, gibt Flow zurück)
    │   │   └── AppDatabase.kt          # @Database (Singleton)
    │   └── PartyRepository.kt          # einzige Brücke zur DB
    │
    └── ui/                             # ── UI-SCHICHT ──
        ├── AppViewModelProvider.kt     # ViewModel-Factory (gibt Repo rein)
        ├── navigation/
        │   ├── NavKeys.kt              # @Serializable NavKeys (Ziele)
        │   └── AppNavDisplay.kt        # NavDisplay + Back-Stack (Nav3)
        ├── home/HomeScreen.kt
        ├── count/
        │   ├── CountViewModel.kt
        │   └── CountScreen.kt
        ├── overview/
        │   ├── OverviewViewModel.kt
        │   └── OverviewScreen.kt
        ├── about/AboutScreen.kt
        └── theme/                      # kommt aus „Empty Activity", unverändert lassen
```

**Faustregel:** Eine Datei = eine Klasse. Package = Ordner. Daten unten, UI oben, dazwischen nur das Repository.

### ✅ Baureihenfolge — 8 Schritte (genau so abtippen)

> Nach jedem Block einmal bauen lohnt sich. Reihenfolge ist wichtig: untere Schichten zuerst.

**① Dependencies** → kompletten Block aus Abschnitt [2](#2-gradle--dependencies) übernehmen (enthält schon Room, **Navigation 3**, Compose, Lifecycle + optional Retrofit). Pflicht-Plugins: `ksp` (für Room) + `jetbrains-kotlin-serialization` (für die NavKeys).

**② `data/local/Party.kt` — Entity**
```kotlin
package at.htl.wahlhelfer.data.local
import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "parties")
data class Party(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String,
    val votes: Int = 0
)
```

**③ `data/local/PartyDao.kt` — DAO**
```kotlin
package at.htl.wahlhelfer.data.local
import androidx.room.*
import kotlinx.coroutines.flow.Flow

@Dao
interface PartyDao {
    @Query("SELECT * FROM parties ORDER BY votes DESC")
    fun observeAll(): Flow<List<Party>>              // LESEN: Flow, kein suspend

    @Query("SELECT COALESCE(SUM(votes), 0) FROM parties")
    fun observeTotal(): Flow<Int>                    // Gesamtsumme reaktiv

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(party: Party): Long           // SCHREIBEN: suspend

    @Query("UPDATE parties SET votes = MAX(0, votes + :delta) WHERE id = :id")
    suspend fun changeVotes(id: Long, delta: Int)    // +1 / -1, nie unter 0

    @Query("UPDATE parties SET votes = 0")
    suspend fun resetAll()                           // alle Stimmen auf 0 ("Reset")

    @Query("SELECT COUNT(*) FROM parties")
    suspend fun count(): Int
}
```

**④ `data/local/AppDatabase.kt` — Database (Singleton)**
```kotlin
package at.htl.wahlhelfer.data.local
import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

@Database(entities = [Party::class], version = 1, exportSchema = false)
abstract class AppDatabase : RoomDatabase() {
    abstract fun partyDao(): PartyDao

    companion object {
        @Volatile private var INSTANCE: AppDatabase? = null
        fun getInstance(context: Context): AppDatabase =
            INSTANCE ?: synchronized(this) {
                INSTANCE ?: Room.databaseBuilder(
                    context.applicationContext, AppDatabase::class.java, "wahlhelfer.db"
                ).build().also { INSTANCE = it }
            }
    }
}
```

**⑤ `data/PartyRepository.kt` — Repository**
```kotlin
package at.htl.wahlhelfer.data
import at.htl.wahlhelfer.data.local.Party
import at.htl.wahlhelfer.data.local.PartyDao
import kotlinx.coroutines.flow.Flow

class PartyRepository(private val dao: PartyDao) {
    val parties: Flow<List<Party>> = dao.observeAll()   // UI liest IMMER hier
    val total: Flow<Int> = dao.observeTotal()

    suspend fun plus(p: Party)  = dao.changeVotes(p.id, +1)   // sofort speichern
    suspend fun minus(p: Party) = dao.changeVotes(p.id, -1)
    suspend fun reset()         = dao.resetAll()

    // Demo-Parteien anlegen, falls DB leer (statt eigener Eingabemaske):
    suspend fun seedIfEmpty() {
        if (dao.count() == 0)
            listOf("SPÖ", "ÖVP", "FPÖ", "GRÜNE", "NEOS").forEach { dao.insert(Party(name = it)) }
    }
}
```

**⑥ `WahlhelferApp.kt` (+ Manifest)** — manuelle DI
```kotlin
package at.htl.wahlhelfer
import android.app.Application
import at.htl.wahlhelfer.data.PartyRepository
import at.htl.wahlhelfer.data.local.AppDatabase

class WahlhelferApp : Application() {
    val repository by lazy { PartyRepository(AppDatabase.getInstance(this).partyDao()) }
}
```
```xml
<!-- AndroidManifest.xml: im <application>-Tag ergänzen -->
<application android:name=".WahlhelferApp" ... >
```

**⑦ ViewModels + Factory**
```kotlin
// ui/AppViewModelProvider.kt
package at.htl.wahlhelfer.ui
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewmodel.CreationExtras
import androidx.lifecycle.viewmodel.initializer
import androidx.lifecycle.viewmodel.viewModelFactory
import at.htl.wahlhelfer.WahlhelferApp
import at.htl.wahlhelfer.ui.count.CountViewModel
import at.htl.wahlhelfer.ui.overview.OverviewViewModel

object AppViewModelProvider {
    val Factory = viewModelFactory {
        initializer { CountViewModel(app().repository) }
        initializer { OverviewViewModel(app().repository) }
    }
}
private fun CreationExtras.app(): WahlhelferApp =
    this[ViewModelProvider.AndroidViewModelFactory.APPLICATION_KEY] as WahlhelferApp
```
```kotlin
// ui/count/CountViewModel.kt
package at.htl.wahlhelfer.ui.count
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import at.htl.wahlhelfer.data.PartyRepository
import at.htl.wahlhelfer.data.local.Party
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

class CountViewModel(private val repo: PartyRepository) : ViewModel() {
    val parties: StateFlow<List<Party>> =
        repo.parties.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    init { viewModelScope.launch { repo.seedIfEmpty() } }   // einmal Demo-Daten

    fun plus(p: Party)  = viewModelScope.launch { repo.plus(p) }   // Klick -> Coroutine -> Room
    fun minus(p: Party) = viewModelScope.launch { repo.minus(p) }
    fun reset()         = viewModelScope.launch { repo.reset() }
}
```
```kotlin
// ui/overview/OverviewViewModel.kt
package at.htl.wahlhelfer.ui.overview
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import at.htl.wahlhelfer.data.PartyRepository
import at.htl.wahlhelfer.data.local.Party
import kotlinx.coroutines.flow.*

class OverviewViewModel(repo: PartyRepository) : ViewModel() {
    val parties: StateFlow<List<Party>> =
        repo.parties.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
    val total: StateFlow<Int> =
        repo.total.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), 0)
}
```

**⑧ Navigation 3 + Screens** (Details + Parameter-Übergabe in [Abschnitt 10★](#10-navigation-3-nav3))
```kotlin
// ui/navigation/NavKeys.kt  — typsichere Ziele
package at.htl.wahlhelfer.ui.navigation
import androidx.navigation3.runtime.NavKey
import kotlinx.serialization.Serializable

@Serializable data object Home : NavKey
@Serializable data object Count : NavKey
@Serializable data object Overview : NavKey
@Serializable data object About : NavKey
```
```kotlin
// ui/navigation/AppNavDisplay.kt
package at.htl.wahlhelfer.ui.navigation
import androidx.compose.runtime.Composable
import androidx.navigation3.runtime.entryProvider
import androidx.navigation3.runtime.rememberNavBackStack
import androidx.navigation3.ui.NavDisplay
import at.htl.wahlhelfer.ui.about.AboutScreen
import at.htl.wahlhelfer.ui.count.CountScreen
import at.htl.wahlhelfer.ui.home.HomeScreen
import at.htl.wahlhelfer.ui.overview.OverviewScreen

@Composable
fun AppNavDisplay() {
    val backStack = rememberNavBackStack(Home)                 // Start = Home
    val goAbout = { if (backStack.lastOrNull() != About) backStack.add(About) }
    NavDisplay(
        backStack = backStack,
        onBack = { backStack.removeLastOrNull() },             // System-Zurück
        entryProvider = entryProvider {
            entry<Home> {
                HomeScreen(
                    onCount    = { backStack.add(Count) },     // vorwärts = add
                    onOverview = { backStack.add(Overview) },
                    onAbout    = goAbout)
            }
            entry<Count>    { CountScreen(onBack = { backStack.removeLastOrNull() }, onAbout = goAbout) }
            entry<Overview> { OverviewScreen(onBack = { backStack.removeLastOrNull() }, onAbout = goAbout) }
            entry<About>    { AboutScreen(onBack = { backStack.removeLastOrNull() }) }
        }
    )
}
```
```kotlin
// ui/count/CountScreen.kt  — Liste + / − + Reset, liest aus DB
package at.htl.wahlhelfer.ui.count
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import at.htl.wahlhelfer.ui.AppViewModelProvider

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CountScreen(
    onBack: () -> Unit,
    onAbout: () -> Unit,
    vm: CountViewModel = viewModel(factory = AppViewModelProvider.Factory)
) {
    val parties by vm.parties.collectAsStateWithLifecycle()
    Scaffold(
        topBar = { TopAppBar(title = { Text("Zählen") },
            actions = { TextButton(onClick = vm::reset) { Text("Reset") }
                        TextButton(onClick = onAbout) { Text("About") } }) }
    ) { p ->
        LazyColumn(Modifier.padding(p)) {
            items(parties, key = { it.id }) { party ->
                ListItem(
                    headlineContent  = { Text(party.name) },
                    supportingContent = { Text("Stimmen: ${party.votes}") },
                    trailingContent  = {
                        Row {
                            Button(onClick = { vm.minus(party) }) { Text("−") }
                            Spacer(Modifier.width(8.dp))
                            Button(onClick = { vm.plus(party) })  { Text("+") }
                        }
                    })
                HorizontalDivider()
            }
        }
    }
}
```
```kotlin
// ui/overview/OverviewScreen.kt  — Gesamtstand aus DB
package at.htl.wahlhelfer.ui.overview
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import at.htl.wahlhelfer.ui.AppViewModelProvider

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun OverviewScreen(
    onBack: () -> Unit,
    onAbout: () -> Unit,
    vm: OverviewViewModel = viewModel(factory = AppViewModelProvider.Factory)
) {
    val parties by vm.parties.collectAsStateWithLifecycle()
    val total   by vm.total.collectAsStateWithLifecycle()
    Scaffold(topBar = { TopAppBar(title = { Text("Übersicht") }) }) { p ->
        Column(Modifier.padding(p).padding(16.dp)) {
            Text("Gesamt: $total Stimmen", style = MaterialTheme.typography.headlineSmall)
            Spacer(Modifier.height(8.dp))
            LazyColumn {
                items(parties, key = { it.id }) { Text("${it.name}: ${it.votes}") }
            }
        }
    }
}
```
```kotlin
// ui/home/HomeScreen.kt  — nur Buttons
package at.htl.wahlhelfer.ui.home
import androidx.compose.foundation.layout.*
import androidx.compose.material3.Button
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun HomeScreen(onCount: () -> Unit, onOverview: () -> Unit, onAbout: () -> Unit) {
    Column(Modifier.fillMaxSize().padding(24.dp)) {
        Button(onClick = onCount,    modifier = Modifier.fillMaxWidth()) { Text("Stimmen zählen") }
        Spacer(Modifier.height(12.dp))
        Button(onClick = onOverview, modifier = Modifier.fillMaxWidth()) { Text("Übersicht") }
        Spacer(Modifier.height(12.dp))
        Button(onClick = onAbout,    modifier = Modifier.fillMaxWidth()) { Text("About") }
    }
}
// ui/about/AboutScreen.kt analog: Text + Button(onClick = onBack) { Text("Zurück") }
```
```kotlin
// MainActivity.kt  — Einstiegspunkt
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContent { WahlhelferTheme { AppNavDisplay() } }   // Theme-Name = dein Projektname
}
```

> ✅ **Navigation-Hinweis:** Dieses Skelett nutzt **Navigation 3** (`androidx.navigation3`) — genau wie dein ElectorApp aus Test 3. Volle Erklärung (NavKeys, Back-Stack, Parameter an einen Screen übergeben, ViewModel mit Nav-Parameter) in **[Abschnitt 10★](#10-navigation-3-nav3)**.

> **Wenn dieses Skelett steht, hast du Room + MVVM + Repository + Navigation + Flow komplett.** Den Rest (Styling, About-Text) drauflegen.

---

## 📑 Inhalt
0. [🎯 TEST-TAG SCHNELLSTART (Wahlhelfer, Ordnerstruktur + 8 Schritte)](#-test-tag-schnellstart--der-größere-wahlhelfer)
1. [Die Architektur in 30 Sekunden](#1-die-architektur-in-30-sekunden)
2. [Gradle / Dependencies](#2-gradle--dependencies)
3. [Manifest (WICHTIG: INTERNET!)](#3-manifest-wichtig-internet)
4. [Coroutines & Flow — Grundlagen](#4-coroutines--flow--grundlagen)
5. [ROOM komplett](#5-room-komplett)
6. [RETROFIT komplett](#6-retrofit-komplett)
7. [Repository (die einzige DB-/Netz-Brücke)](#7-repository)
8. [Die MISCHUNG: offline-first (Netzwerk → Room → UI)](#8-die-mischung-offline-first)
8★. [⭐ TEST-SZENARIO: Lesen → Berechnen → Speichern](#8-test-szenario-lesen--berechnen--speichern)
8★★. [⭐ Mehrere DAOs / mehrere Datenbanken](#8-mehrere-daos--mehrere-datenbanken)
9. [ViewModel + UiState](#9-viewmodel--uistate)
10. [Compose-UI anbinden](#10-compose-ui-anbinden)
10★. [🧭 NAVIGATION 3 (Nav3)](#10-navigation-3-nav3)
11. [Dependency Injection (manuell + Hilt)](#11-dependency-injection)
12. [KOMPLETTBEISPIEL end-to-end](#12-komplettbeispiel-end-to-end)
13. [Häufige Fehler / Fallen](#13-häufige-fehler--fallen)
13★. [🔄 Versionen aktuell halten / updaten](#13-versionen-aktuell-halten--updaten)
14. [Nützliche Links](#14-nützliche-links)

---

## 1. Die Architektur in 30 Sekunden

**Regel:** Daten fließen in **eine** Richtung. UI kennt nur das ViewModel, das ViewModel kennt nur das Repository, das Repository kennt die Datenquellen (Room + Retrofit). Niemals überspringen.

```
┌──────────────┐   StateFlow    ┌────────────┐   Flow/suspend   ┌──────────────┐
│   UI         │ ◀───────────── │ ViewModel  │ ◀─────────────── │ Repository   │
│ (Compose)    │   Events ───▶  │ (UiState)  │   Aktionen ───▶  │ (Single      │
└──────────────┘                └────────────┘                  │  Source of   │
                                                                 │  Truth)      │
                                                          ┌──────┴──────┐
                                                          ▼             ▼
                                                    ┌──────────┐  ┌──────────┐
                                                    │  Room    │  │ Retrofit │
                                                    │ (lokal)  │  │ (Netz)   │
                                                    └──────────┘  └──────────┘
```

| Schicht | Aufgabe | Faustregeln |
|---|---|---|
| **UI (Compose)** | anzeigen + Events melden | keine Logik, kein DB/Netz-Zugriff |
| **ViewModel** | UI-State halten, Aktionen auslösen | überlebt Rotation, `viewModelScope`, kein Android-Context-Leak |
| **Repository** | Datenquellen koordinieren | **einzige** Stelle, die Room/Retrofit kennt |
| **Room** | lokale Persistenz (Single Source of Truth) | gibt `Flow` zurück |
| **Retrofit** | Netzwerk | gibt `suspend` zurück, befüllt Room |

**Begriffe für die Prüfung:** *Unidirectional Data Flow (UDF)*, *Single Source of Truth (SSOT)*, *Separation of Concerns*, *MVVM*, *offline-first*.

---

## 2. Gradle / Dependencies

> ⭐ **Erprobter Versionssatz** — läuft sync-sauber auf deinem Rechner (aus dem ElectorApp übernommen). Nur Zahlen **hier** in der `.toml` ändern. Versionen sind eine Kette (siehe [13★](#13-versionen-aktuell-halten--updaten)) — Kotlin/AGP/KSP nie einzeln anfassen.
>
> **`gradle/wrapper/gradle-wrapper.properties`:** `distributionUrl=...gradle-9.5.1-bin.zip` (passt zu AGP 9.2.1).
>
> **SDK-Merke:** `minSdk = 29` (Android 10 „Q") = Untergrenze, bleibt. `compileSdk` = **mindestens so hoch, wie deine neueste Library verlangt** (aktuell **37**, weil `core-ktx 1.19.0` das fordert) — und so hoch, wie du installiert hast. `targetSdk` darf niedriger sein (36). Fehler „requires compile against version X"? → einfach `compileSdk = X` (sofern installiert), das ist ein sicherer Eingriff. Du brauchst **keinen** API-29-Emulator — auf einem neueren testen (API 37) ist erlaubt.

### `gradle/libs.versions.toml` (Version Catalog)
```toml
[versions]
agp = "9.2.1"                       # passt zu Gradle 9.5.1
kotlin = "2.3.21"
ksp = "2.3.9"                       # MUSS exakt zur Kotlin-Version passen
composeBom = "2026.05.01"
room = "2.8.4"
lifecycle = "2.10.0"
activityCompose = "1.13.0"
coreKtx = "1.18.0"          # frische Projekte ziehen evtl. 1.19.0 -> dann compileSdk = 37 (siehe unten)
navigation3 = "1.1.2"
kotlinxSerialization = "1.11.0"
# --- nur falls Retrofit gefordert ---
retrofit = "2.11.0"
retrofitKotlinxConverter = "1.0.0"
okhttp = "4.12.0"

[libraries]
# Basis
androidx-core-ktx                    = { group = "androidx.core", name = "core-ktx", version.ref = "coreKtx" }
androidx-activity-compose            = { group = "androidx.activity", name = "activity-compose", version.ref = "activityCompose" }
androidx-lifecycle-runtime-ktx       = { group = "androidx.lifecycle", name = "lifecycle-runtime-ktx", version.ref = "lifecycle" }
# Compose (BOM steuert ALLE Compose-Versionen automatisch)
androidx-compose-bom                 = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
androidx-compose-ui                  = { group = "androidx.compose.ui", name = "ui" }
androidx-compose-ui-graphics         = { group = "androidx.compose.ui", name = "ui-graphics" }
androidx-compose-ui-tooling          = { group = "androidx.compose.ui", name = "ui-tooling" }
androidx-compose-ui-tooling-preview  = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
androidx-compose-material3           = { group = "androidx.compose.material3", name = "material3" }
# ViewModel + Flow in Compose
androidx-lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycle" }
androidx-lifecycle-runtime-compose   = { group = "androidx.lifecycle", name = "lifecycle-runtime-compose", version.ref = "lifecycle" }
# Navigation 3 (Nav3 — siehe Abschnitt 10★)
androidx-navigation3-runtime         = { group = "androidx.navigation3", name = "navigation3-runtime", version.ref = "navigation3" }
androidx-navigation3-ui              = { group = "androidx.navigation3", name = "navigation3-ui", version.ref = "navigation3" }
# Room
androidx-room-runtime  = { group = "androidx.room", name = "room-runtime",  version.ref = "room" }
androidx-room-ktx      = { group = "androidx.room", name = "room-ktx",      version.ref = "room" }
androidx-room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
# Serialization: -core reicht für @Serializable NavKeys; -json zusätzlich nur für Retrofit
kotlinx-serialization-core     = { module = "org.jetbrains.kotlinx:kotlinx-serialization-core", version.ref = "kotlinxSerialization" }
kotlinx-serialization-json     = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinxSerialization" }
# --- nur falls Retrofit gefordert ---
retrofit                       = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
retrofit-kotlinx-serialization = { module = "com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter", version.ref = "retrofitKotlinxConverter" }
okhttp-logging-interceptor     = { module = "com.squareup.okhttp3:logging-interceptor", version.ref = "okhttp" }

[plugins]
android-application            = { id = "com.android.application", version.ref = "agp" }
kotlin-compose                 = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
ksp                            = { id = "com.google.devtools.ksp", version.ref = "ksp" }
jetbrains-kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

### `app/build.gradle.kts`
```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.compose)                  // ab AGP 9 erledigt das auch die Kotlin-Einrichtung
    alias(libs.plugins.jetbrains.kotlin.serialization)  // für @Serializable (Nav3-Keys + JSON)
    alias(libs.plugins.ksp)                             // für Room
}

android {
    namespace = "htl.kuwilsky.deinprojekt"              // DEIN Package, nicht com.example
    compileSdk = 37        // so HOCH wie deine neueste Lib verlangt (core-ktx 1.19.0 -> braucht 37)
    // Block-Form (AGP 9), falls dein Template das nutzt: compileSdk { version = release(37) }

    defaultConfig {
        applicationId = "htl.kuwilsky.deinprojekt"
        minSdk = 29        // Android 10 "Q" – läuft AB hier (nur die Untergrenze!)
        targetSdk = 36     // NICHT auf 29 senken, sonst Build-Crash bei modernem AndroidX (37 ginge auch)
        versionCode = 1
        versionName = "1.0"
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_21    // mind. Java 21
        targetCompatibility = JavaVersion.VERSION_21
    }
    buildFeatures { compose = true }
}

dependencies {
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.lifecycle.runtime.ktx)
    implementation(libs.androidx.activity.compose)
    // Compose
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.compose.ui)
    implementation(libs.androidx.compose.ui.graphics)
    implementation(libs.androidx.compose.ui.tooling.preview)
    implementation(libs.androidx.compose.material3)
    debugImplementation(libs.androidx.compose.ui.tooling)
    // ViewModel + Flow in Compose
    implementation(libs.androidx.lifecycle.viewmodel.compose)
    implementation(libs.androidx.lifecycle.runtime.compose)
    // Navigation 3
    implementation(libs.androidx.navigation3.runtime)
    implementation(libs.androidx.navigation3.ui)
    implementation(libs.kotlinx.serialization.core)     // @Serializable NavKeys
    // Room
    implementation(libs.androidx.room.runtime)
    implementation(libs.androidx.room.ktx)              // Flow + suspend
    ksp(libs.androidx.room.compiler)                    // Codegenerierung
    // --- nur falls Retrofit gefordert ---
    implementation(libs.retrofit)
    implementation(libs.retrofit.kotlinx.serialization)
    implementation(libs.kotlinx.serialization.json)     // -json, nicht nur -core!
    implementation(libs.okhttp.logging.interceptor)
}
```

---

## 3. Manifest (WICHTIG: INTERNET!)

> ⚠️ **Der #1-Anfängerfehler bei Retrofit:** INTERNET-Permission vergessen → App stürzt / kein Netz.

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:name=".MyApplication"          <!-- für manuelle DI -->
        android:usesCleartextTraffic="false"   <!-- true NUR für http:// in Tests -->
        ... >
        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

---

## 4. Coroutines & Flow — Grundlagen

```kotlin
// suspend = "kann pausieren, läuft asynchron" (nur aus Coroutine/anderer suspend-Funktion aufrufbar)
suspend fun ladeDaten(): List<String> { /* ... */ }

// Coroutine im ViewModel starten (an Lifecycle gebunden):
viewModelScope.launch {
    val daten = repository.ladeDaten()   // suspend-Aufruf
}

// Dispatcher (Thread-Pool):
withContext(Dispatchers.IO)      { /* Netz/DB/Datei */ }
withContext(Dispatchers.Default) { /* CPU-lastig (Sortieren etc.) */ }
// Main = UI-Thread (Compose/View). Room & Retrofit wechseln intern selbst auf IO.
```

### Flow / StateFlow (reaktive Streams)
```kotlin
// Flow<T>  : Strom von Werten über die Zeit (z. B. aus Room)
// StateFlow<T> : Flow mit IMMER genau einem aktuellen Wert (perfekt für UI-State)

// Flow → StateFlow umwandeln (im ViewModel):
val state: StateFlow<List<Party>> = repository.parties   // Flow<List<Party>>
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),  // 5s nach letztem Beobachter stoppen
        initialValue = emptyList()
    )

// Mehrere Flows kombinieren:
val ui = combine(repository.parties, repository.total) { parteien, summe ->
    UiState(parteien, summe)
}

// Transformieren:
flow.map { it.filter { p -> p.votes > 0 } }
flow.catch { e -> emit(emptyList()) }    // Fehler abfangen
```

**Operatoren-Spickzettel:** `map` (umwandeln) · `filter` · `combine` (mehrere Flows) · `stateIn` (→ StateFlow) · `catch` (Fehler) · `flowOn(Dispatchers.IO)` (Thread wechseln) · `collect` / `collectAsStateWithLifecycle()` (lesen).

---

## 5. ROOM komplett

### 5.1 Entity (= eine Tabelle)
```kotlin
import androidx.room.*

@Entity(tableName = "parties")
data class Party(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "party_name") val name: String,
    val votes: Int = 0
)
```

### 5.2 DAO (= Datenbankzugriff)
```kotlin
import androidx.room.*
import kotlinx.coroutines.flow.Flow

@Dao
interface PartyDao {

    // LESEN: Flow -> UI reagiert automatisch auf jede Änderung. KEIN suspend nötig.
    @Query("SELECT * FROM parties ORDER BY votes DESC")
    fun observeAll(): Flow<List<Party>>

    @Query("SELECT * FROM parties WHERE id = :id")
    fun observeById(id: Long): Flow<Party?>

    @Query("SELECT COALESCE(SUM(votes), 0) FROM parties")
    fun observeTotal(): Flow<Int>

    // Einmal-Abfrage (kein Stream): suspend
    @Query("SELECT * FROM parties WHERE id = :id")
    suspend fun getById(id: Long): Party?

    // SCHREIBEN: immer suspend (läuft off-main-thread)
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(party: Party): Long

    @Insert
    suspend fun insertAll(parties: List<Party>)

    @Update
    suspend fun update(party: Party)

    @Delete
    suspend fun delete(party: Party)

    @Upsert                       // Insert ODER Update (praktisch beim Cachen)
    suspend fun upsertAll(parties: List<Party>)

    @Query("UPDATE parties SET votes = MAX(0, votes + :delta) WHERE id = :id")
    suspend fun changeVotes(id: Long, delta: Int)

    @Query("DELETE FROM parties")
    suspend fun clear()

    // Transaktion (mehrere Schritte atomar):
    @Transaction
    suspend fun replaceAll(parties: List<Party>) {
        clear()
        insertAll(parties)
    }
}
```

### 5.3 Database (Singleton)
```kotlin
import android.content.Context
import androidx.room.*

@Database(entities = [Party::class], version = 1, exportSchema = false)
abstract class AppDatabase : RoomDatabase() {

    abstract fun partyDao(): PartyDao

    companion object {
        @Volatile private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase =
            INSTANCE ?: synchronized(this) {
                INSTANCE ?: Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app.db"
                )
                // .fallbackToDestructiveMigration() // bei Schemaänderung DB neu (Daten weg)
                .build()
                .also { INSTANCE = it }
            }
    }
}
```

### 5.4 TypeConverter (für Typen, die SQLite nicht kennt, z. B. Date/List)
```kotlin
class Converters {
    @TypeConverter fun fromTimestamp(v: Long?): Date? = v?.let { Date(it) }
    @TypeConverter fun dateToTimestamp(d: Date?): Long? = d?.time
}
// in der @Database-Klasse:
@Database(/* ... */)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() { /* ... */ }
```

### 5.5 Relationen (1:n)
```kotlin
@Entity data class User(@PrimaryKey val id: Long, val name: String)

@Entity data class Post(
    @PrimaryKey val id: Long,
    val userId: Long,        // Fremdschlüssel
    val title: String
)

data class UserWithPosts(
    @Embedded val user: User,
    @Relation(parentColumn = "id", entityColumn = "userId")
    val posts: List<Post>
)

@Dao interface UserDao {
    @Transaction
    @Query("SELECT * FROM User")
    fun getUsersWithPosts(): Flow<List<UserWithPosts>>
}
```

### 5.6 Migration (Schema ändern, ohne Daten zu verlieren)
```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE parties ADD COLUMN color TEXT NOT NULL DEFAULT '#000000'")
    }
}
Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2)
    .build()
```

---

## 6. RETROFIT komplett

### 6.1 Datenmodell (DTO) mit kotlinx.serialization
```kotlin
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

@Serializable
data class UserDto(
    val id: Long,
    val name: String,
    @SerialName("email_address") val email: String,   // JSON-Feldname != Kotlin-Name
    val phone: String? = null                          // optional → nullable + Default
)
```

### 6.2 API-Interface (alle HTTP-Annotationen)
```kotlin
import retrofit2.Response
import retrofit2.http.*

interface ApiService {

    @GET("users")
    suspend fun getUsers(): List<UserDto>

    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Long): UserDto

    // Query-Parameter:  GET users?page=2&limit=20
    @GET("users")
    suspend fun getUsersPaged(@Query("page") page: Int, @Query("limit") limit: Int): List<UserDto>

    // Response<T> = Zugriff auf Statuscode/Header/Fehler
    @GET("users/{id}")
    suspend fun getUserResponse(@Path("id") id: Long): Response<UserDto>

    // POST mit JSON-Body
    @POST("users")
    suspend fun createUser(@Body user: UserDto): UserDto

    @PUT("users/{id}")
    suspend fun updateUser(@Path("id") id: Long, @Body user: UserDto): UserDto

    @PATCH("users/{id}")
    suspend fun patchUser(@Path("id") id: Long, @Body fields: Map<String, String>): UserDto

    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: Long): Response<Unit>

    // Header
    @GET("profile")
    suspend fun profile(@Header("Authorization") token: String): UserDto

    @Headers("Accept: application/json")
    @GET("users")
    suspend fun getUsersWithHeader(): List<UserDto>

    // Form-URL-encoded (z. B. Login)
    @FormUrlEncoded
    @POST("login")
    suspend fun login(@Field("user") user: String, @Field("pass") pass: String): TokenDto

    // Datei-Upload (Multipart)
    @Multipart
    @POST("upload")
    suspend fun upload(@Part file: okhttp3.MultipartBody.Part): Response<Unit>
}
```

### 6.3 Retrofit-Instanz bauen (mit OkHttp-Logging + Auth-Interceptor)
```kotlin
import com.jakewharton.retrofit2.converter.kotlinx.serialization.asConverterFactory
import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit

object NetworkModule {

    private val json = Json {
        ignoreUnknownKeys = true   // unbekannte JSON-Felder ignorieren (sehr wichtig!)
        coerceInputValues = true
    }

    private val logging = HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY   // im Debug; sonst NONE
    }

    private val client = OkHttpClient.Builder()
        .addInterceptor(logging)
        // Beispiel Auth-Header automatisch anhängen:
        .addInterceptor { chain ->
            val req = chain.request().newBuilder()
                .addHeader("Authorization", "Bearer TOKEN")
                .build()
            chain.proceed(req)
        }
        .build()

    val api: ApiService = Retrofit.Builder()
        .baseUrl("https://api.example.com/")          // MUSS mit / enden!
        .client(client)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()
        .create(ApiService::class.java)
}
```

> **Alternative Converter:** Moshi → `MoshiConverterFactory.create()` (Dep `converter-moshi`), Gson → `GsonConverterFactory.create()` (Dep `converter-gson`).

### 6.4 Fehlerbehandlung — sauberes `Result`-Pattern
```kotlin
sealed interface ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>
    data class Error(val message: String) : ApiResult<Nothing>
}

suspend fun <T> safeCall(call: suspend () -> T): ApiResult<T> = try {
    ApiResult.Success(call())
} catch (e: retrofit2.HttpException) {     // 4xx/5xx
    ApiResult.Error("HTTP ${e.code()}")
} catch (e: java.io.IOException) {         // kein Netz / Timeout
    ApiResult.Error("Keine Verbindung")
} catch (e: Exception) {
    ApiResult.Error(e.message ?: "Unbekannter Fehler")
}

// Aufruf:
when (val r = safeCall { api.getUsers() }) {
    is ApiResult.Success -> { /* r.data */ }
    is ApiResult.Error   -> { /* r.message anzeigen */ }
}
```

---

## 7. Repository

> Die **einzige** Klasse, die Room **und** Retrofit kennt. ViewModels greifen NUR hierauf zu.

```kotlin
class UserRepository(
    private val dao: UserDao,
    private val api: ApiService
) {
    // Lesen: immer aus Room (Single Source of Truth) → Flow
    val users: Flow<List<User>> = dao.observeAll()

    // Schreiben: sofort in Room
    suspend fun add(user: User) = dao.insert(user)
    suspend fun delete(user: User) = dao.delete(user)

    // Netzwerk → in Room speichern (UI aktualisiert sich über den Flow von selbst)
    suspend fun refreshFromNetwork(): ApiResult<Unit> = safeCall {
        val dtos = api.getUsers()                 // 1. vom Server holen
        val entities = dtos.map { it.toEntity() } // 2. DTO -> Entity mappen
        dao.replaceAll(entities)                  // 3. in Room schreiben (Transaktion)
    }
}

// Mapping DTO <-> Entity (Schichten sauber trennen!)
fun UserDto.toEntity() = User(id = id, name = name, email = email)
```

---

## 8. Die MISCHUNG: offline-first

> **Das ist das typische Test-Szenario.** Idee: Die UI liest **immer aus Room**. Das Netzwerk dient nur dazu, Room zu **befüllen/aktualisieren**. Vorteile: App funktioniert offline, sofortige Anzeige, eine Wahrheitsquelle.

```
            refresh()                       Flow
   ┌────────────────────┐        ┌───────────────────────┐
   │ Retrofit (Server)  │ ─────▶ │ Room (lokale DB)      │ ─────▶ ViewModel ─▶ UI
   └────────────────────┘  save  └───────────────────────┘ observe
        (suspend, IO)                 (Single Source of Truth)
```

### Variante A — einfach & klar (empfohlen für den Test)
```kotlin
class UserRepository(private val dao: UserDao, private val api: ApiService) {

    // 1) UI beobachtet IMMER die DB:
    val users: Flow<List<User>> = dao.observeAll()

    // 2) Refresh holt vom Netz und speichert in die DB:
    suspend fun refresh(): ApiResult<Unit> = safeCall {
        val fresh = api.getUsers().map { it.toEntity() }
        dao.replaceAll(fresh)   // Room sendet danach automatisch über den Flow
    }
}

// ViewModel
class UserViewModel(private val repo: UserRepository) : ViewModel() {
    val users = repo.users.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    var isLoading by mutableStateOf(false); private set
    var error by mutableStateOf<String?>(null); private set

    init { refresh() }   // beim Start einmal laden

    fun refresh() = viewModelScope.launch {
        isLoading = true; error = null
        when (val r = repo.refresh()) {
            is ApiResult.Success -> {}                 // UI kommt über den Flow
            is ApiResult.Error   -> error = r.message
        }
        isLoading = false
    }
}
```

### Variante B — `NetworkBoundResource` (DB + Netz als ein Flow)
```kotlin
// Liefert zuerst den Cache, lädt parallel neu, speichert, und gibt Lade-/Fehlerzustand mit aus.
fun <T> networkBoundResource(
    query: () -> Flow<T>,                 // aus Room lesen
    fetch: suspend () -> T,               // aus dem Netz holen  (hier vereinfacht)
    saveFetch: suspend (T) -> Unit        // in Room speichern
): Flow<Resource<T>> = flow {
    emit(Resource.Loading)
    val cached = query().first()
    emit(Resource.Success(cached))        // sofort Cache zeigen
    try {
        saveFetch(fetch())                // Netz -> Room
    } catch (e: Exception) {
        emit(Resource.Error(e.message ?: "Fehler"))
    }
    // ab jetzt live aus der DB:
    emitAll(query().map { Resource.Success(it) })
}

sealed interface Resource<out T> {
    data object Loading : Resource<Nothing>
    data class Success<T>(val data: T) : Resource<T>
    data class Error(val message: String) : Resource<Nothing>
}
```

---

## 8★. TEST-SZENARIO: Lesen → Berechnen → Speichern

> **Genau das hat der Lehrer angekündigt.** Drei Schritte, immer in **einer** Coroutine:

```
   1. LESEN                 2. BERECHNEN                3. SPEICHERN
┌──────────────┐        ┌────────────────────┐      ┌──────────────┐
│ api.getX()   │  ───▶  │ reine Kotlin-Logik │ ───▶ │ dao.insert() │
│ (Retrofit)   │        │ summe/avg/filter…  │      │ (Room)       │
│  oder Eingabe│        │ (KEIN Netz/Android)│      │              │
└──────────────┘        └────────────────────┘      └──────────────┘
        alles in  viewModelScope.launch { ... }  bzw. einer suspend-Funktion
```

**Die 3 goldenen Regeln:**
1. **Lesen** ist `suspend` (Netz) → in einer Coroutine aufrufen.
2. **Berechnen** in eine **reine Funktion** auslegen (keine IO, kein Context) → sauber + unit-testbar.
3. **Speichern** ist `suspend` (Room) → UI sieht das Ergebnis automatisch über den `Flow`.

### 🧩 TEMPLATE zum Ausfüllen (auswendig können!)
```kotlin
class DataRepository(private val dao: ResultDao, private val api: ApiService) {

    // UI liest IMMER aus Room:
    val results: Flow<List<ResultEntity>> = dao.observeAll()

    // 1+2+3 in EINER suspend-Funktion:
    suspend fun ladeBerechneSpeichere(): ApiResult<Unit> = safeCall {
        val roh    = api.getItems()        // 1) LESEN  (Retrofit, suspend)
        val ergebnis = berechne(roh)       // 2) BERECHNEN (reine Logik)
        dao.insert(ergebnis)               // 3) SPEICHERN (Room, suspend)
    }

    // reine Funktion -> keine Abhängigkeiten -> leicht testbar:
    fun berechne(items: List<ItemDto>): ResultEntity {
        val summe = items.sumOf { it.value }
        return ResultEntity(
            summe        = summe,
            durchschnitt = if (items.isEmpty()) 0.0 else summe / items.size,
            anzahl       = items.size
        )
    }
}
```

### ✅ KOMPLETTBEISPIEL: Transaktionen lesen → Summe/Ø berechnen → speichern
```kotlin
// ---- DTO (was vom Server kommt) ----
@Serializable
data class TransaktionDto(val id: Long, val betrag: Double, val kategorie: String)

// ---- ENTITIES (was in die DB kommt) ----
@Entity(tableName = "transaktionen")
data class TransaktionEntity(@PrimaryKey val id: Long, val betrag: Double, val kategorie: String)

@Entity(tableName = "report")
data class ReportEntity(
    @PrimaryKey val id: Int = 1,          // immer eine Zeile -> feste id
    val gesamtsumme: Double,
    val durchschnitt: Double,
    val anzahl: Int
)

// ---- DAO ----
@Dao interface FinanzDao {
    @Query("SELECT * FROM transaktionen") fun observeTransaktionen(): Flow<List<TransaktionEntity>>
    @Query("SELECT * FROM report WHERE id = 1") fun observeReport(): Flow<ReportEntity?>
    @Upsert suspend fun upsertTransaktionen(list: List<TransaktionEntity>)
    @Upsert suspend fun upsertReport(report: ReportEntity)
}

// ---- API ----
interface FinanzApi { @GET("transaktionen") suspend fun getTransaktionen(): List<TransaktionDto> }

// ---- REPOSITORY: lesen -> rechnen -> speichern ----
class FinanzRepository(private val dao: FinanzDao, private val api: FinanzApi) {

    val report: Flow<ReportEntity?> = dao.observeReport()
    val transaktionen: Flow<List<TransaktionEntity>> = dao.observeTransaktionen()

    suspend fun aktualisieren(): ApiResult<Unit> = safeCall {
        // 1) LESEN
        val dtos = api.getTransaktionen()
        // 2) BERECHNEN
        val entities = dtos.map { TransaktionEntity(it.id, it.betrag, it.kategorie) }
        val report = berechneReport(entities)
        // 3) SPEICHERN (beide Tabellen)
        dao.upsertTransaktionen(entities)
        dao.upsertReport(report)
    }

    fun berechneReport(list: List<TransaktionEntity>): ReportEntity {
        val summe = list.sumOf { it.betrag }
        return ReportEntity(
            gesamtsumme  = summe,
            durchschnitt = if (list.isEmpty()) 0.0 else summe / list.size,
            anzahl       = list.size
        )
    }
}

// ---- VIEWMODEL ----
class FinanzViewModel(private val repo: FinanzRepository) : ViewModel() {
    val report = repo.report.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)
    var fehler by mutableStateOf<String?>(null); private set

    init { aktualisieren() }
    fun aktualisieren() = viewModelScope.launch {
        when (val r = repo.aktualisieren()) {
            is ApiResult.Success -> fehler = null   // UI kommt über den Flow
            is ApiResult.Error   -> fehler = r.message
        }
    }
}

// ---- UI ----
@Composable
fun ReportScreen(vm: FinanzViewModel) {
    val report by vm.report.collectAsStateWithLifecycle()
    Column(Modifier.padding(16.dp)) {
        Button(onClick = vm::aktualisieren) { Text("Laden & Berechnen") }
        Spacer(Modifier.height(16.dp))
        report?.let {
            Text("Summe: ${it.gesamtsumme} €")
            Text("Durchschnitt: ${"%.2f".format(it.durchschnitt)} €")
            Text("Anzahl: ${it.anzahl}")
        } ?: Text("Noch keine Daten")
    }
}
```

### 🔁 Variante: „Lesen" = Benutzer-Eingabe (statt Netz)
```kotlin
// UI: zwei Textfelder -> berechnen -> speichern
class RechnerViewModel(private val dao: ErgebnisDao) : ViewModel() {
    fun berechneUndSpeichere(aText: String, bText: String) = viewModelScope.launch {
        val a = aText.toDoubleOrNull() ?: 0.0          // 1) LESEN/parsen
        val b = bText.toDoubleOrNull() ?: 0.0
        val ergebnis = a + b                            // 2) BERECHNEN
        dao.insert(ErgebnisEntity(summe = ergebnis))    // 3) SPEICHERN
    }
}
```

### 🧮 Berechnungs-Spickzettel (Kotlin-Collections)
```kotlin
list.sumOf { it.betrag }                       // Summe
list.map { it.betrag }.average()               // Durchschnitt (Achtung: leere Liste -> NaN!)
list.maxByOrNull { it.betrag }                 // größtes Element
list.minByOrNull { it.betrag }
list.count { it.betrag > 100 }                 // zählen mit Bedingung
list.filter { it.kategorie == "Essen" }        // filtern
list.groupBy { it.kategorie }                  // Map<String, List<...>>
list.groupBy { it.kategorie }
    .mapValues { (_, v) -> v.sumOf { it.betrag } }  // Summe pro Kategorie
list.sortedByDescending { it.betrag }          // sortieren
list.associateBy { it.id }                     // Map<Id, Item>
```

### 🧪 Mini-Unit-Test der Berechnung (weil reine Funktion!)
```kotlin
@Test fun berechneReport_summe_und_durchschnitt() {
    val repo = FinanzRepository(dao = FakeDao(), api = FakeApi())
    val r = repo.berechneReport(listOf(
        TransaktionEntity(1, 10.0, "a"),
        TransaktionEntity(2, 20.0, "b"),
    ))
    assertEquals(30.0, r.gesamtsumme, 0.001)
    assertEquals(15.0, r.durchschnitt, 0.001)
    assertEquals(2, r.anzahl)
}
```

> **Merksatz für genau diese Aufgabe:** *„Ein Klick startet eine Coroutine. Die liest per `suspend` (Retrofit oder Eingabe), übergibt die Daten an eine reine Berechnungsfunktion, und speichert das Ergebnis per `suspend` in Room. Die UI zeigt das Ergebnis über den `Flow` automatisch an."*

---

## 8★★. Mehrere DAOs / mehrere Datenbanken

> ⚠️ **Begriffe klären:** Oft sagt man „2 Datenbanken", meint aber **2 Tabellen (Entities) mit 2 DAOs in EINER Room-Datenbank**. Es kann aber auch wirklich **2 getrennte Datenbanken** (2 `@Database`-Klassen, 2 `.db`-Dateien) sein. Hier beide Fälle — geh im Zweifel von **Fall A** aus (das ist der saubere Standard).

### ✅ FALL A — eine Datenbank, mehrere Entities + mehrere DAOs (Standard!)
```kotlin
// Mehrere Entities anmelden, mehrere DAOs bereitstellen:
@Database(
    entities = [User::class, Product::class, OrderEntity::class],
    version = 1,
    exportSchema = false
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao        // DAO 1
    abstract fun productDao(): ProductDao  // DAO 2
    abstract fun orderDao(): OrderDao      // DAO 3

    companion object {
        @Volatile private var I: AppDatabase? = null
        fun getInstance(c: Context) = I ?: synchronized(this) {
            I ?: Room.databaseBuilder(c.applicationContext, AppDatabase::class.java, "app.db")
                .build().also { I = it }
        }
    }
}
```
**Vorteile von Fall A:** Tabellen können sich per **Fremdschlüssel/`@Relation`** verknüpfen, **JOINs** sind möglich, eine **`@Transaction`** kann über mehrere Tabellen atomar laufen.

### 🅱️ FALL B — zwei getrennte Datenbanken (2 `@Database`, 2 Dateien)
> Sinnvoll, wenn die Datenbereiche wirklich unabhängig sind (z. B. eine mitgelieferte Lese-DB + eine Benutzer-DB).

```kotlin
@Database(entities = [User::class], version = 1, exportSchema = false)
abstract class UserDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    companion object {
        @Volatile private var I: UserDatabase? = null
        fun getInstance(c: Context) = I ?: synchronized(this) {
            I ?: Room.databaseBuilder(c.applicationContext, UserDatabase::class.java, "users.db")
                .build().also { I = it }                          // ← Dateiname 1
        }
    }
}

@Database(entities = [Product::class], version = 1, exportSchema = false)
abstract class ProductDatabase : RoomDatabase() {
    abstract fun productDao(): ProductDao
    companion object {
        @Volatile private var I: ProductDatabase? = null
        fun getInstance(c: Context) = I ?: synchronized(this) {
            I ?: Room.databaseBuilder(c.applicationContext, ProductDatabase::class.java, "products.db")
                .build().also { I = it }                          // ← Dateiname 2 (MUSS anders sein!)
        }
    }
}
```
> ❗ **Achtung bei Fall B:** Die beiden DB-Dateinamen **müssen unterschiedlich** sein. Über zwei getrennte DBs sind **keine** Fremdschlüssel/`@Relation`/JOINs und **keine** gemeinsame Transaktion möglich — Verknüpfen passiert dann erst im Repository (siehe unten).

### 🔌 Wiring: beide DAOs ins Repository (Application = Container)
```kotlin
class MyApplication : Application() {
    // Fall A:
    private val db by lazy { AppDatabase.getInstance(this) }
    val repository by lazy { ShopRepository(db.userDao(), db.orderDao(), Network.api) }

    // Fall B (alternativ): zwei DBs
    // private val userDb by lazy { UserDatabase.getInstance(this) }
    // private val productDb by lazy { ProductDatabase.getInstance(this) }
    // val repository by lazy { ShopRepository(userDb.userDao(), productDb.productDao(), Network.api) }
}
```

### 🧮 Repository mit ZWEI DAOs — lesen, berechnen, in beide speichern, kombinieren
```kotlin
class ShopRepository(
    private val userDao: UserDao,
    private val orderDao: OrderDao,
    private val api: ApiService
) {
    val users: Flow<List<User>>   = userDao.observeAll()
    val orders: Flow<List<Order>> = orderDao.observeAll()

    // ZWEI Flows zu einem UI-Modell kombinieren (das "etc." aus der Angabe):
    val userSummaries: Flow<List<UserSummary>> =
        combine(users, orders) { us, os ->
            us.map { u ->
                val eigene = os.filter { it.userId == u.id }       // berechnen
                UserSummary(name = u.name, anzahl = eigene.size, summe = eigene.sumOf { it.betrag })
            }
        }

    // Lesen -> berechnen -> in BEIDE DAOs speichern:
    suspend fun sync(): ApiResult<Unit> = safeCall {
        val dto = api.getShopData()                                // 1) LESEN
        val users  = dto.users.map  { it.toUserEntity() }          // 2) MAPPEN/BERECHNEN
        val orders = dto.orders.map { it.toOrderEntity() }
        userDao.upsertAll(users)                                   // 3) SPEICHERN (DAO 1)
        orderDao.upsertAll(orders)                                 //    SPEICHERN (DAO 2)
    }
}

data class UserSummary(val name: String, val anzahl: Int, val summe: Double)
```

### Welcher Fall wann?
| | Fall A: 1 DB, mehrere DAOs | Fall B: 2 getrennte DBs |
|---|---|---|
| Verknüpfung der Tabellen | ✅ Fremdschlüssel, `@Relation`, JOIN | ❌ nur im Repo per `combine` |
| Gemeinsame Transaktion | ✅ `@Transaction` über alle Tabellen | ❌ getrennt |
| Wann nehmen? | **Standard** – Daten gehören zusammen | Daten wirklich unabhängig / getrennte Module |

> **Merksatz:** *„Mehrere DAOs" heißt meist: eine `@Database` mit mehreren `entities` und je einer `abstract fun xDao()`. Das Repository bekommt die DAOs, die es braucht, und kombiniert mehrere `Flow`s bei Bedarf mit `combine`.*

---

## 9. ViewModel + UiState

### Sauberer UI-State per `sealed interface`
```kotlin
sealed interface UsersUiState {
    data object Loading : UsersUiState
    data class Success(val users: List<User>) : UsersUiState
    data class Error(val message: String) : UsersUiState
}

class UsersViewModel(private val repo: UserRepository) : ViewModel() {

    val uiState: StateFlow<UsersUiState> =
        repo.users
            .map<List<User>, UsersUiState> { UsersUiState.Success(it) }
            .catch { emit(UsersUiState.Error(it.message ?: "Fehler")) }
            .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), UsersUiState.Loading)

    fun onRefresh() = viewModelScope.launch { repo.refresh() }
    fun onDelete(u: User) = viewModelScope.launch { repo.delete(u) }
}
```

### ViewModel-Factory ohne Hilt (Argumente reingeben)
```kotlin
class UsersViewModelFactory(private val repo: UserRepository) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        @Suppress("UNCHECKED_CAST")
        return UsersViewModel(repo) as T
    }
}
// moderne Variante per viewModelFactory { initializer { ... } } siehe Abschnitt 11.
```

---

## 10. Compose-UI anbinden

```kotlin
@Composable
fun UsersScreen(viewModel: UsersViewModel) {
    // collectAsStateWithLifecycle: liest StateFlow lebenszyklus-sicher
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    when (val s = state) {
        is UsersUiState.Loading -> CircularProgressIndicator()
        is UsersUiState.Error   -> Text("Fehler: ${s.message}")
        is UsersUiState.Success -> {
            LazyColumn {
                items(s.users, key = { it.id }) { user ->
                    ListItem(
                        headlineContent = { Text(user.name) },
                        supportingContent = { Text(user.email) },
                        trailingContent = {
                            TextButton(onClick = { viewModel.onDelete(user) }) { Text("Löschen") }
                        }
                    )
                    HorizontalDivider()
                }
            }
        }
    }
}
```
> `import androidx.lifecycle.compose.collectAsStateWithLifecycle` (Dep `lifecycle-runtime-compose`).

---

## 10★. NAVIGATION 3 (Nav3)

> ⚠️ **Das ist NICHT das alte navigation-compose** (`NavHost` + `composable("route")` + `rememberNavController`). Nav3 (`androidx.navigation3`) arbeitet mit **typsicheren NavKey-Objekten**, einem **Back-Stack als Liste** und **`NavDisplay`** statt String-Routen. Genau so läuft dein ElectorApp.

**Dependencies (schon in Abschnitt 2):** `navigation3-runtime` + `navigation3-ui` + `kotlinx-serialization-core` + Plugin `jetbrains-kotlin-serialization` (die Keys sind `@Serializable`).

### 1) NavKeys = die Ziele (eigene Datei `NavKeys.kt`)
```kotlin
import androidx.navigation3.runtime.NavKey
import kotlinx.serialization.Serializable

@Serializable data object Home     : NavKey   // Ziel OHNE Parameter -> data object
@Serializable data object Overview : NavKey
@Serializable data object About    : NavKey
@Serializable data class  Count(val partyId: Long) : NavKey   // Ziel MIT Parameter -> data class
```
> **Regel:** Nur die *bedeutungstragende* ID als Parameter mitgeben (z.B. `partyId`). Name/Stimmen NICHT mitschleppen — die kommen im Ziel-Screen aus der DB (Flow). Hält die Navigation minimal und die Anzeige DB-konsistent.

### 2) NavDisplay + Back-Stack
```kotlin
import androidx.compose.runtime.Composable
import androidx.compose.ui.platform.LocalContext
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.lifecycle.viewmodel.initializer
import androidx.lifecycle.viewmodel.viewModelFactory
import androidx.navigation3.runtime.entryProvider
import androidx.navigation3.runtime.rememberNavBackStack
import androidx.navigation3.ui.NavDisplay

@Composable
fun AppNavDisplay() {
    val backStack = rememberNavBackStack(Home)             // Start-Ziel
    // von überall erreichbar, ohne About doppelt zu stapeln:
    val goAbout = { if (backStack.lastOrNull() != About) backStack.add(About) }

    NavDisplay(
        backStack = backStack,
        onBack = { backStack.removeLastOrNull() },         // System-Zurück-Geste
        entryProvider = entryProvider {

            entry<Home> {
                HomeScreen(
                    onOverview = { backStack.add(Overview) },
                    onAbout    = goAbout)
            }

            entry<Overview> {
                OverviewScreen(
                    onPartyClick = { party -> backStack.add(Count(party.id)) },  // Parameter mitgeben!
                    onBack  = { backStack.removeLastOrNull() },
                    onAbout = goAbout)
            }

            entry<Count> { key ->                          // key = der NavKey -> key.partyId auslesen
                // ViewModel, das den Nav-Parameter braucht: eigene Factory + eigener key
                val app = LocalContext.current.applicationContext as MyApplication
                val vm: CountViewModel = viewModel(
                    key = "count_${key.partyId}",
                    factory = viewModelFactory {
                        initializer { CountViewModel(app.repository, key.partyId) }
                    })
                CountScreen(viewModel = vm, onBack = { backStack.removeLastOrNull() }, onAbout = goAbout)
            }

            entry<About> { AboutScreen(onBack = { backStack.removeLastOrNull() }) }
        }
    )
}
```
```kotlin
// MainActivity: setContent { DeinTheme { AppNavDisplay() } }
```

### 🧭 Spickzettel Nav3
| Aktion | Code |
|---|---|
| Vorwärts navigieren | `backStack.add(Ziel)` |
| Zurück | `backStack.removeLastOrNull()` |
| Start-Ziel setzen | `rememberNavBackStack(Home)` |
| Aktuellen Screen prüfen | `backStack.lastOrNull()` |
| Ziel ohne Parameter | `@Serializable data object X : NavKey` |
| Ziel **mit** Parameter | `@Serializable data class X(val id: Long): NavKey` → `add(X(id))` → `entry<X> { key -> key.id }` |
| Screen anzeigen | `entry<X> { ... }` im `entryProvider { }` |

> **Merksätze:** *NavKey statt String-Route. `add()` statt `navigate()`. `removeLastOrNull()` statt `popBackStack()`. Parameter = Feld im `data class`-Key, im `entry<X> { key -> ... }` auslesen.*

---

## 11. Dependency Injection

### 11a. Manuell (ohne Bibliothek) — wie in „Der größere Wahlhelfer"
```kotlin
// Application = zentraler Container
class MyApplication : Application() {
    val database by lazy { AppDatabase.getInstance(this) }
    val api by lazy { NetworkModule.api }
    val repository by lazy { UserRepository(database.userDao(), api) }
}
// im Manifest:  <application android:name=".MyApplication" ... >

// ViewModel das Repository per Factory geben:
object AppViewModelProvider {
    fun factory(app: MyApplication) = viewModelFactory {
        initializer { UsersViewModel(app.repository) }
    }
}

// im Composable:
val app = LocalContext.current.applicationContext as MyApplication
val vm: UsersViewModel = viewModel(factory = AppViewModelProvider.factory(app))
```

### 11b. Hilt (annotationsbasiert — falls erlaubt/gefordert)
```kotlin
// build.gradle.kts: plugin "com.google.dagger.hilt.android" + ksp("com.google.dagger:hilt-compiler:…")

@HiltAndroidApp
class MyApplication : Application()

@Module
@InstallIn(SingletonComponent::class)
object DataModule {
    @Provides @Singleton
    fun provideDb(@ApplicationContext ctx: Context): AppDatabase =
        Room.databaseBuilder(ctx, AppDatabase::class.java, "app.db").build()

    @Provides fun provideDao(db: AppDatabase): UserDao = db.userDao()

    @Provides @Singleton
    fun provideApi(): ApiService = NetworkModule.api

    @Provides @Singleton
    fun provideRepo(dao: UserDao, api: ApiService) = UserRepository(dao, api)
}

@HiltViewModel
class UsersViewModel @Inject constructor(
    private val repo: UserRepository
) : ViewModel() { /* ... */ }

@AndroidEntryPoint
class MainActivity : ComponentActivity() { /* ... */ }

// im Composable:  val vm: UsersViewModel = hiltViewModel()
```

---

## 12. Komplettbeispiel end-to-end

> **Mini-App:** Nutzer vom Server laden, in Room cachen, in Compose anzeigen, offline-fähig. Alle Bausteine in Reihenfolge.

```kotlin
// ---------- 1. ENTITY ----------
@Entity(tableName = "users")
data class User(@PrimaryKey val id: Long, val name: String, val email: String)

// ---------- 2. DAO ----------
@Dao interface UserDao {
    @Query("SELECT * FROM users ORDER BY name") fun observeAll(): Flow<List<User>>
    @Upsert suspend fun upsertAll(users: List<User>)
    @Query("DELETE FROM users") suspend fun clear()
    @Delete suspend fun delete(user: User)
    @Transaction suspend fun replaceAll(users: List<User>) { clear(); upsertAll(users) }
}

// ---------- 3. DATABASE ----------
@Database(entities = [User::class], version = 1, exportSchema = false)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    companion object {
        @Volatile private var I: AppDatabase? = null
        fun get(c: Context) = I ?: synchronized(this) {
            I ?: Room.databaseBuilder(c.applicationContext, AppDatabase::class.java, "app.db")
                .build().also { I = it }
        }
    }
}

// ---------- 4. RETROFIT (DTO + API + Instanz) ----------
@Serializable data class UserDto(val id: Long, val name: String, val email: String)

interface ApiService { @GET("users") suspend fun getUsers(): List<UserDto> }

object Network {
    private val json = Json { ignoreUnknownKeys = true }
    val api: ApiService = Retrofit.Builder()
        .baseUrl("https://jsonplaceholder.typicode.com/")
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build().create(ApiService::class.java)
}

// ---------- 5. REPOSITORY (die Mischung) ----------
class UserRepository(private val dao: UserDao, private val api: ApiService) {
    val users: Flow<List<User>> = dao.observeAll()                 // lesen: aus Room
    suspend fun delete(u: User) = dao.delete(u)
    suspend fun refresh() {                                        // schreiben: Netz -> Room
        val fresh = api.getUsers().map { User(it.id, it.name, it.email) }
        dao.replaceAll(fresh)
    }
}

// ---------- 6. APPLICATION (manuelle DI) ----------
class MyApplication : Application() {
    val repository by lazy { UserRepository(AppDatabase.get(this).userDao(), Network.api) }
}

// ---------- 7. VIEWMODEL ----------
class UsersViewModel(private val repo: UserRepository) : ViewModel() {
    val users = repo.users.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
    var error by mutableStateOf<String?>(null); private set
    init { load() }
    fun load() = viewModelScope.launch {
        try { repo.refresh() } catch (e: Exception) { error = e.message }
    }
    fun delete(u: User) = viewModelScope.launch { repo.delete(u) }
}

// ---------- 8. UI ----------
@Composable
fun UsersScreen() {
    val app = LocalContext.current.applicationContext as MyApplication
    val vm: UsersViewModel = viewModel(
        factory = viewModelFactory { initializer { UsersViewModel(app.repository) } }
    )
    val users by vm.users.collectAsStateWithLifecycle()

    Scaffold(topBar = { TopAppBar(title = { Text("Users") },
        actions = { TextButton(onClick = vm::load) { Text("Refresh") } }) }
    ) { p ->
        LazyColumn(Modifier.padding(p)) {
            items(users, key = { it.id }) { u ->
                ListItem(
                    headlineContent = { Text(u.name) },
                    supportingContent = { Text(u.email) },
                    trailingContent = { TextButton(onClick = { vm.delete(u) }) { Text("X") } }
                )
            }
        }
    }
}
```

---

## 13. Häufige Fehler / Fallen

| Symptom | Ursache | Lösung |
|---|---|---|
| `SecurityException` / kein Netz | INTERNET-Permission fehlt | `<uses-permission android:name="android.permission.INTERNET"/>` |
| `CLEARTEXT communication not permitted` | `http://` statt `https://` | `https` nutzen oder `usesCleartextTraffic="true"` (nur Test) |
| Crash: `IllegalArgumentException: baseUrl must end in /` | baseUrl ohne `/` | `"https://api.x.com/"` |
| Room-Klassen werden nicht generiert | `ksp(...)`-Compiler fehlt / falsches Plugin | `ksp(libs.androidx.room.compiler)` + `alias(libs.plugins.ksp)` |
| `Cannot access database on the main thread` | DB-Aufruf ohne suspend/Flow | Schreiben `suspend`, Lesen `Flow`, in `viewModelScope` |
| JSON-Feld kommt nicht an | Feldname ≠ Kotlin-Property | `@SerialName("json_name")` |
| Crash bei unbekanntem JSON-Feld | strikte Deserialisierung | `Json { ignoreUnknownKeys = true }` |
| UI aktualisiert sich nicht | nicht reaktiv beobachtet | `collectAsStateWithLifecycle()` + DAO gibt `Flow` zurück |
| App stürzt bei Rotation / Daten weg | State in Composable statt ViewModel | State ins `ViewModel` |
| `@Serializable`-Fehler | Serialization-Plugin fehlt | `alias(libs.plugins.kotlin.serialization)` |

**Merksätze:**
- **Lesen aus Room → `Flow` (kein suspend). Schreiben → `suspend`.**
- **UI liest immer aus der DB; das Netz befüllt nur die DB.** (Single Source of Truth)
- **DTO (Netz) ≠ Entity (DB) ≠ UI-Model** — mit `toEntity()`/`toUi()` mappen.
- **Jeder Netz-/DB-Aufruf läuft in einer Coroutine** (`viewModelScope.launch` / `suspend`).

---

## 13★. Versionen aktuell halten / updaten

> Ziel: neueste **stabile** Versionen (für benotete Abgaben **keine** alpha/beta/rc). „Möglichst aktuell" = aktuell **stabil**.

### Was hängt zusammen? (Kompatibilitäts-Kette)
```
Gradle  ──min──▶  AGP  ◀──────  Kotlin  ──muss passen──▶  KSP
                                  │
                                  └──(Compose-Plugin folgt Kotlin automatisch)
Compose BOM ──steuert──▶ alle Compose-Libs (ui, material3, foundation …)
```
- **Gradle ↔ AGP:** jede AGP-Version braucht eine **Mindest-Gradle-Version**.
- **Kotlin ↔ KSP:** die KSP-Version **muss zur Kotlin-Version passen**.
- **Kotlin ↔ Compose-Compiler:** ab Kotlin 2.0 erledigt das **`org.jetbrains.kotlin.plugin.compose`**-Plugin automatisch (keine separate Compose-Compiler-Version mehr).
- **Compose BOM:** **eine** BOM-Version pinnen → alle Compose-Bibliotheken bekommen automatisch zusammenpassende Versionen. Nur die BOM hochziehen, nicht jede Lib einzeln.

### Reihenfolge beim Updaten
1. **Gradle** (Wrapper) → 2. **AGP** → 3. **Kotlin** (+ KSP + Compose-Plugin) → 4. **AndroidX-Libs / Compose BOM** → 5. **Rest** (Retrofit, OkHttp …). Nach **jedem** größeren Schritt bauen.

---

### Weg 1 — Android Studio (am einfachsten)
- **`libs.versions.toml`** öffnen: veraltete Versionen werden **gelb unterstrichen** → Cursor drauf, **Alt+Enter** → „Change to …".
- **AGP/Gradle:** `Tools ▸ AGP Upgrade Assistant` (führt AGP **und** den passenden Gradle-Wrapper sicher hoch).
- **Compose BOM:** ebenfalls in der toml, neue BOM-Version eintragen.

### Weg 2 — Gradle „versions"-Plugin (zeigt alles Veraltete)
```kotlin
// build.gradle.kts (root) oder settings:
plugins { id("com.github.ben-manes.versions") version "0.51.0" }   // Version ggf. prüfen
```
```bash
./gradlew dependencyUpdates       # listet je Dependency: aktuell -> verfügbar
```

### Weg 3 — Die Repos direkt fragen (zuverlässig, ohne Plugin) ⭐
```bash
# Neueste STABILE Version einer AndroidX-Lib (Google Maven) — alpha/beta/rc rausgefiltert:
curl -s https://dl.google.com/dl/android/maven2/androidx/room/room-runtime/maven-metadata.xml \
  | grep -oP '(?<=<version>)[^<]+' | grep -viE 'alpha|beta|-rc|dev' | tail -1

# Maven Central (z. B. Retrofit):
curl -s https://repo1.maven.org/maven2/com/squareup/retrofit2/retrofit/maven-metadata.xml \
  | grep -oP '(?<=<version>)[^<]+' | grep -viE 'alpha|beta|-rc|dev' | tail -1

# Neueste Gradle-Version:
curl -sL https://services.gradle.org/versions/current        # -> {"version":"9.5.1", ...}
```
> Pfad-Regel: Gruppe `androidx.room` + Artefakt `room-runtime` ⇒ URL `.../androidx/room/room-runtime/maven-metadata.xml`. Punkte in der Gruppe werden zu `/`.

---

### Gradle-Wrapper hochziehen
```bash
# Sauber per Wrapper-Task (rechnet die Prüfsumme selbst aus, 2x ausführen empfohlen):
./gradlew wrapper --gradle-version 9.5.1 --distribution-type bin

# Falls das hakt: manuell in gradle/wrapper/gradle-wrapper.properties:
#   distributionUrl=https\://services.gradle.org/distributions/gradle-9.5.1-bin.zip
# und die offizielle Prüfsumme eintragen (distributionSha256Sum):
curl -sL https://services.gradle.org/distributions/gradle-9.5.1-bin.zip.sha256
```

### Versionen zentral ändern (Version Catalog)
```toml
# gradle/libs.versions.toml — NUR hier die Zahl ändern, gilt projektweit:
[versions]
kotlin = "2.3.21"
ksp = "2.3.9"            # zur Kotlin-Version passend!
room = "2.8.4"
composeBom = "2026.05.01"
retrofit = "2.11.0"
```

### Danach IMMER verifizieren
```bash
./gradlew clean :app:assembleDebug :app:testDebugUnitTest
# -> BUILD SUCCESSFUL ?  Dann passt das Update. Sonst Fehlermeldung lesen
#    (meist KSP/Kotlin-Mismatch oder AGP braucht neueres Gradle).
```

> **Merksatz:** *Zahlen nur in `libs.versions.toml` ändern, Gradle über den Wrapper, Reihenfolge Gradle→AGP→Kotlin/KSP→Libs, danach `clean`-Build. Nur stabile Versionen für die Abgabe.*

---

## 14. Nützliche Links

**Architektur**
- Guide to app architecture: https://developer.android.com/topic/architecture
- UI-State / UDF: https://developer.android.com/topic/architecture/ui-layer
- Data layer / Repository: https://developer.android.com/topic/architecture/data-layer
- Now in Android (Referenz-App): https://github.com/android/nowinandroid
- Architecture Samples: https://github.com/android/architecture-samples

**Room**
- Übersicht: https://developer.android.com/training/data-storage/room
- Daten lesen/schreiben (DAO): https://developer.android.com/training/data-storage/room/accessing-data
- Beziehungen: https://developer.android.com/training/data-storage/room/relationships
- Migrationen: https://developer.android.com/training/data-storage/room/migrating-db-versions
- Room + Flow: https://developer.android.com/training/data-storage/room/async-queries

**Retrofit / OkHttp / JSON**
- Retrofit: https://square.github.io/retrofit/
- OkHttp: https://square.github.io/okhttp/
- kotlinx.serialization: https://github.com/Kotlin/kotlinx.serialization
- Retrofit kotlinx-serialization Converter: https://github.com/JakeWharton/retrofit2-kotlinx-serialization-converter
- Moshi: https://github.com/square/moshi
- Test-API zum Üben: https://jsonplaceholder.typicode.com/

**Coroutines / Flow**
- Coroutines: https://kotlinlang.org/docs/coroutines-overview.html
- Flow: https://kotlinlang.org/docs/flow.html
- Coroutines on Android: https://developer.android.com/kotlin/coroutines
- StateFlow/SharedFlow: https://developer.android.com/kotlin/flow/stateflow-and-sharedflow

**ViewModel / Compose / DI**
- ViewModel: https://developer.android.com/topic/libraries/architecture/viewmodel
- State in Compose: https://developer.android.com/develop/ui/compose/state
- collectAsStateWithLifecycle: https://developer.android.com/develop/ui/compose/state#use-other-types-of-state
- Hilt: https://developer.android.com/training/dependency-injection/hilt-android
- Hilt + Compose: https://developer.android.com/develop/ui/compose/libraries#hilt

**Versionen / Updates**
- Aktuelle Gradle-Version: https://services.gradle.org/versions/current
- AGP ↔ Gradle Kompatibilität: https://developer.android.com/build/releases/gradle-plugin
- Compose-BOM → Bibliotheks-Mapping: https://developer.android.com/develop/ui/compose/bom/bom-mapping
- KSP Releases (zur Kotlin-Version): https://github.com/google/ksp/releases
- gradle-versions-plugin: https://github.com/ben-manes/gradle-versions-plugin
- Maven Central Suche: https://central.sonatype.com/
- Google Maven (AndroidX) Index: https://maven.google.com/web/index.html

---

*Tipp für den Test: Halte dich an die Reihenfolge in Abschnitt 12 (Entity → DAO → DB → DTO/API → Repository → Application → ViewModel → UI). Wenn du diese 8 Schritte runterschreiben kannst, hast du die „saubere Architektur" + Room + Retrofit komplett abgedeckt.*
