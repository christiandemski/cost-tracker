# Deployment · kosten.codekorb.de

Die API und die Web-App laufen zusammen auf dem Netcup-Webhosting. Gebaut wird in GitHub Actions, auf den Server kommt nur das fertige Ergebnis (rsync über SSH). Die Android-App wird über Expo EAS gebaut.

## 1. Einmalig im Netcup-Panel (CCP / Plesk)

1. **Subdomain** `kosten.codekorb.de` anlegen.
   - Dokumentenwurzel: `<DEPLOY_PATH>/public`, z. B. `/kosten.codekorb.de/api/public`
   - Der Laravel-Ordner liegt damit unter `<DEPLOY_PATH>`, `.env` außerhalb von `public/`.
2. **PHP** für die Subdomain auf **8.5** stellen (FPM).
3. **SSL:** Let's-Encrypt-Zertifikat aktivieren, „HTTP auf HTTPS umleiten“ einschalten. HSTS aktivieren.
4. **Datenbank:** MySQL-Datenbank und Benutzer anlegen. Zugangsdaten für die `.env` notieren.
5. **Postfach** `kosten@codekorb.de` anlegen (SMTP). SPF und DKIM für `codekorb.de` prüfen.
6. **SSH-Zugang** aktivieren und den öffentlichen Deploy-Schlüssel hinterlegen (siehe Abschnitt 3).
7. **PHP-Pfad auf der Kommandozeile prüfen:** per SSH `php -v` ausführen. Zeigt es nicht 8.5, den vollen Pfad suchen (z. B. `/usr/local/php85/bin/php`) und in GitHub als Variable `PHP_BIN` eintragen (Abschnitt 3).

## 2. `.env` auf dem Server

Einmalig per SSH unter `<DEPLOY_PATH>/.env` anlegen (Vorlage: `api/.env.example`). Wichtige Werte:

```dotenv
APP_NAME=Cost-Tracker
APP_ENV=production
APP_KEY=            # einmalig: php artisan key:generate --show
APP_DEBUG=false
APP_URL=https://kosten.codekorb.de
FRONTEND_URL=https://kosten.codekorb.de

APP_LOCALE=de
LOG_LEVEL=warning

DB_CONNECTION=mysql
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=…
DB_USERNAME=…
DB_PASSWORD=…

SESSION_DRIVER=database
CACHE_STORE=database
QUEUE_CONNECTION=database

MAIL_MAILER=smtp
MAIL_SCHEME=smtps
MAIL_HOST=mxe9bf.netcup.net
MAIL_PORT=465
MAIL_USERNAME=kosten@codekorb.de
MAIL_PASSWORD=…
MAIL_FROM_ADDRESS="kosten@codekorb.de"
MAIL_FROM_NAME="Cost-Tracker"

ANDROID_PACKAGE_NAME=de.codekorb.kosten
ANDROID_SHA256_CERT_FINGERPRINTS=   # siehe Abschnitt 6
```

`chmod 600 .env` setzen.

## 3. GitHub: Secrets und Variable

Repo → Settings → Environments → `production` anlegen und dort eintragen:

| Name | Art | Inhalt |
| --- | --- | --- |
| `SSH_HOST` | Secret | SSH-Host aus dem Netcup-Panel |
| `SSH_USER` | Secret | SSH-Benutzer |
| `SSH_PORT` | Secret | nur wenn nicht 22 |
| `SSH_PRIVATE_KEY` | Secret | privater Schlüssel, nur für den Deploy erzeugt |
| `SSH_KNOWN_HOSTS` | Secret | Ausgabe von `ssh-keyscan -p <port> <host>` |
| `DEPLOY_PATH` | Secret | absoluter Pfad des Laravel-Ordners, z. B. `/kosten.codekorb.de/api` |
| `PHP_BIN` | Variable | nur wenn `php` auf dem Server nicht 8.5 ist |

Deploy-Schlüssel erzeugen (lokal, ohne Passphrase):

```bash
ssh-keygen -t ed25519 -f cost-deploy -C "github-deploy cost-tracker"
# cost-deploy.pub → im Netcup-Panel als SSH-Schlüssel hinterlegen
# cost-deploy     → Inhalt als Secret SSH_PRIVATE_KEY, danach lokal löschen
```

