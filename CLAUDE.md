# IntervalTimer

Offline Android-App (Intervall-Timer fürs Radtraining), reine Web-App in
einer Datei, die per [apk-builder](../apk-builder) zur APK wird. Details zu
Programmen, Datenmodell, Timer-Engine, Audio/Vibration/Wake-Lock stehen in
[README.md](README.md) — dort nachlesen statt hier duplizieren.

## Build & Test

Kein Bundler, kein Build-Step für die Web-App selbst — `web/index.html` ist
direkt per `file://` im Browser lauffähig und dort auch primär zu testen
(Programmablauf, Pause/Skip/Zurück, Settings, Verlauf, Persistenz nach
Reload). `?fast=1` an die URL anhängen, um alle Phasendauern durch 20 zu
teilen und schnell durchzuklicken.

APK bauen:

```powershell
cd "D:\claude code projects\apk-builder"
.\new-app.ps1 -Name IntervalTimer -PackageId com.daniel.intervaltimer `
              -WebRoot "D:\claude code projects\interval-timer\web" `
              -Icon "D:\claude code projects\interval-timer\icon.xml" `
              -IconBackground "#1D2530" `
              -VersionName "1.0" -VersionCode <hochzaehlen> -Force
.\build-apk.ps1 -App IntervalTimer          # Debug
.\build-apk.ps1 -App IntervalTimer -Release # signiert, fuer echten Gebrauch
```

Was sich nur auf dem echten Geraet (Pixel 11 Pro) verifizieren laesst, nicht
im Desktop-Browser: Vibrationsgefuehl, ob der Wake Lock den Bildschirm ueber
eine volle Session wachhaelt, WebView-Force-Dark-Verhalten, Touch-/Scroll-
Verhalten in Einstellungen/Verlauf.

## Konventionen

- Alles in `web/index.html` (HTML+CSS+JS inline), kein Framework, keine
  externen Abhaengigkeiten — muss offline funktionieren.
- `icon.xml` liegt bewusst **neben** `web/`, nicht darin (siehe README).
- localStorage-Keys sind versioniert (`it_settings_v1`, `it_sessions_v1`,
  `it_custom_programs_v1`) — bei einer Schemaaenderung neue Versionsnummer
  statt stillschweigender Migration.
- Neue Programme (Stufe 2: Custom-Programme) duerfen `buildPhases()` nicht
  mit Programm-spezifischer Logik verzweigen — die Funktion ist bewusst
  generisch ueber `config` gehalten.
- Harter Anspruch: kein Netzwerkzugriff, keine Gerätedaten ausser dem, was
  der Timer selbst braucht. `new-app.ps1` ohne `-Online`-Schalter aufrufen.

## Aktueller Stand

Stufe 1 (Presets Norwegian 4x4 + HIIT 30/30, Verlauf, Einstellungen) ist
fertig und im Browser durchgetestet; Debug-APK wurde gebaut und an den
Nutzer zum Testen auf dem Pixel 11 Pro geschickt. Stufe 2 (frei
konfigurierbare Custom-Programme, Tabata-Style) ist geplant, aber noch
nicht gebaut — siehe README Abschnitt "Datenmodell" für die vorbereitete
Erweiterungsstelle.

Offener Punkt für den nächsten Gerätetest (Review 2026-09-15): Die App hat
keinen eigenen `build.ps1`-Wrapper und damit kein `FLAG_KEEP_SCREEN_ON` wie
ice-breath/breathe-well, sondern nur `navigator.wakeLock` — das die
Android-WebView vermutlich gar nicht unterstützt. Bitte bei einer langen
Session (Norwegian 4x4) prüfen, ob der Bildschirm ausgeht. Falls ja: Wrapper
nach Vorbild von `ice-breath/build.ps1` (Patch "Bildschirm wach halten")
anlegen.

Behoben im selben Review: Weiter/Zurück während einer Pause verrechneten die
Pausendauer doppelt (Position sprang nach dem Fortsetzen), und der Verlauf
speichert jetzt die echte Trainingszeit ohne Pausen statt der Timer-Position
(Überspringen des Aufwärmens zählt also nicht mehr als 5 Min Training).
