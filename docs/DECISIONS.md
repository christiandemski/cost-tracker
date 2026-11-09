# Entscheidungen

Punkte, die SPEC.md und DESIGN.md offen ließen und die mit dem Nutzer geklärt wurden (Stand: 09.11.2026, M1–M7 umgesetzt, M8 geplant).

## Arbeitsweise

- **Commits ohne `Co-Authored-By`-Zeile** (ausdrücklicher Wunsch). Commit-Mail: christian.demski@outlook.com.
- Vor jedem Meilenstein kurz planen, offene Punkte nachfragen, dann in kleinen lauffähigen Commits arbeiten.
- Lokal: bevorzugt DDEV (`.ddev/`, PHP 8.5, MySQL 8.4, Mailpit), siehe README. Ohne DDEV: System-PHP (8.4 über winget) reicht für die Tests (SQLite in-memory).

## Name (07.11.2026)

- Projekt heißt **Cost-Tracker** (Repo `cost-tracker`), Domain **kosten.codekorb.de**, Android-Paket `de.codekorb.kosten`, Postfach `kosten@codekorb.de`. Grund: neben Abos kommen Raten und Sparpläne dazu (M7).
- Verweise auf den Design-Canvas „Abo-Tracker Design“ und das Claude-Doc bleiben, das sind Namen externer Dokumente.

## M8 · PWA statt Android-App (09.11.2026)

- Statt nativer Android-App wird die Web-Version eine **installierbare PWA** (Android-Chrome und Desktop). Grund: kein EAS/Firebase/APK-Verteilen, Updates kommen mit jedem Deploy sofort an.
- Nativer Android-Code bleibt im Repo, wird aber nicht gebaut; Android-App „später optional“.
- **Push über Web Push (VAPID)**, verschickt per `minishlink/web-push` im Scheduler (kein Dauerprozess). Eigene Tabelle `web_push_subscriptions`; `push_tokens` bleibt für Android.
- **Login im Web per Sanctum-Session-Cookie** (httpOnly, CSRF, Remember-Cookie) statt Token im `sessionStorage`. Bearer-Token bleibt für native Clients.
- Offline: Service Worker cacht die App-Hülle; Daten weiter aus dem bestehenden `localStorage`-Cache. API-Antworten werden nie im Service Worker gecacht.
- **Nur Android + Desktop**, kein iPhone-Feinschliff (Safari läuft, aber ohne Apple-Meta-Tags und Installationshinweis).
- Plan: `docs/START-M8.md`.

## M7 · Raten & Sparen

- Neben Abos gibt es die Arten **Rate** (Kredit, Ratenkauf; mit letzter Rate) und **Sparen** (Sparplan, Bausparen, Vorsorge). Keine „Pflichtzahlungen“, kein Wohnen/Miete, kein Haushaltsbuch (keine Nicht-Abo-Umsätze).
- Sparen zählt nicht zu „Fixkosten pro Monat“, sondern erscheint als eigene Zeile; die Rücklage rechnet Sparen mit ein.
- Tab „Abos“ heißt „Fixkosten“. Neue Standardkategorien „Kredite & Raten“ und „Sparen & Vorsorge“.
- Rate: `ends_at` = letzte Rate (Pflicht). Danach setzt der nächtliche Lauf den Status `finished` („Abbezahlt“); wird die letzte Rate nach hinten verschoben, wird sie wieder aktiv. Raten haben keine Laufzeit/Frist, lassen sich nicht kündigen; Restbetrag und offene Raten liefert die API.
- Sparen: keine Laufzeit/Frist, kein Quartals-Check; „Sparplan beenden“ nutzt den Kündigen-Endpunkt ohne Assistent.
- Import legt alles als Abo an (Hinweis: Art unter „Bearbeiten“ umstellen). Ändert sich der Betrag einer Rate oder eines Sparplans, wird er direkt übernommen (angenommene Preisänderung, kein Orange, kein Push).
- Listenfilter „Beendet“ = gekündigt + abbezahlt (API `status=ended`), dazu Filter nach Art.

## M2 · Abos & Übersicht

- „Demnächst“: orange Fristkarten nur für Abos mit Mindestlaufzeit/automatischer Verlängerung; jederzeit kündbare Abos erscheinen als normale graue Abbuchungskarten.
- Erinnerung pro Abo an-/abschaltbar (`remind`, Standard an); die Abstände gelten global pro Nutzer.
- Rhythmus-Auswahl: monatlich / vierteljährlich / jährlich plus „Anderer Rhythmus“ (wöchentlich, halbjährlich, alle N Wochen/Monate/Jahre).
- Betragsänderungen legen eine angenommene manuelle Preisänderung an (Preisverlauf auf der Detailseite).