## 4. Ausrollen

GitHub → Actions → **Deploy** → „Run workflow“. Der Workflow:

1. `composer install --no-dev`, Expo-Web-Export nach `api/public/app/`
2. Upload per rsync. `.env`, `storage/` und Logs bleiben auf dem Server.
3. `php artisan migrate --force`, Standardkategorien seeden, `php artisan optimize`
4. Prüft `https://kosten.codekorb.de/api/v1/health` → `{"status":"ok"}`

## 5. Cronjob

Im Netcup-Panel einen Cronjob anlegen, **jede Minute**:

```
* * * * * cd <DEPLOY_PATH> && <PHP_BIN> artisan schedule:run >> /dev/null 2>&1
```

Der Scheduler arbeitet die Mail-Queue ab (`queue:work --stop-when-empty`), räumt abgelaufene Tokens auf und startet ab M3 täglich um 07:00 die Erinnerungen.

## 6. Android-App (EAS)

Einmalig: Expo-Konto anlegen, dann im Ordner `app/`:

```bash
npm install -g eas-cli
eas login
eas init                  # verknüpft das Projekt mit deinem Expo-Konto
eas build -p android --profile preview
```

Am Ende gibt es einen Link zur APK zum direkten Installieren.

**Push-Erinnerungen** (Android über Firebase Cloud Messaging):

1. Nach `eas init` die Projekt-ID aus der Ausgabe (oder expo.dev → Projekt → „Project ID“) als EAS-Umgebungsvariable `EAS_PROJECT_ID` für das Profil `preview` hinterlegen (`eas env:create`) bzw. lokal in `app/.env` setzen. Ohne sie kann die App kein Push-Token holen. Erinnerungen kommen dann nur per Mail.
2. In der [Firebase-Konsole](https://console.firebase.google.com) ein Projekt anlegen, eine Android-App mit Paketname `de.codekorb.kosten` hinzufügen.
3. Firebase → Projekteinstellungen → Dienstkonten → „Neuen privaten Schlüssel generieren“ (JSON).
4. `eas credentials -p android` → „Google Service Account“ → „FCM V1“ → die JSON-Datei hochladen. Die Datei danach lokal löschen, sie gehört nicht ins Repo.
5. Neue APK bauen (`eas build -p android --profile preview`). Beim ersten Start fragt die App nach der Erlaubnis für Benachrichtigungen.
6. Optional: In expo.dev „Enhanced Push Security“ aktivieren und das Access-Token als `EXPO_ACCESS_TOKEN` in die Server-`.env` schreiben.

Erinnerungen manuell testen (auf dem Server): `php artisan reminders:dispatch`, dann `php artisan queue:work --stop-when-empty`. Doppelte Läufe verschicken nichts doppelt.

**App Links** (Einladungs- und Passwort-Links öffnen direkt die App):

1. `eas credentials -p android` → SHA-256-Fingerprint des Signaturzertifikats kopieren.
2. In der Server-`.env` bei `ANDROID_SHA256_CERT_FINGERPRINTS` eintragen, danach `php artisan optimize`.
3. Prüfen: `https://kosten.codekorb.de/.well-known/assetlinks.json` liefert den Eintrag.

## 7. Erste Einladung

Per SSH auf dem Server:

```bash
cd <DEPLOY_PATH>
php artisan invite:create deine@adresse.de --admin
```

Die Mail enthält den Link `https://kosten.codekorb.de/register?token=…` (7 Tage gültig). Weitere Einladungen verschickt danach ein Admin über die API (`POST /api/v1/invitations`).

## Fehlersuche

- **500 nach dem Deploy:** `storage/logs/laravel.log` lesen. Häufig fehlt die `.env` oder `APP_KEY`.
- **Mails kommen nicht an:** Läuft der Cronjob? `php artisan queue:work --stop-when-empty` einmal von Hand starten. Danach SPF/DKIM prüfen.
- **Web-App zeigt „noch nicht gebaut“:** Der Export fehlt in `public/app/`, Deploy erneut starten.
