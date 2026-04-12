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
















---------------------







#  🚀 WMC Android Test: Der Masterplan

## 1. Projekt-Setup & Konfiguration
Bevor du startest, müssen die Abhängigkeiten (Dependencies) stimmen. Ohne diese funktioniert die Navigation und Serialisierung nicht.

### `libs.versions.toml` (Ausschnitt)
Stelle sicher, dass diese Versionen oder ähnliche vorhanden sind:
```toml
[versions]
kotlinSerialization = "1.6.3" # Aktualisierte Version
navigationCompose = "2.9.7" # Aktualisierte Version
lifecycleViewmodelCompose = "2.10.0" # Neu hinzugefügt

[libraries]
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigationCompose" }
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "kotlinSerialization" }
androidx-lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycleViewmodelCompose" } # Neu hinzugefügt

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
    implementation(libs.androidx.lifecycle.viewmodel.compose) // Für MVVM
    // Weitere Abhängigkeiten
    implementation(libs.androidx.navigation3.runtime)
    implementation(libs.androidx.navigation3.ui)
    implementation(libs.kotlinx.serialization.core)
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

### Der NavHost mit Top- und Bottom-Bar (in MainActivity.kt)
```kotlin
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.compose.foundation.layout.*
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.navigation.NavType
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import androidx.navigation.navArgument
import androidx.navigation.compose.currentBackStackEntryAsState
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Home
import androidx.compose.material.icons.filled.Person
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    val userViewModel: UserViewModel = viewModel()
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute = navBackStackEntry?.destination?.route

    Scaffold(
        topBar = {
            TopAppBar(title = { Text("NavTest App") })
        },
        bottomBar = {
            NavigationBar {
                val items = listOf("home", "profile")
                items.forEach { screen ->
                    NavigationBarItem(
                        icon = {
                            when (screen) {
                                "home" -> Icon(Icons.Filled.Home, contentDescription = "Home")
                                "profile" -> Icon(Icons.Filled.Person, contentDescription = "Profile")
                            }
                        },
                        label = { Text(screen.replaceFirstChar { it.uppercase() }) },
                        selected = currentRoute == screen || (currentRoute?.startsWith("details") == true && screen == "home"),
                        onClick = {
                            navController.navigate(screen) {
                                popUpTo(navController.graph.startDestinationId) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        }
                    )
                }
            }
        }
    ) { paddingValues ->
        NavHost(
            navController = navController,
            startDestination = "home",
            modifier = Modifier.padding(paddingValues)
        ) {
            // Erste Seite: Home / Input
            composable("home") {
                HomeScreen(
                    viewModel = userViewModel,
                    onNavigateToDetails = { name ->
                        navController.navigate("details/$name")
                    }
                )
            }

            // Zweite Seite: Details mit Parameter
            composable(
                route = "details/{userName}",
                arguments = listOf(navArgument("userName") { type = NavType.StringType })
            ) { backStackEntry ->
                val userName = backStackEntry.arguments?.getString("userName") ?: "Unbekannt"
                DetailScreen(
                    name = userName,
                    onBack = { navController.popBackStack() }
                )
            }

            // Neue Seite: Profile
            composable("profile") {
                ProfileScreen(viewModel = userViewModel)
            }
        }
    }
}
```

---

## 3. MVVM (Model-View-ViewModel)
Trenne Logik von der UI. Speichere Zustände hier.

### Das Model & State (Datenstruktur)
```kotlin
data class UserState(val name: String = "", val clickCount: Int = 0)
```

### Das ViewModel (Logik)
```kotlin
import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

class UserViewModel : ViewModel() { // Umbenannt von MainViewModel zu UserViewModel
    // Interner veränderbarer State
    private val _uiState = MutableStateFlow(UserState())
    // Öffentlicher unveränderbarer State für die UI
    val uiState: StateFlow<UserState> = _uiState.asStateFlow()

    // Event: Textfeld wurde geändert
    fun updateName(newValue: String) {
        _uiState.update { it.copy(name = newValue) }
    }

    // Event: Button wurde geklickt
    fun incrementClick() {
        _uiState.update { it.copy(clickCount = it.clickCount + 1) }
    }
}
```

---

## 4. UI Komponenten & Layouts
Hier baust du deine Screens zusammen.

### Grundgerüst (`Scaffold` - Beispiel für HomeScreen)
```kotlin
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.foundation.layout.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.viewmodel.compose.viewModel

@Composable
fun HomeScreen(viewModel: UserViewModel = viewModel(), onNavigateToDetails: (String) -> Unit) {
    // GANZ WICHTIG: Den State beobachten!
    val state by viewModel.uiState.collectAsState()

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp), // Layout verändern mit Modifier
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = "Demo Projekt",
            style = MaterialTheme.typography.headlineLarge,
            modifier = Modifier.padding(bottom = 16.dp)
        )

        // Komponente: TextField
        TextField(
            value = state.name,
            onValueChange = { viewModel.updateName(it) },
            label = { Text("Gib deinen Namen ein") },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(20.dp))

        // Event von Komponente für die Navigation benutzen
        Button(
            onClick = { onNavigateToDetails(state.name) },
            modifier = Modifier.fillMaxWidth(),
            enabled = state.name.isNotBlank()
        ) {
            Text("Weiter zur Detailseite")
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
                .clickable { /* onNavigateToDetail(1, item) */ } // Navigation bei Klick
        ) {
            Text(text = item, modifier = Modifier.padding(16.dp))
        }
    }
}
```

### DetailScreen
```kotlin
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.compose.foundation.layout.*

@Composable
fun DetailScreen(name: String, onBack: () -> Unit) {
    Card(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 8.dp)
    ) {
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(24.dp),
            verticalArrangement = Arrangement.Center
        ) {
            Text(
                text = "Hallo, $name!",
                style = MaterialTheme.typography.displaySmall
            )

            Text(
                text = "Dieser Name wurde erfolgreich per Navigation übergeben.",
                modifier = Modifier.padding(top = 8.dp, bottom = 24.dp)
            )

            Button(onClick = onBack) {
                Text("Zurück zum Start")
            }
        }
    }
}
```

### ProfileScreen (Neu hinzugefügt)
```kotlin
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.compose.foundation.layout.*
import androidx.lifecycle.viewmodel.compose.viewModel

@Composable
fun ProfileScreen(viewModel: UserViewModel) {
    val state by viewModel.uiState.collectAsState()
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = androidx.compose.ui.Alignment.CenterHorizontally
    ) {
        Text(
            text = "Profile Page",
            style = MaterialTheme.typography.headlineLarge,
            modifier = Modifier.padding(bottom = 16.dp)
        )
        Text(
            text = "Current User: ${state.name.ifBlank { "Guest" }}",
            style = MaterialTheme.typography.bodyLarge
        )
        Spacer(modifier = Modifier.height(8.dp))
        Text(
            text = "Clicks: ${state.clickCount}",
            style = MaterialTheme.typography.bodyMedium
        )
        Spacer(modifier = Modifier.height(20.dp))
        Button(onClick = { viewModel.incrementClick() }) {
            Text("Increment Click Count")
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