## Fristen (Korrektur nach Browser-Test, 06.11.2026)

- `term_end_date` ist der **Verlängerungstag** (erster Tag der neuen Laufzeit); der Vertrag läuft bis zum Vortag.
- `cancel_by` = Verlängerungstag − Kündigungsfrist − 1 Tag, damit die volle Frist vor Vertragsende liegt. Beispiel: Vertrag endet 30.11., Frist 1 Monat → Kündigung muss bis 31.10. da sein.
- „Behalten“ gilt für die Laufzeit mit `term_end_date <= kept_until`, nicht für Fristen vor `kept_until`.

## M3 · Erinnerungen

- Mail: eine Sammel-Mail pro Nutzer und Lauf; Push: einer pro Abo, bei mehr als 3 ein zusammenfassender Push. Deduplizierung in `sent_reminders` weiter pro Abo/Abstand/Kanal.
- Abstände: feste Mehrfachauswahl aus 30, 14, 7, 3, 1 Tagen; mindestens einer; Standard 14 + 3.
- „Abbuchung morgen“: standardmäßig aus; wenn an, nur für Abos mit Rhythmus seltener als monatlich.

## M4 · CSV-Import

- Einzelbuchungen werden nur dann Kandidaten, wenn es SEPA-Lastschriften mit Gläubiger-ID und Mandatsreferenz sind („Rhythmus prüfen“).
- Nicht übernommene Kandidaten werden direkt nach „Übernehmen“ gelöscht, spätestens nach 24 Stunden.
- **Offen:** Der Sparkasse-Adapter (CSV-CAMT V2) ist nur gegen eine synthetische Datei getestet. Für die Abnahme fehlt noch ein echter, anonymisierter Export.

## M5 · Sparen

- Absenderadresse wird im Konto gespeichert; Kundennummer und Kündigungs-Mail des Anbieters als optionale Felder am Abo.
- Keine echte Erinnerung nach „Ich habe gekündigt“, nur ein Hinweis in der Zeitleiste.
- Quartals-Check: wegklickbare Karte in der Übersicht (wenn letzter Check > 3 Monate her) plus fester Link im Tab „Abos“; keine Benachrichtigungen dafür.
- Banner: „Kündigung vormerken“ öffnet den Assistenten; „Behalten“ blendet Banner, orange Karte und Fristen-Erinnerungen bis zur nächsten Verlängerung aus (rücknehmbar auf der Detailseite).

## M6 · Überblick

- Monatsansicht: Tagesfarben in drei Blaustufen (< 30 / 30–100 / > 100 €), **nicht** Orange wie in `Monat.dc.html` – Orange bedeutet nur Handlungsbedarf.
- iCal-Feed: Standard Abbuchungen an, Fristen an, Vorab-Erinnerungen (14 Tage) aus, **Betrag im Titel aus** (Datenschutz).
- iCal-Link wird nur direkt nach dem Erzeugen angezeigt (gespeichert ist nur der SHA-256-Hash); danach nur neu erzeugen oder abschalten. Route: `/api/v1/ical/{token}.ics`.
- Statistik-Verlauf wird rekonstruiert (Startdatum, Kündigungsende, angenommene Preisänderungen), keine Snapshots. Abos ohne Startdatum zählen für alle vergangenen Monate.
- Rücklage-Diagramm: Monate mit Abbuchung grau umrandet statt orange.
- Desktop (Web ab 1024 px): Seitenleiste statt Tab-Leiste, eigene Desktop-Übersicht; Unterseiten mit begrenzter Breite.

## Noch offen

- Browser-Test M4–M6 erledigt (06.11.2026). Daraus entschieden:
  - Import schlägt keinen Preis als Änderung vor, wenn das Abo eine angenommene Preisänderung hat, die nach der letzten Abbuchung im Auszug wirksam wurde (der Auszug ist dann veraltet).
  - Gekündigte Abos zählen ab sofort nicht mehr zu „Fixkosten pro Monat“ (wie bisher); der Text auf dem Kündigungs-Screen sagt das jetzt so.
  - „Kündbar bis“ in der Zeitleiste ist bei „Behalten“ grau statt orange.
  - Statistik: „Seltenere Zahlungen“ statt „Nächste Jahreszahlungen“ (enthält auch Quartalsabos).
  - Web: Ohne Session wird auf `/login` umgeleitet (vorher „Unmatched Route“ unter `/(auth)`), und der Daten-Cache im `localStorage` wird geleert.
- Echter Sparkasse-Export (siehe M4).
- Server-Einrichtung: Netcup, GitHub-Secrets, Cronjob, ab M8 VAPID-Schlüssel (siehe `docs/DEPLOY.md`). EAS/FCM entfällt vorerst.
