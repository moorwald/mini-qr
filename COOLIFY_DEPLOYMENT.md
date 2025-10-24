# Coolify Deployment Anleitung

Diese Anleitung beschreibt, wie Mini-QR auf Coolify deployed wird.

## Wichtige Änderungen für Coolify

Die `docker-compose.yml` wurde für Coolify optimiert:

### 1. Nginx-Proxy entfernt
- **Grund**: Coolify verwendet bereits Caddy als Reverse Proxy
- Der separate `nginx-proxy` Service ist nicht mehr notwendig
- Das Routing wird direkt durch Coolify/Caddy übernommen

### 2. Healthcheck hinzugefügt
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

### 3. Port-Exposition
- Port 8080 wird mit `expose` im Docker-Netzwerk verfügbar gemacht
- **NICHT** mit `ports` exponiert, um direkte öffentliche Zugriffe zu vermeiden
- Coolify's Caddy-Proxy leitet Traffic basierend auf der Domain-Konfiguration weiter

### 4. Curl installiert
- Das Dockerfile installiert nun `curl` für den Healthcheck
- Coolify benötigt curl oder wget für Container-Healthchecks

## Deployment in Coolify

### Voraussetzungen
- Coolify-Installation
- Git-Repository mit diesem Code
- Docker Compose Build Pack aktiviert

### Deployment-Schritte

1. **Neues Projekt in Coolify erstellen**
   - Wähle "Docker Compose" als Deployment-Typ
   - Verbinde dein Git-Repository

2. **Umgebungsvariablen konfigurieren** (optional)

   Für eine Standard-Installation sind **keine** Umgebungsvariablen erforderlich.

   Falls Du die App anpassen möchtest, siehe die detaillierte Dokumentation:
   **[COOLIFY_ENV_VARS.md](./COOLIFY_ENV_VARS.md)** - Komplette Anleitung zu allen ENV-Variablen

   Quick-Beispiel für häufige Anpassungen:
   ```bash
   BASE_PATH=/                    # Standard: / (Root-Pfad)
   HIDE_CREDITS=false            # Credits anzeigen (true = ausblenden)
   DEFAULT_PRESET=plain          # Welcher Preset beim Start geladen wird
   DEFAULT_DATA=                 # Vorausgefüllter Text/URL (optional)
   DISABLE_LOCAL_STORAGE=false   # LocalStorage aktivieren
   ```

3. **Domain und Port konfigurieren** ⚠️ **WICHTIG!**

   Mini-QR läuft auf **Port 8080**. Du musst Coolify mitteilen, auf welchen Port geroutet werden soll:

   **Option A: Domain mit Port (Empfohlen)**
   - Im Domain-Feld: `deine-domain.com:8080`
   - Beispiel: `qr.tools.moorwald.com:8080`

   **Option B: Separates Port-Feld**
   - Im Domain-Feld: `deine-domain.com`
   - Im Port-Feld: `8080`

   - Coolify konfiguriert automatisch SSL/TLS mit Let's Encrypt
   - Die Domain wird über HTTPS erreichbar sein

4. **Deploy starten**
   - Coolify baut das Image und startet den Container
   - Der Healthcheck überwacht die Container-Gesundheit (~40 Sekunden Start-Zeit)
   - Caddy-Proxy routet Traffic automatisch zur Anwendung
   - Nach erfolgreicher Healthcheck ist die App unter der Domain erreichbar

## BASE_PATH Konfiguration

**Wichtig**: Der `BASE_PATH` wurde von `/mini-qr/` auf `/` geändert.

- **Grund**: Coolify's Caddy-Proxy übernimmt das Routing
- Die App wird direkt unter der Root-Domain erreichbar sein
- Falls ein Subpath gewünscht ist: `BASE_PATH` Umgebungsvariable in Coolify anpassen

## Healthcheck-Details

Der Healthcheck:
- Prüft alle 30 Sekunden, ob die Anwendung antwortet
- Wartet 40 Sekunden nach Container-Start (start_period)
- Gibt dem Container 10 Sekunden Zeit zum Antworten
- Nach 3 fehlgeschlagenen Versuchen gilt der Container als unhealthy

## Troubleshooting

### 502 Bad Gateway Error
**Symptom:** Container läuft, Healthcheck ist OK, aber Browser zeigt 502 Error

**Ursache:** Coolify's Caddy-Proxy weiß nicht, auf welchen Port geroutet werden soll

**Lösung:**

1. **Port in Coolify UI konfigurieren** (Schnellste Lösung)
   - Gehe zu deinem Service in Coolify
   - Ändere die Domain zu: `deine-domain.com:8080`
   - Oder trage im Port-Feld `8080` ein
   - Speichern und warten (~30 Sekunden)

2. **Wenn das nicht hilft:**
   - Redeploy durchführen
   - Container-Logs in Coolify prüfen
   - Caddy-Proxy-Logs prüfen (unter "Proxy" in Coolify)

**Technischer Hintergrund:**
- Coolify verwendet **Caddy docker-proxy**, nicht Traefik
- Die `docker-compose.yml` nutzt `expose` statt `ports`
- Caddy benötigt die Port-Information aus der Domain-Konfiguration

### Container startet nicht
- Prüfe die Build-Logs in Coolify
- Stelle sicher, dass alle Build-Args korrekt gesetzt sind

### Healthcheck schlägt fehl
- Überprüfe, ob der Container auf Port 8080 lauscht
- Prüfe die Container-Logs für Fehler
- Stelle sicher, dass `serve` korrekt startet
- Warte 40 Sekunden (start_period) nach Container-Start

### Routing-Probleme
- Prüfe die Coolify Domain-Konfiguration (Domain-Feld muss `:8080` enthalten)
- Stelle sicher, dass der richtige Port (8080) konfiguriert ist
- Überprüfe ob der Container im gleichen Docker-Netzwerk wie Caddy läuft
- Prüfe Caddy-Proxy-Logs in Coolify für Routing-Fehler
- Test: `curl http://localhost:8080` auf dem Server sollte die App anzeigen

## Netzwerk

- Das separate Docker-Netzwerk wurde entfernt
- Coolify verwaltet das Netzwerk automatisch
- Services können über Container-Namen kommunizieren (falls weitere Services hinzugefügt werden)

## Rolling Updates

**Hinweis**: Rolling Updates werden bei Docker Compose Deployments in Coolify nicht unterstützt.
- Bei Updates wird der alte Container gestoppt, bevor der neue startet
- Kurze Downtime während des Updates ist zu erwarten
