# ZG ERPNext v15 + E-Rechnung

Dieses Repository baut ein eigenes ERPNext-v15-Docker-Image mit:

- Frappe `version-15`
- ERPNext `version-15`
- `alyf-de/eu_einvoice` `version-15`

## Fertiges Image

```
ghcr.io/etcallshome/zg-erpnext:v15
```

Zusätzlich bekommt jeder Build einen unveränderlichen Tag:

```
ghcr.io/etcallshome/zg-erpnext:build-<Nummer>
```

## Wann wird gebaut?

Der Workflow baut automatisch:

- sonntags um 03:17 UTC
- wenn `apps.json` geändert wird
- wenn der Build-Workflow geändert wird
- manuell über GitHub Actions

## Update auf dem ERPNext-Server

Der eigentliche Produktivserver wird absichtlich **nicht automatisch** aktualisiert.

Damit ein ERPNext-Update erst nach einem Backup eingespielt wird:

1. ERPNext-Backup ausführen.
2. In Portainer den Stack `erp_next` öffnen.
3. **Update the stack** ausführen.
4. Das Stack-Image muss auf `ghcr.io/etcallshome/zg-erpnext:v15` stehen und `pull_policy: always` verwenden.
5. Der `migrator`-Service führt `bench --site invoice.etmail.de migrate` und anschließend `clear-cache` automatisch aus.
6. Nach dem Update in ERPNext kurz Version und Grundfunktionen prüfen.

## Kontrolle

Im Backend-Container:

```bash
cd /home/frappe/frappe-bench && bench version
```

Erwartet werden drei Apps:

```
erpnext ...
eu_einvoice ...
frappe ...
```

Installierte Apps der Site:

```bash
cd /home/frappe/frappe-bench && bench --site invoice.etmail.de list-apps
```

## Rollback

Die `build-<Nummer>`-Tags bleiben als Rücksprungpunkte erhalten.

Wichtig: Nach einer Datenbankmigration nicht nur das Docker-Image zurückdrehen. Bei einem echten Rollback auch das zum alten Stand passende Backup wiederherstellen.
