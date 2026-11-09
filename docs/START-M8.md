# Auftrag M8 · PWA statt Android-App

> **Start-Prompt für Claude in der IDE:**
>
> Lies `CLAUDE.md`, `docs/DECISIONS.md` (Abschnitt M8) und `docs/START-M8.md`. Setze M8 Schritt für Schritt um, ein Commit pro Schritt, und frag nach, wenn etwas unklar ist.

## Ziel

Die Web-Version unter kosten.codekorb.de lässt sich auf Android (Chrome) und am Desktop (Chrome/Edge) als App installieren, bleibt eingeloggt, öffnet auch offline mit den zuletzt geladenen Daten und bekommt Push-Erinnerungen über Web Push. Expo EAS, Firebase/FCM, APK-Verteilung und App Links werden für den Betrieb nicht mehr gebraucht.

Der native Android-Code (`push.ts`, `tokenStorage.ts`, `ExpoPushClient`, `push_tokens`) bleibt unverändert im Repo, wird aber nicht mehr gebaut oder ausgeliefert.

## Schritte

### 1. Doku
- [ ] `docs/SPEC.md`: Plattform, Push-Kanal, Datenmodell (`web_push_subscriptions`), Endpunkte, Sicherheit (Session-Cookie im Web), Deployment, M8 in der Meilensteinliste.
- [ ] `docs/DEPLOY.md`: Abschnitt 6 (EAS/FCM/App Links) als „zurückgestellt“ markieren, VAPID-Schlüssel in der `.env` ergänzen. Außerdem: `APP_KEY` lokal erzeugen und die `.env` vor dem ersten Deploy anlegen.
- [ ] `CLAUDE.md`: „Aktueller Auftrag“ auf diese Datei zeigen lassen.

### 2. Login im Web per Session-Cookie (Sanctum SPA-Auth)
- [ ] Backend: `statefulApi()` in `bootstrap/app.php`, `SANCTUM_STATEFUL_DOMAINS=kosten.codekorb.de` (lokal `localhost:8081`), `SESSION_SECURE_COOKIE=true`, `SESSION_SAME_SITE=lax`, `http_only` an.
- [ ] Login mit `"session": true` (Web) meldet über `Auth::guard('web')->login($user, remember: true)` an, regeneriert die Session und gibt **kein** Token zurück. Ohne das Flag bleibt alles wie bisher (Bearer-Token, für Android).
- [ ] Logout im Web: Session invalidieren und CSRF-Token neu erzeugen. Passwort-Reset beendet zusätzlich alle Sessions des Nutzers (Session-Tabelle nach `user_id`).
- [ ] Sitzungsdauer: Remember-Cookie, damit die PWA eingeloggt bleibt (Laufzeit wie Laravel-Standard, bis zum Logout).
- [ ] App: `tokenStorage.web.ts` entfällt. `api.ts` im Web nutzt `credentials: 'include'`, holt vor dem Login `/sanctum/csrf-cookie` und schickt `X-XSRF-TOKEN`. `auth.tsx` stellt die Session über `GET /me` wieder her statt über ein gespeichertes Token. Der Cache-Löschen-Pfad bleibt (401 bedeutet: Cache leeren).
- [ ] Tests: Login/Logout per Session, CSRF-Pflicht (419 ohne Token), Bearer-Token funktioniert weiter, Passwort-Reset beendet Sessions, Nutzer B sieht per Session nichts von Nutzer A.

### 3. Installierbar und offline (Manifest + Service Worker)
- [ ] `app/public/` (Expo kopiert den Ordner beim Export ins Wurzelverzeichnis): `manifest.webmanifest` (Name, `display: standalone`, `start_url: /`, Farben aus dem Theme, `lang: de`), Icons 192/512 und `maskable`.
- [ ] `sw.js`, handgeschrieben, ohne Workbox:
  - Navigation: zuerst das Netz, bei Fehler die gecachte `index.html`.
  - `/_expo/static/*` und Fonts/Bilder: zuerst der Cache (Dateinamen enthalten einen Hash).
  - `/api/*` und `/sanctum/*`: nie cachen.
  - Alte Caches beim `activate` löschen.
