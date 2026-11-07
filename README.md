# Chora – PWA

Version 1.4.0 · verbunden mit Supabase (Daten) und WorkOS (Anmeldung)

## Installieren

1. Den ganzen Ordner auf einen Webserver mit **HTTPS** hochladen (z. B. GitHub Pages, Netlify, eigener Webspace).
2. Auf dem iPhone/iPad in Safari öffnen → Teilen → „Zum Home-Bildschirm“.
3. Bei jedem Update in `sw.js` die Versionsnummer erhöhen. Die Seite selbst wird immer zuerst aus dem Netz geladen, Änderungen kommen also beim nächsten Start an.

Die installierte App arbeitet mit den echten Daten. Mit `?demo` hinter der Adresse (z. B. `https://…/chora/?demo`) startet sie stattdessen mit Beispieldaten im Browser – dort gibt es die Beispielkonten „Mitglied“ und „Verwaltung“, und jeder 6-stellige Code funktioniert.

## Einrichtung (einmalig)

1. **WorkOS**: JWT-Template (Authentication → Sessions → JWT Template):
   ```json
   {
     "role": "authenticated",
     "user_role": {{organization_membership.role}},
     "user_roles": {{organization_membership.roles}}
   }
   ```
   Unter Roles „Multiple Roles“ aktivieren und die Rollen `mitgliedsverwalter`, `aufgabenverwalter`, `aufgabenchecker`, `terminverwalter` anlegen. E-Mail/Passwort-Anmeldung und E-Mail-Bestätigung einschalten.
2. **Supabase → Edge Functions → Secrets**: `WORKOS_API_KEY` = WorkOS API-Key (`sk_…`). Optional: `WORKOS_DEFAULT_ROLE`, falls die Standardrolle eurer Organisation nicht `member` heißt.
3. Der ersten Person im WorkOS-Dashboard die Rolle `mitgliedsverwalter` geben. Alles Weitere geht danach in der App.

Datenbank (Tabellen, Regeln, Funktionen) und die Edge Functions `chora-auth` und `chora-admin` sind im Supabase-Projekt angelegt. Der Quelltext liegt in `chora-supabase.zip`.

## Sicherheit: WorkOS API-Key

Der WorkOS API-Key (`sk_…`) darf **nicht** in diese App. Alles in einer PWA ist für jeden Nutzer lesbar.
Er gehört als Umgebungsvariable in ein kleines Backend. In der App steht nur die öffentliche Client-ID (`CONFIG.workosClientId`).

## Backend-Schnittstelle (für den nächsten Schritt)

In `index.html` steht das Objekt `Api`. Jede Methode ist mit dem passenden Endpunkt kommentiert.
Für den echten Betrieb `CONFIG.mock = false` setzen und die Methoden auf `fetch(CONFIG.apiBase + …)` umstellen.

| App ruft auf | Backend macht bei WorkOS |
| --- | --- |
| `POST /api/auth/register` {email, password} | `POST /user_management/users` (User anlegen), danach `POST /user_management/authenticate` mit `grant_type=password`. WorkOS antwortet mit „email verification required“, schickt den Code per Mail und liefert ein `pending_authentication_token`. |
| `POST /api/auth/verify` {email, code} | `POST /user_management/authenticate` mit `grant_type=urn:workos:oauth:grant-type:email-verification:code`, `code` und `pending_authentication_token` |
| `POST /api/auth/login` {email, password} | `POST /user_management/authenticate` mit `grant_type=password`. Ist die Adresse noch nicht bestätigt → App zeigt wieder die Code-Eingabe. |
| `PUT /api/me/profile` | Vorname, Nachname, Stimmgruppe, Chor-ID in eurer Datenbank (Chor-ID eindeutig prüfen) |
| `GET /api/tasks`, `POST /api/tasks/:id/claim` | eure Datenbank |
| `PUT /api/tasks/:id` | eure Datenbank – nur mit Rolle `aufgabenverwalter` |
| `PUT /api/members/:id/active` | `PUT /user_management/organization_memberships/:id/deactivate` bzw. `/reactivate` – nur `mitgliedsverwalter`, nicht für sich selbst |
| `PUT /api/members/:id/roles` | `PUT /user_management/organization_memberships/:id` mit `role_slugs` – nur `mitgliedsverwalter`; sich selbst `mitgliedsverwalter` wegnehmen wird abgelehnt |
| `POST /api/tasks/:id/done`, `DELETE /api/tasks/:id/done` | eure Datenbank – übernommene Aufgabe als erledigt melden bzw. Meldung zurücknehmen |
| `GET /api/checks` | eure Datenbank – nur `aufgabenchecker`: alle offenen Meldungen |
| `POST /api/checks/:id/confirm` | eure Datenbank – Punkte gutschreiben (Aufgabencheck mit Chor-ID des Checkers); nicht für eigene Meldungen |
| `POST /api/checks/:id/reject` {comment} | eure Datenbank – Meldung abgelehnt, Kommentar optional |
| `PUT /api/members/:id/since` | eure Datenbank – nur `mitgliedsverwalter` |
| `GET /api/me/points` | eure Datenbank |
| `GET /api/members` | eure Datenbank. Für Benutzer ohne `mitgliedsverwalter` nur aktive Mitglieder und nur `chorId` + `voice` ausliefern. |

Das Backend gibt der App nach erfolgreicher Anmeldung eine Session (z. B. HttpOnly-Cookie) zurück.

## Rollen in WorkOS

- Einen Chor als **Organization** anlegen, jedes Mitglied hat dort eine Organization Membership.
- Im WorkOS-Dashboard die Rollen `mitgliedsverwalter`, `aufgabenverwalter`, `aufgabenchecker`, `terminverwalter` anlegen und **Multiple Roles** aktivieren (sonst nur eine Rolle pro Membership).
- Die App blendet Funktionen nur aus. Die eigentliche Prüfung muss das Backend machen.

## Punkte und Level

Alle Punkte werden mit der Währung „p“ angezeigt. Summe = zugesagt + gesammelt.
Level nach Summe: unter 20p Gast, 20–29p Mitglied, 30–39p Silber, ab 40p Gold.

## Datenmodell

- **Aufgabe**: id, title, category, interval (`M` Monat, `Q` Quartal, `J` Jahr, `J3` 3 Jahre), points, total (1–50), assigned (1–50), validFrom, validTo
- **Aufgabencheck**: taskId, date, points, checker (Chor-ID)
- **Meldung**: taskId, Mitglied, date, status (`reported` | `rejected`), comment, checker, decidedAt. Nach der Bestätigung wird daraus ein Aufgabencheck.
- **Mitglied**: firstName, lastName, voice (`B`, `T`, `A1`, `A2`, `S1`, `S2`, `U` = unbekannt), chorId, committed, collected, active, roles, since (Mitglied seit); fehlend = committed − collected
- **Chor-ID**: `^(B|T|A1|A2|S1|S2)-Name(-Name)*$`, Name nur aus Buchstaben (inkl. Umlaute)

## Dateien

- `index.html` – die komplette App (HTML, CSS, JavaScript)
- `manifest.webmanifest` – Name, Farben, Icons für den Home-Bildschirm
- `sw.js` – Offline-Cache und Updates
- `icons/` – App-Icons aus dem Logo
