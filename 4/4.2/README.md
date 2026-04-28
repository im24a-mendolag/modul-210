# 4.2 – Docker Compose

## Tutorial: Docker Compose Getting Started

Quelle: https://docs.docker.com/compose/gettingstarted/

---

## Verwendete Container

Das Tutorial verwendet zwei Container:

```
┌─────────────────────────────────────────────────────┐
│                   Docker Network                    │
│                                                     │
│  ┌──────────────────────┐   ┌─────────────────────┐ │
│  │      web             │   │       redis         │ │
│  │  Python 3.12-Alpine  │──▶│   redis:alpine      │ │
│  │  Flask App           │   │   Port 6379         │ │
│  │  Port 5000           │   │   Volume: redis-data│ │
│  └──────────────────────┘   └─────────────────────┘ │
│           │                                         │
└───────────┼─────────────────────────────────────────┘
            │
     Host: Port 8000
```

| Container | Image | Aufgabe |
|---|---|---|
| **web** | Python 3.12-Alpine (selbst gebaut) | Flask-Webapp, zählt Seitenaufrufe |
| **redis** | `redis:alpine` (Docker Hub) | Speichert den Zählerstand persistent |

Der `web`-Container verbindet sich mit `redis` über den internen Docker-Netzwerknamen `redis` auf Port `6379`. Der Zählerstand (`hits`) wird bei jedem Seitenaufruf um 1 erhöht.

---

## Fragen

### Was ist Redis?
Redis ist eine **In-Memory-Datenbank** (Key-Value-Store), die Daten extrem schnell im Arbeitsspeicher speichert. Im Tutorial dient Redis als einfacher Zähler-Speicher: Die Flask-App ruft `cache.incr("hits")` auf, und Redis erhöht den Wert atomar. Mit einem Volume (`redis-data:/data`) bleibt der Zählerstand auch nach einem Container-Neustart erhalten.

### Welche Ports werden genutzt?

| Port | Wo | Zweck |
|---|---|---|
| `8000` | Host → Container | Erreichbarkeit der Flask-App im Browser |
| `5000` | Intern im Container | Flask läuft auf Port 5000 |
| `6379` | Intern (web → redis) | Standard-Redis-Port, nur intern erreichbar |

### Was ist die Bedeutung von ENV im Dockerfile?
`ENV` setzt **Umgebungsvariablen** im Container-Image, die beim Start des Containers verfügbar sind:

```dockerfile
ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0
```

- `FLASK_APP=app.py` teilt Flask mit, welche Datei die App-Instanz enthält.
- `FLASK_RUN_HOST=0.0.0.0` sorgt dafür, dass Flask auf allen Netzwerkinterfaces lauscht (nicht nur `localhost`), damit der Container von aussen erreichbar ist.
- Ohne `0.0.0.0` wäre die App nur innerhalb des Containers erreichbar – Port-Mapping würde nicht funktionieren.

---

## Docker Compose File für Auftrag 3.2

Die Datei `docker-compose.yml` in diesem Ordner liefert die HTML-Seite aus Auftrag 3.2 mit nginx aus.

### Starten

```powershell
docker compose up -d
```

Webseite aufrufen: http://localhost:8080

### Stoppen

```powershell
docker compose down
```

### docker-compose.yml

```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ../3.2/index.html:/usr/share/nginx/html/index.html:ro
```

**Erklärung:**
- `image: nginx:latest` — verwendet das offizielle nginx-Image von Docker Hub
- `ports: "8080:80"` — Port 8080 am PC wird auf Port 80 im Container weitergeleitet
- `volumes` — bindet die HTML-Datei aus Auftrag 3.2 read-only in den Container ein
