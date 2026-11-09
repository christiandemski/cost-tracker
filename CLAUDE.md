# Cost-Tracker – Projektkontext für Claude

Persönlicher Abo- und Fixkosten-Tracker: Überblick über monatliche wiederkehrende Kosten, Kündigungsfristen, Preiserhöhungen, CSV-Import aus Kontoauszügen. Mehrere Nutzer (Freunde) mit strikt getrennten Daten.

- **Domain:** https://kosten.codekorb.de (API unter `/api/v1`, Web-App unter `/`)
- **Spezifikation:** `docs/SPEC.md` (Datenmodell, API, Fachlogik, Import, Sicherheit, Deployment, Meilensteine)
- **Design:** `docs/DESIGN.md` (Tokens, Komponenten, Screens, UX-Regeln) + Screen-Vorlagen in `design/screens/`
- **Aktueller Auftrag:** siehe `docs/START-M8.md` (PWA statt Android-App)
- **Geklärte Entscheidungen:** `docs/DECISIONS.md` (vor Änderungen lesen)

## Stack

| Teil | Technik |
| --- | --- |
| Backend | Laravel (aktuelle Version), PHP 8.x, MySQL, Sanctum (Token-Auth) |
| App | React Native mit Expo, Expo Router, TypeScript; Ziele: Android + Web (Desktop) |
| Push | Expo Push Service (Android über FCM) |
| Mail | SMTP über Netcup-Postfach |
| Hosting | Netcup Webhosting 8000 (klassischer Webspace) |

## Repo-Struktur

```
api/        Laravel-Projekt
app/        Expo-Projekt (Android + Web)
docs/       SPEC.md, DESIGN.md, START-M1.md
design/     theme.ts (Vorlage), screens/*.dc.html (Design-Referenz)
```

## Harte Rahmenbedingungen (Hosting)

- **Kein Docker, keine dauerhaft laufenden Prozesse** (kein Queue-Worker-Daemon, kein Node-Server, kein WebSocket). Alles Zeitgesteuerte läuft über **einen Cronjob** → `php artisan schedule:run`.
- Queue-Treiber `database`, abgearbeitet im Scheduler mit `queue:work --stop-when-empty`.
- Composer- und Build-Schritte laufen lokal oder in GitHub Actions; auf den Server kommt das fertige Ergebnis (SFTP/rsync).
- Dokumentenwurzel der Subdomain zeigt auf `api/public/`. `.env` liegt außerhalb von `public/`.
- Web-Version der App = statischer Export (`npx expo export --platform web`), von Laravel unter `/` ausgeliefert (SPA-Fallback auf `index.html` für alle Nicht-API-Pfade).

## Konventionen

- **Code, Bezeichner, Commits: Englisch.** **UI-Texte: Deutsch** (Du-Form, kurz, konkret).
- **Geld immer als Integer in Cent** (`amount_cents`, `monthly_cents`). Nie Floats für Beträge. Formatierung erst in der Anzeige (`de-DE`, `1.234,56 €`).
- **Datumswerte** als `DATE`, Rechnung in `Europe/Berlin`. Monatsenden mit `addMonthsNoOverflow`, immer vom `anchor_date` aus rechnen (nie Monat für Monat aufaddieren).
- **Rechenlogik nur im Backend** in Service-Klassen (`BillingCalculator`, `DeadlineCalculator`, `ReserveCalculator`). Die App zeigt an, was die API liefert.
- **Mandantentrennung:** Jedes Model mit `user_id` hat einen Global Scope + Policy. Für jeden neuen Endpunkt einen Feature-Test schreiben, der prüft, dass Nutzer B nicht an Daten von Nutzer A kommt.
- **Erinnerungen idempotent:** vor jedem Versand Eintrag in `sent_reminders` (Unique-Index). Doppelte Cron-Läufe dürfen nichts doppelt senden.
- **Tests:** Pest oder PHPUnit für das Backend; Rechenlogik mit Randfällen (31. des Monats, Februar, Schaltjahr, Zeitumstellung). Im Frontend mindestens Tests für Formatierung und API-Client.
- API-Antworten als JSON-Resources, Fehler im Laravel-Standardformat (422 mit `errors`).

## Datenschutz (nicht verhandelbar)

- Hochgeladene CSV-Dateien **nie dauerhaft speichern**: parsen, Kandidaten ablegen, Datei löschen.
- **Keine IBANs/BICs** und keine Nicht-Abo-Umsätze in der Datenbank.
- iCal-Token nur als Hash speichern. Push-Texte ohne Beträge.
- Registrierung nur per Einladungslink (signierte URL).

## Design-Regeln (Kurzfassung, Details in docs/DESIGN.md)

- Nur Dark Mode. Farben ausschließlich aus dem Theme (`theme.ts`), keine Hex-Werte in Komponenten.
- **Orange (`warn`) bedeutet ausschließlich: Handlungsbedarf** (Kündigungsfrist, Preiserhöhung). Keine Kategorie, kein Deko-Element bekommt Orange.
- Touch-Ziele mindestens 44 × 44 px. Beträge immer mit `fontVariant: ['tabular-nums']`.
- Kategoriefarben nie allein als Bedeutungsträger: immer mit Text daneben.
- Icons: Lucide (`lucide-react-native`), Strichstärke 1.8–2. Keine Emojis.

## Arbeitsweise

- Meilensteine aus `docs/SPEC.md` der Reihe nach. Vor jedem Meilenstein kurz planen, dann in kleinen, lauffähigen Schritten committen.
- Bei Unklarheiten in Spec oder Design: nachfragen statt raten. Offene Punkte stehen am Ende von `docs/SPEC.md`.
