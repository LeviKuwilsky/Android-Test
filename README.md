# 📝 Übungsangabe 4 (Probe) — „Team-Aufgaben-Dashboard" · Lesen → Berechnen → Speichern + 2 DAOs

> Neues Thema (nicht Wahlhelfer), **echte API**. Bau aus einem **leeren** Projekt, Cheatsheet offen, ohne KI-Weiterprompten.
> Lösungsbausteine: Cheatsheet **8★** (Lesen→Berechnen→Speichern), **8★★** (mehrere DAOs), **6** (Retrofit), Schnellstart-Block (Room/MVVM/UI).

---

## Test (Probe) aus WMC4 – SS 2026 — „Team-Aufgaben-Dashboard" · 4BHIF · 50 Punkte

**Grundregeln**
- Android Studio, Kotlin, Jetpack Compose. Start von **leerem** Projekt (Empty Activity), kein Klonen.
- Eigene Package-Hierarchie (nicht `com.example`), Java 21, aktuelle Versionen, `ksp` für Room.
- Internet erlaubt, **kein Weiterprompten in KI-Tools**.

**Szenario**
Ein Team verwaltet seine Aufgaben in einem Online-Tool. Sie bekommen alle Aufgaben (mit Status „erledigt/offen") über eine **echte REST-API**. Ihre App soll die Aufgaben **laden**, daraus pro Teammitglied eine **Statistik berechnen** (wie viel ist erledigt?) und **beides speichern**: die Roh-Aufgaben in der einen Tabelle, die berechnete Statistik in einer zweiten — über **zwei DAOs**. Das Dashboard liest **nur die berechnete Statistik aus der DB**.

**🌐 Echte API (ohne Login, sofort nutzbar):**
```
GET https://jsonplaceholder.typicode.com/todos
```
Antwort = Liste von Objekten dieser Form:
```json
{ "userId": 1, "id": 1, "title": "delectus aut autem", "completed": false }
```
> 200 Aufgaben, `userId` 1–10 (= 10 Teammitglieder, je 20 Aufgaben). `completed` ist ein **Boolean**.
> Base-URL für Retrofit: `https://jsonplaceholder.typicode.com/`  (muss mit `/` enden!)

---

### Teil 1 — Projektsetup & Architektur (8 P)
- Eigene Package-Hierarchie, Java 21, aktuelle Versionen, `ksp`-Plugin.
- Saubere Schichten **UI → ViewModel → Repository → Room/Retrofit**. UI greift **nie** direkt auf DB/Netz zu.
- **INTERNET-Permission** im Manifest (sonst stürzt Retrofit ab!).

### Teil 2 — LESEN (10 P)
- Aufgaben per **Retrofit** von der obigen URL laden. Eigenes **DTO** `TodoDto(userId, id, title, completed)`.
- DTO ≠ Entity → mit `toEntity()` mappen. Netzfehler abfangen (`safeCall`/try-catch), App darf **nicht abstürzen**.

### Teil 3 — BERECHNEN (12 P)  ⬅ Kernstück
- Eine **reine Funktion** (keine DB, kein Netz, kein Android-Context), die aus den Aufgaben **pro `userId`** berechnet:
  - **gesamt** (Anzahl Aufgaben),
  - **erledigt** (Anzahl `completed == true`),
  - **offen** (gesamt − erledigt),
  - **erledigtProzent** (`erledigt / gesamt * 100`),
- und zusätzlich das **fleißigste Mitglied** markieren (höchste Prozent bzw. meiste erledigte).
- Reine Funktion = leicht testbar → kleiner **Unit-Test** gibt Pluspunkte.
- *Tipp:* `groupBy { it.userId }`, dann je Gruppe `count { it.completed }`. Spickzettel: Cheatsheet 8★.

### Teil 4 — SPEICHERN mit 2 DAOs (10 P)
- **Zwei Tabellen, zwei DAOs** in **einer** Room-DB (Fall A):
  - `TodoDao` → Roh-Aufgaben schreiben/leeren,
  - `UserStatsDao` → berechnete Statistik je Mitglied schreiben + als `Flow` lesen.
- Speichern in einer **Transaktion** (erst Rohdaten, dann Statistik), beide `suspend`.
- *(Optional Fall B: zwei getrennte `@Database`-Dateien — siehe Cheatsheet 8★★.)*

### Teil 5 — UI & ViewModel (10 P)
- ViewModel nur über das Repository. Statistik-Liste als `StateFlow` (`stateIn`).
- **Button „Laden & Auswerten"** startet Lesen→Berechnen→Speichern in **einer** Coroutine.
- **DashboardScreen** liest die **Statistik aus der DB** (Flow) und zeigt je Mitglied: `userId`, erledigt/gesamt, Prozent (z. B. als Text oder `LinearProgressIndicator`); das **fleißigste Mitglied hervorgehoben**.

**Gutes Gelingen!**

---

## 💡 Datenmodell-Vorschlag (nur Gerüst — Code schreibst du selbst!)

```
data/local/
├── Todo.kt        @Entity("todos")      @PrimaryKey id: Int, userId: Int, title: String, completed: Boolean
├── UserStats.kt   @Entity("user_stats") @PrimaryKey userId: Int, gesamt: Int, erledigt: Int,
│                                         offen: Int, prozent: Double, fleissigster: Boolean
├── TodoDao.kt        @Dao -> upsertAll(list) suspend, clear() suspend
├── UserStatsDao.kt   @Dao -> observeAll(): Flow (ORDER BY prozent DESC), upsertAll(list) suspend, clear() suspend
└── AppDatabase.kt @Database(entities = [Todo::class, UserStats::class], version = 1)
                     abstract fun todoDao(): TodoDao
                     abstract fun userStatsDao(): UserStatsDao
```

**DTO + API:**
```kotlin
@Serializable
data class TodoDto(val userId: Int, val id: Int, val title: String, val completed: Boolean)

interface TodoApi {
    @GET("todos") suspend fun getTodos(): List<TodoDto>
}
// Base-URL: "https://jsonplaceholder.typicode.com/"
```

**Repository-Kern (das Herzstück) — du füllst die Logik:**
```kotlin
suspend fun ladenAuswertenSpeichern(): ApiResult<Unit> = safeCall {
    val todos = api.getTodos().map { it.toEntity() }   // 1) LESEN  (Retrofit, suspend)
    val stats = werteAus(todos)                        // 2) BERECHNEN (reine Funktion!)
    todoDao.clear();      todoDao.upsertAll(todos)     // 3) SPEICHERN (DAO 1)
    userStatsDao.clear(); userStatsDao.upsertAll(stats)//    SPEICHERN (DAO 2)
}

// reine Funktion -> kein dao/api/Context darin -> unit-testbar:
fun werteAus(todos: List<Todo>): List<UserStats> {
    val proUser = todos.groupBy { it.userId }.map { (userId, list) ->
        val erledigt = list.count { it.completed }
        UserStats(userId, list.size, erledigt, list.size - erledigt,
                  prozent = erledigt * 100.0 / list.size, fleissigster = false)
    }
    val maxProzent = proUser.maxOfOrNull { it.prozent }
    return proUser.map { it.copy(fleissigster = it.prozent == maxProzent) }
}
```
> Du darfst `werteAus` natürlich selbst schreiben (gut zum Üben!) — oben steht eine Lösung, an der du dich kontrollieren kannst.

---

## ✅ Selbstkontrolle
- [ ] Klick auf „Laden & Auswerten" füllt **beide** Tabellen (todos + user_stats).
- [ ] `werteAus(...)` ist eine **reine Funktion** (kein `dao`, kein `api`, kein `Context` darin).
- [ ] Prozente stimmen (je Mitglied 20 Aufgaben → Prozent in 5er-Schritten); per **Unit-Test** belegt.
- [ ] DashboardScreen liest **aus dem UserStatsDao** (Flow), nicht aus der berechneten Variable.
- [ ] Daten überleben App-Neustart (→ Room).
- [ ] Kein Absturz ohne Netz (Fehler abgefangen). INTERNET-Permission gesetzt.
- [ ] Schichten sauber: UI → ViewModel → Repository → (2 DAOs + Api).

> **Merksatz (Cheatsheet 8★):** *„Ein Klick startet eine Coroutine. Die liest per `suspend`, übergibt die Daten an eine reine Berechnungsfunktion, und speichert das Ergebnis per `suspend` in Room. Die UI zeigt es über den `Flow` automatisch an."*

> **Zeitziel Trockenlauf:** Datenmodell + 2 DAOs + `werteAus` + Speichern in ~50 min lauffähig, dann Retrofit-Anbindung + Dashboard-UI drauf.

---

## 🔁 Weitere echte URLs zum Variieren (alle ohne Login)
| API | URL | Berechnungs-Idee |
|---|---|---|
| Posts pro User | `https://jsonplaceholder.typicode.com/posts` | Anzahl Posts je `userId`, längster Titel |
| Länder | `https://restcountries.com/v3.1/all?fields=name,region,population` | Bevölkerung & Anzahl Länder **pro Region** |
| Krypto | `https://api.coingecko.com/api/v3/coins/markets?vs_currency=eur` | Ø-Preis, größte Marktkapitalisierung, Top-Gewinner 24h |
> Base-URL endet immer mit `/`, Pfad kommt ins `@GET("...")`. Bei verschachteltem JSON (z. B. `name.common` bei restcountries) eigenes verschachteltes `@Serializable`-DTO bauen.
