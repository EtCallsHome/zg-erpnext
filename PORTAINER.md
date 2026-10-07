# Einmalige Portainer-Umstellung

Diese Änderung macht spätere ERPNext-v15-Updates sehr einfach:

1. Backup erstellen.
2. In Portainer beim Stack `erp_next` auf **Update the stack** klicken.
3. Fertig - das neue Image wird gezogen und die Site automatisch migriert.

## 1. Image im bestehenden Stack ändern

Oben im vorhandenen Stack:

```yaml
x-erpnext-image: &erpnext_image
  image: ghcr.io/etcallshome/zg-erpnext:v15
  pull_policy: always
  restart: unless-stopped
```

Alle Frappe-Dienste benutzen bereits diesen Anchor und erhalten damit dasselbe Image.

## 2. Migrator-Service ergänzen

Unter den Services einen einmaligen Migrator ergänzen:

```yaml
  migrator:
    <<: *backend_defaults
    restart: "no"
    entrypoint:
      - bash
      - -lc
    command: 'bench --site invoice.etmail.de migrate && bench --site invoice.etmail.de clear-cache'
    depends_on:
      configurator:
        condition: service_completed_successfully
```

Der vorhandene `configurator` schreibt vor dem Migrator automatisch den aktuellen Inhalt des Image-App-Verzeichnisses nach `sites/apps.txt`. Dadurch bleiben `frappe`, `erpnext` und `eu_einvoice` synchron.

## 3. Frappe-Dienste erst nach erfolgreicher Migration starten

Bei `backend`, `websocket`, `queue-short`, `queue-long` und `scheduler` jeweils den bisherigen `depends_on: configurator`-Block durch folgenden Block ersetzen:

```yaml
    depends_on:
      migrator:
        condition: service_completed_successfully
```

`frontend` bleibt wie bisher von `backend` und `websocket` abhängig.

Damit startet die Anwendung erst, wenn `bench migrate` erfolgreich war.

## 4. GHCR-Paket einmalig freigeben

Das Container-Paket `ghcr.io/etcallshome/zg-erpnext:v15` muss für einen anonymen Pull aus Portainer öffentlich sein.

Wenn das Package nach dem ersten erfolgreichen GitHub-Action-Lauf privat ist:

- GitHub -> Profil -> Packages -> `zg-erpnext`
- Package settings
- Visibility auf **Public** stellen

Alternativ kann in Portainer eine authentifizierte GHCR-Registry hinterlegt werden.

## Künftiger Update-Ablauf

GitHub baut automatisch jeden Sonntag ein frisches v15-Image aus:

- Frappe `version-15`
- ERPNext `version-15`
- European e-Invoice `version-15`

Zusätzlich wird bei Änderungen an `apps.json` oder am Workflow sofort neu gebaut.

Für das Einspielen auf dem Produktivsystem:

1. Backup erstellen.
2. Sicherstellen, dass der GitHub-Action-Build grün ist.
3. Portainer -> Stacks -> `erp_next` -> **Update the stack**.
4. Kurz warten, bis `migrator` erfolgreich beendet ist und die normalen Container laufen.
5. ERPNext öffnen und kurz prüfen.

Es ist kein manuelles `docker build`, kein Image-Tag-Wechsel und kein manuelles `bench migrate` mehr nötig.

## Kontrolle nach einem Update

Im Backend-Container:

```bash
cd /home/frappe/frappe-bench && bench version
```

Erwartet:

```
erpnext 15.x
eu_einvoice 15.x
frappe 15.x
```

## Rollback

Jeder GitHub-Build wird zusätzlich als `build-<Nummer>` veröffentlicht.

Bei einem problematischen Update nicht nur das Image zurücksetzen, wenn bereits eine Datenbankmigration gelaufen ist. In diesem Fall das passende Backup zum vorherigen Versionsstand wiederherstellen.
