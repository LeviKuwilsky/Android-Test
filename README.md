# 🚀 WMC Android Test: Der Masterplan

## 1. Projekt-Setup & Konfiguration
Bevor du startest, müssen die Abhängigkeiten (Dependencies) stimmen. Ohne diese funktioniert die Navigation und Serialisierung nicht.

### `libs.versions.toml` (Ausschnitt)
Stelle sicher, dass diese Versionen oder ähnliche vorhanden sind:
```toml
[versions]
kotlinSerialization = "1.7.1"
navigationCompose = "2.8.0-beta02" # Oder aktuellere

[libraries]
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigationCompose" }
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "kotlinSerialization" }

[plugins]
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

### `build.gradle.kts` (Project Level)
```kotlin
plugins {
    alias(libs.plugins.kotlin.serialization) apply false
}
```

### `app/build.gradle.kts` (App Level)
```kotlin
plugins {
    alias(libs.plugins.kotlin.serialization)
}

dependencies {
    implementation(libs.androidx.navigation.compose)
    implementation(libs.kotlinx.serialization.json)
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.0") // Für MVVM
}
```

---

## 2. Zentrale Navigation (Navigation 3 & Type Safety)
Das Herzstück deiner App. Hier definierst du, wohin man gehen kann und wie Daten übergeben werden.

### Routen definieren
Erstelle eine Datei `Routes.kt` (oder packe es in deine Navigation-Datei):
```kotlin
import kotlinx.serialization.Serializable

@Serializable
object HomeRoute // Startseite ohne Parameter

@Serializable
data class DetailRoute(
    val id: Int,
    val title: String
) // Seite mit Parametern
```

### Der NavHost (in MainActivity.kt)
```kotlin
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import androidx.navigation.toRoute

@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = HomeRoute) {
        // Screen 1: Home
        composable<HomeRoute> {
            HomeScreen(
                onNavigateToDetail = { id, name -> 
                    // Navigation mit Parametern
                    navController.navigate(DetailRoute(id, name)) 
                }
            )
        }

        // Screen 2: Details mit Parameter-Empfang
        composable<DetailRoute> { backStackEntry ->
            // Parameter auslesen
            val args = backStackEntry.toRoute<DetailRoute>()
            DetailScreen(
                itemId = args.id,
                itemTitle = args.title,
                onBack = { navController.popBackStack() }
            )
        }
    }
}
```

---

## 3. MVVM (Model-View-ViewModel)
Trenne Logik von der UI. Speichere Zustände hier.

### Das Model & State (Datenstruktur)
```kotlin
data class AppState(
    val items: List<String> = emptyList(),
    val isLoading: Boolean = false,
    val userInput: String = ""
)
```

### Das ViewModel (Logik)
```kotlin
import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

class MainViewModel : ViewModel() {
    // Interner veränderbarer State
    private val _uiState = MutableStateFlow(AppState())
    // Öffentlicher unveränderbarer State für die UI
    val uiState = _uiState.asStateFlow()

    // Event: Textfeld wurde geändert
    fun updateInput(newValue: String) {
        _uiState.update { it.copy(userInput = newValue) }
    }

    // Event: Button wurde geklickt
    fun addItem() {
        val currentInput = _uiState.value.userInput
        if (currentInput.isNotBlank()) {
            _uiState.update { 
                it.copy(
                    items = it.items + currentInput, // Neues Element hinzufügen
                    userInput = "" // Textfeld wieder leeren
                ) 
            }
        }
    }
}
```

---

## 4. UI Komponenten & Layouts
Hier baust du deine Screens zusammen.

### Grundgerüst (`Scaffold`)
```kotlin
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.foundation.layout.*
import androidx.compose.ui.Modifier
import androidx.lifecycle.viewmodel.compose.viewModel

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun HomeScreen(viewModel: MainViewModel = viewModel(), onNavigateToDetail: (Int, String) -> Unit) {
    // GANZ WICHTIG: Den State beobachten!
    val state by viewModel.uiState.collectAsState()

    Scaffold(
        topBar = { TopAppBar(title = { Text("Meine App") }) },
        floatingActionButton = {
            FloatingActionButton(onClick = { viewModel.addItem() }) {
                Text("+")
            }
        }
    ) { padding ->
        // WICHTIG: padding vom Scaffold verwenden
        Column(modifier = Modifier.padding(padding).fillMaxSize()) {
            
            // Textfeld an ViewModel anbinden
            TextField(
                value = state.userInput,
                onValueChange = { viewModel.updateInput(it) },
                modifier = Modifier.fillMaxWidth().padding(16.dp),
                label = { Text("Neuer Eintrag") }
            )

            // Hier kommt die Liste hin (siehe unten)
        }
    }
}
```

### Wichtige Layout-Modifier
| Modifier | Zweck |
| :--- | :--- |
| `.fillMaxSize()` | Nimmt den ganzen verfügbaren Platz ein. |
| `.fillMaxWidth()` | Nimmt die ganze Breite ein. |
| `.padding(16.dp)` | Abstand nach außen. (Tipp: `padding(top = 8.dp)` für spezifische Seiten) |
| `.weight(1f)` | In einer Row/Column: Nimmt den restlichen Platz proportional ein. Sehr nützlich für Listen in einer Column! |
| `.clickable { ... }` | Macht jedes Element (z.B. Box, Card, Text) klickbar. |

### Listen (`LazyColumn`)
Niemals eine normale Column für viele Daten verwenden!
```kotlin
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.clickable

// Diese Komponente kommt z.B. in die Column vom Scaffold
LazyColumn(modifier = Modifier.weight(1f)) {
    items(state.items) { item ->
        Card(
            modifier = Modifier
                .fillMaxWidth()
                .padding(horizontal = 16.dp, vertical = 8.dp)
                .clickable { onNavigateToDetail(1, item) } // Navigation bei Klick
        ) {
            Text(text = item, modifier = Modifier.padding(16.dp))
        }
    }
}
```

---

## 5. Häufige Fehler & Lösungen (Notfall-Checkliste)

1.  **"Unresolved Reference: toRoute" oder "Serializable not found"** -> Hast du die Kotlin Serialization Library (Plugins UND Dependencies) in den `build.gradle.kts` Dateien hinzugefügt?
2.  **Layout verschwindet oben unter der TopBar** -> Hast du das `padding` vom `Scaffold` (meist `innerPadding` oder `padding` genannt) in deinem ersten Element (`Column`) benutzt? `Modifier.padding(padding)`
3.  **App stürzt bei Navigation sofort ab** -> Prüfe, ob die Route in deinem `navController.navigate(...)` Aufruf *genau* die Parameter hat, die in der `data class` gefordert sind.
4.  **ViewModel Daten (z.B. Liste) ändern sich nicht in der UI, obwohl der Code läuft** -> Hast du `val state by viewModel.uiState.collectAsState()` geschrieben? Ohne `collectAsState` merkt das UI (Compose) nicht, dass neue Daten da sind.
5.  **Imports funktionieren nicht (rot)**
    -> Alt+Enter drücken. Achte darauf, die `androidx.compose...` oder `androidx.navigation...` Imports auszuwählen, nicht die alten Android-Imports.