- [ ] `scripts/post-export-web.js`: `<link rel="manifest">` und `theme-color` in `index.html` einfügen, Build-Kennung in `sw.js` schreiben (Cache-Version).
- [ ] Service Worker nur im Web-Build registrieren, z. B. in `_layout.tsx` über eine `.web.ts`-Datei.
- [ ] `.htaccess`/`WebAppController`: `sw.js` mit `Cache-Control: no-cache` ausliefern, `.webmanifest` als `application/manifest+json`.
- [ ] Einstellungen: Zeile „App installieren“. Erscheint nur, wenn der Browser `beforeinstallprompt` meldet, und fehlt, wenn die App schon installiert ist (`display-mode: standalone`).

### 4. Web Push im Backend
- [ ] `minishlink/web-push` (Composer). VAPID-Schlüssel in `.env` (`VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT=mailto:kosten@codekorb.de`) und Artisan-Befehl `webpush:vapid`, der ein Schlüsselpaar ausgibt.
- [ ] Migration `web_push_subscriptions`: user_id, endpoint (`text`), endpoint_hash (SHA-256, unique), p256dh, auth, device_name, last_seen_at. Model mit `BelongsToUser`.
- [ ] Endpunkte: `POST /web-push-subscriptions` (ein bestehender Endpoint wechselt zum aktuellen Nutzer, wie bei `push_tokens`) und `DELETE /web-push-subscriptions` (Body: endpoint). `GET /settings/reminders` liefert zusätzlich `vapid_public_key`; `push_devices` zählt beide Arten mit.
- [ ] `WebPushClient` analog zu `ExpoPushClient`: Payload `{title, body, url}`, verschlüsselt. Antworten 404/410 löschen das Abo. `SendPushReminders` schickt an beide Arten. `sent_at` wird gesetzt, sobald mindestens eine Zustellung angenommen wurde.
- [ ] Texte unverändert aus `ReminderTexts` (keine Beträge).
- [ ] Tests: Registrieren/Löschen, Mandantentrennung, Versand mit gefaktem Client, 410 räumt auf, doppelter Lauf sendet nichts doppelt.

### 5. Web Push in der App
- [ ] `push.web.ts` umsetzen: `pushSupported` = `'serviceWorker' in navigator && 'PushManager' in window`. `registerForPush` fragt die Berechtigung nur nach einem Klick (Button „Jetzt einrichten“), ruft `pushManager.subscribe({ userVisibleOnly: true, applicationServerKey })` auf und schickt das Abo an die API. `unregisterPush` beim Logout.
- [ ] `usePushSetup.web.ts`: Ist die Berechtigung schon erteilt, das Abo beim Start still neu melden (`last_seen_at`, Endpoint kann sich ändern).
- [ ] `sw.js`: `push` → `showNotification` (Icon, `tag` je Abo); `notificationclick` → vorhandenes Fenster fokussieren und zur `url` navigieren, sonst ein neues öffnen.
- [ ] `reminders.tsx`: Den Text „Im Browser gibt es keine Push-Nachrichten …“ ersetzen. Neuer Zustand, wenn der Browser kein Push kann.
- [ ] Tests: Umwandlung des VAPID-Schlüssels (base64url → Uint8Array), Ablauf von `registerForPush` mit gemocktem `PushManager`.

### 6. Abnahme
- [ ] Deploy, dann in Chrome auf Android installieren: App-Icon, Vollbild, Login bleibt nach Neustart erhalten.
- [ ] Flugmodus: App öffnet, Übersicht zeigt die gecachten Daten.
- [ ] `php artisan reminders:dispatch` + `queue:work --stop-when-empty`: Push kommt an, Tippen öffnet das richtige Abo.
- [ ] Desktop (Chrome/Edge): Installation und Push.
- [ ] Logout auf dem Handy: Danach kommt kein Push mehr an dieses Gerät.

## Nicht in M8

- iPhone-Feinschliff (Apple-Meta-Tags, Hinweis „Zum Home-Bildschirm“). Im Safari-Browser läuft die App trotzdem.
- Native Android-App, EAS, Firebase, App Links. Bleiben im Repo, „später optional“.
