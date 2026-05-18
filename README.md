### `libs.versions.toml` (Ausschnitt)
SINNIGE LIBS
```toml
[versions]
# --- Core & Plugins ---
agp = "8.4.1"
kotlin = "1.9.24"
coreKtx = "1.13.1"
lifecycle = "2.8.0"
activityCompose = "1.9.0"

# --- Compose & UI ---
composeBom = "2024.05.00"

# --- Navigation ---
navigationCompose = "2.8.0"          # Aktueller Standard (mit Type-Safety)
navigation3 = "3.0.0-alpha01"        # Falls du wirklich schon das experimentelle Nav3 testest

# --- Networking & Data ---
retrofit = "2.11.0"
kotlinSerialization = "1.7.1"

[libraries]
# --- AndroidX Core ---
androidx-core-ktx = { module = "androidx.core:core-ktx", version.ref = "coreKtx" }
androidx-lifecycle-runtime = { module = "androidx.lifecycle:lifecycle-runtime-ktx", version.ref = "lifecycle" }
androidx-lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycle" }
androidx-activity-compose = { module = "androidx.activity:activity-compose", version.ref = "activityCompose" }

# --- Compose ---
compose-bom = { module = "androidx.compose:compose-bom", version.ref = "composeBom" }
compose-ui = { module = "androidx.compose.ui:ui" }
compose-ui-tooling = { module = "androidx.compose.ui:ui-tooling" }
compose-ui-preview = { module = "androidx.compose.ui:ui-tooling-preview" }
compose-material3 = { module = "androidx.compose.material3:material3" }

# --- Navigation ---
# Standard Compose Navigation (empfohlen)
androidx-navigation-compose = { module = "androidx.navigation:navigation-compose", version.ref = "navigationCompose" }
# Experimental Navigation3 (aus deinem Snippet)
navigation3-runtime = { module = "androidx.navigation3:navigation3-runtime", version.ref = "navigation3" }
navigation3-ui = { module = "androidx.navigation3:navigation3-ui", version.ref = "navigation3" }

# --- Serialization ---
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinSerialization" }

# --- Networking (REST Client) ---
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
converter-jackson = { module = "com.squareup.retrofit2:converter-jackson", version.ref = "retrofit" }


[plugins]
# --- Project Plugins ---
android-application = { id = "com.android.application", version.ref = "agp" }
jetbrains-kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```
