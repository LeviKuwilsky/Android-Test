# Android REST Client Setup (Compose + MVVM + Retrofit)

Dieses Dokument enthält alle notwendigen Konfigurationen, um ein modernes Android-Projekt mit Jetpack Compose, Navigation, Retrofit und Jackson aufzusetzen.

## 1. `gradle/libs.versions.toml`

Diese Datei definiert zentral alle Versionsnummern, Bibliotheken und Plugins für das Projekt.

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
navigationCompose = "2.8.0"
navigation3 = "3.0.0-alpha01"

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
androidx-navigation-compose = { module = "androidx.navigation:navigation-compose", version.ref = "navigationCompose" }
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

---

## 2. `app/build.gradle.kts`

Das Build-Skript für das App-Modul, welches die Bibliotheken aus dem Version Catalog einbindet.

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.jetbrains.kotlin.android)
    alias(libs.plugins.kotlin.serialization)
}

android {
    namespace = "at.htlleonding.restclient"
    compileSdk = 34

    defaultConfig {
        applicationId = "at.htlleonding.restclient"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions {
        jvmTarget = "17"
    }
    buildFeatures {
        compose = true
    }
    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.14" 
    }
}

dependencies {
    // --- Core & Lifecycle ---
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.lifecycle.runtime)
    implementation(libs.androidx.lifecycle.viewmodel.compose)
    implementation(libs.androidx.activity.compose)

    // --- Compose ---
    implementation(platform(libs.compose.bom))
    implementation(libs.compose.ui)
    implementation(libs.compose.ui.preview)
    implementation(libs.compose.material3)
    debugImplementation(libs.compose.ui.tooling)

    // --- Navigation ---
    implementation(libs.androidx.navigation.compose)

    // --- Serialization ---
    implementation(libs.kotlinx.serialization.json)

    // --- Networking (Retrofit & Jackson) ---
    implementation(libs.retrofit)
    implementation(libs.converter.jackson)
}
```

---

## 3. `app/src/main/AndroidManifest.xml`

Das Android-Manifest, das die essenziellen Netzwerkberechtigungen für Retrofit bereitstellt.

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="[http://schemas.android.com/apk/res/android](http://schemas.android.com/apk/res/android)">

    <!-- Zwingend erforderlich für Retrofit / REST-Calls -->
    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="RestClientApp"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.RestClient"
        android:usesCleartextTraffic="true">
        
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/Theme.RestClient">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

---
# Versions- & Update-Guide für Android-Projekte (Android Studio & Gradle)

```bash
./update_project.sh
```
Dieser Guide dient als Referenz, um sicherzustellen, dass die Entwicklungsumgebung, der Gradle-Wrapper sowie alle Projekt-Dependencies stets auf dem aktuellsten, stabilen Stand sind. Dies ist besonders für die Abgabe bei versionstreuen Prüfern relevant.

---

## 1. Entwicklungsumgebung (Android Studio)

Um Android Studio ohne manuellen Aufwand plattformübergreifend aktuell zu halten, wird die **JetBrains Toolbox** genutzt. Sie verwaltet Updates im Hintergrund und ermöglicht ein sicheres Verwalten der IDE-Versionen.

* **Vorteil:** Verhindert Versionskonflikte im System und aktualisiert die IDE sauber im Hintergrund.
* **Integrierter AGP-Assistent:** Für das Android Gradle Plugin (AGP) selbst sollte innerhalb der IDE immer der **AGP Upgrade Assistant** unter `Tools > AGP Upgrade Assistant...` verwendet werden, da dieser automatische Code-Refactorings in den `build.gradle.kts`-Dateien vornimmt.

---

## 2. Automatischer Dependency Checker (Plugin-Setup)

Um veraltete Bibliotheken im Projekt schnell zu identifizieren, wird das `ben-manes` Versions-Plugin in die projektweite `build.gradle.kts` eingebunden.

Füge folgende Zeile in die `plugins`-Block deiner **projektweiten** `build.gradle.kts` ein:

```kotlin
plugins {
    // Bestehende Plugins ...
    id("com.github.ben-manes.versions") version "0.51.0"
}
