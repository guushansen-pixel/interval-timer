# IntervalTimer

Offline Intervall-Timer fürs Radtraining, reine Web-App, die per
[apk-builder](../apk-builder) zu einer Android-APK wird. Läuft komplett
offline (kein Netzwerkzugriff, keine Gerätedaten), alles in einer Datei.

## Bauen

```powershell
cd "D:\claude code projects\apk-builder"
.\new-app.ps1 -Name IntervalTimer -PackageId com.daniel.intervaltimer `
              -WebRoot "D:\claude code projects\interval-timer\web" `
              -Icon "D:\claude code projects\interval-timer\icon.xml" `
              -IconBackground "#1D2530" `
              -VersionName "1.0" -VersionCode 1 -Force
.\build-apk.ps1 -App IntervalTimer -Release
```

`-VersionCode` bei jeder Auslieferung hochzählen. Kein `-Online`-Schalter -
die App bekommt bewusst keine INTERNET-Berechtigung.

Das Icon liegt bewusst **neben** `web\`, nicht darin - sonst wanderte es
zusätzlich als Web-Asset in die APK.

## Programme (Stufe 1)

Zwei feste Presets, Dauer/Rundenzahl aber einstellbar (Setup-Screen vor
dem Start bzw. global in den Einstellungen):

- **Norwegian 4x4** - 4× 4 Min hart / 3 Min locker, dazu Aufwärmen/Abwärmen.
- **HIIT 30/30** - 30 Sek hart / 30 Sek locker, Standard 10 Runden.

Aufwärmen/Abwärmen sind standardmäßig je 5 Minuten (kurz gehalten, weil
der Nutzer oft wenig Zeit hat), aber jederzeit anpassbar.

## Datenmodell

Ein `Program` ist reine Konfiguration (`config: {warmupSec, cooldownSec,
rounds, workSec, restSec, restAfterLast}`), nie eine hartkodierte
Phasenliste. `buildPhases(program)` in `web/index.html` erzeugt daraus
generisch die Phasenfolge (Bereit → Aufwärmen → n× Arbeit/Erholung →
Abwärmen → Fertig). Diese eine Funktion bedient sowohl die Presets heute
als auch künftige Custom-Programme (Stufe 2) - keine Programm-spezifische
Verzweigung irgendwo im Code.

`getAllPrograms()` liefert `PRESETS.concat(loadCustomPrograms())`;
`it_custom_programs_v1` im localStorage ist für Stufe 2 reserviert und
wird heute nur gelesen (immer `[]`), nie geschrieben. Der einzige neue
Aufwand für freie Tabata-Programme später: ein Editor-Screen, der ein
`Program`-Objekt mit `type:"custom"` dort hineinschreibt.

## Timer-Engine

Kein `setInterval`-Delta-Aufaddieren - der Zustand wird bei jedem Tick aus
`Date.now()` neu abgeleitet (`elapsedMs = jetzt - start - pausenzeit -
manueller Offset`). Dadurch gibt es nach einer WebView-Drosselung im
Hintergrund keinen Drift zu korrigieren: der nächste Tick berechnet sofort
den korrekten Zustand. Pause friert die Anzeige ein, indem `tick()` bei
`state.paused` einfach den zuletzt berechneten Wert weiterzeigt, statt neu
zu rechnen; beim Fortsetzen kompensiert `pausedAccumMs` die verstrichene
Pausenzeit. Skip/Zurück verschieben `manualOffsetMs` auf die nächste bzw.
vorherige Phasengrenze.

Debug-Flag `?fast=1` teilt alle Phasendauern durch 20, für schnelles
Durchklicken beim Testen. Nicht im normalen Pfad aktiv.

## Ton, Vibration, Wake Lock

Sound wird zur Laufzeit mit der Web Audio API synthetisiert - keine
Audiodateien, passt zur fehlenden INTERNET-Berechtigung. `audioInit()`
hängt an der ersten Nutzergeste (Start-Button), wie von Browsern für
Autoplay verlangt.

Vibration (`navigator.vibrate`) und Wake Lock (`navigator.wakeLock`) sind
feature-detected und in try/catch gewrappt, damit eine WebView ohne
Unterstützung nicht abstürzt, sondern einfach no-opt. Der Wake Lock wird
bei `visibilitychange` neu angefordert, da das Betriebssystem ihn beim
Verstecken des Tabs freigibt.

Alle drei Signalkanäle (Ton/Vibration/Bildschirm-Flash) sind in den
Einstellungen einzeln schaltbar. Die Vibrationsintensität steuert nur
Muster/Dauer, nicht die Amplitude - das kann die Vibration API schlicht
nicht.

## Verlauf

`it_sessions_v1` ist ein simples Append-only-Array (Datum, Programm,
Dauer, `completedFully`). Gesamtsumme und Liste werden beim Anzeigen aus
dem Log berechnet, nie separat gespeichert - eine einzige Quelle der
Wahrheit.
