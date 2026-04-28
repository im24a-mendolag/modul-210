# 3.2 – HTML-Webseite mit nginx und Docker

## Was wurde gemacht?

Eine einfache HTML-Webseite (`index.html`) wird lokal über einen **nginx**-Container ausgeliefert.

---

## Voraussetzungen

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) ist installiert und läuft

---

## Container starten

```powershell
docker run -d --name bzz210-nginx -p 8080:80 -v "c:/ims/bzz/210/3/3.2/index.html:/usr/share/nginx/html/index.html:ro" nginx:latest
```

**Webseite aufrufen:** http://localhost:8080

---

## Befehle erklärt

| Flag | Bedeutung |
|---|---|
| `-d` | Container läuft im Hintergrund (detached) |
| `--name bzz210-nginx` | Name des Containers |
| `-p 8080:80` | Port 8080 am PC → Port 80 im Container |
| `-v "..."` | Bindet die lokale HTML-Datei in den Container ein |
| `:ro` | Read-only – der Container kann die Datei nur lesen |
| `nginx:latest` | Image von Docker Hub |

---

## Container stoppen und entfernen

```powershell
docker stop bzz210-nginx; docker rm bzz210-nginx
```

## Container neu starten (falls gestoppt, aber noch vorhanden)

```powershell
docker start bzz210-nginx
```

---

## Beliebige HTML-Datei ausliefern

Das `-v`-Flag kann auf jede beliebige HTML-Datei zeigen:

```powershell
docker run -d --name mein-container -p 8080:80 -v "C:/Pfad/zu/deiner/datei.html:/usr/share/nginx/html/index.html:ro" nginx:latest
```

Einfach `C:/Pfad/zu/deiner/datei.html` durch den Pfad zur gewünschten Datei ersetzen.

---

## Image auf GitHub Container Registry (ghcr.io) pushen

### Voraussetzungen
- GitHub Personal Access Token (classic) mit den Scopes: `repo`, `write:packages`, `read:packages`

### 1. Login
```powershell
docker login ghcr.io -u im24a-mendolag
```
Token (`ghp_...`) als Passwort eingeben.

### 2. Image bauen
```powershell
docker build -t ghcr.io/im24a-mendolag/bzz210-web:latest c:/ims/bzz/210/3/3.2
```

### 3. Image pushen
```powershell
docker push ghcr.io/im24a-mendolag/bzz210-web:latest
```

Das Image ist danach sichtbar unter: **github.com/im24a-mendolag?tab=packages**

### 4. Image von ghcr.io pullen und starten
```powershell
docker pull ghcr.io/im24a-mendolag/bzz210-web:latest
docker run -d -p 8080:80 --name web01 ghcr.io/im24a-mendolag/bzz210-web:latest
```

Webseite aufrufen: http://localhost:8080
