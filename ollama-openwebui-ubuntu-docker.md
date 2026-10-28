# Ollama + Open WebUI auf Ubuntu mit Docker

Anleitung, um auf einer Ubuntu-VM (getestet mit Ubuntu 26.04 „resolute“) per Docker **Ollama** (lokale KI-Modelle) und **Open WebUI** (Chat-Oberfläche im Browser) zu installieren.

> **Kein Docker Desktop nötig!** Auf Ubuntu, besonders in einer VM, nimmt man die schlanke **Docker Engine**. Docker Desktop würde intern selbst eine VM starten und eine grafische Oberfläche brauchen.

---

## Voraussetzungen

- Ubuntu-VM mit Internetzugang
- Empfohlen: **mind. 8 GB RAM** und mehrere CPU-Kerne (ohne GPU läuft alles auf der CPU)
- Ein Benutzer mit `sudo`-Rechten

---

## 1. Docker Engine installieren

### Alte bzw. inoffizielle Pakete entfernen (falls vorhanden)

```bash
sudo apt remove docker.io docker-compose docker-doc podman-docker containerd runc 2>/dev/null
```

### Docker-Repository einrichten

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Jetzt die Docker-Paketquelle eintragen. **Wichtig: alles in einer Zeile!**

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Der Befehl gibt keine Ausgabe aus, das ist normal.

**Was macht der Befehl?** Er trägt das offizielle Docker-Repository als neue Paketquelle für `apt` ein. Damit bekommt man die aktuelle Docker-Version direkt vom Hersteller (inkl. `docker compose`), und Updates kommen künftig ganz normal über `sudo apt upgrade`.

- `arch=...` setzt automatisch die CPU-Architektur ein (meist `amd64`)
- `signed-by=...` verweist auf den Schlüssel, mit dem `apt` prüft, dass die Pakete echt sind
- `$(. /etc/os-release && echo $VERSION_CODENAME)` setzt den Codenamen der Ubuntu-Version ein
- `sudo tee ...` schreibt die Zeile mit Admin-Rechten in die Datei

Kontrolle:

```bash
cat /etc/apt/sources.list.d/docker.list
```

### Docker installieren

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Bei `apt update` sollte eine Zeile mit `download.docker.com` auftauchen.

### Eigenen Benutzer zur docker-Gruppe hinzufügen

Damit man Docker ohne `sudo` benutzen kann:

```bash
sudo usermod -aG docker $USER
```

Das greift erst nach einer neuen Anmeldung. Am einfachsten:

```bash
sudo reboot
```

### Testen

```bash
docker run hello-world
```

Erscheint **„Hello from Docker!“**, ist alles bereit.

---

## 2. Ollama + Open WebUI per Docker Compose

### Ordner und Datei anlegen

```bash
mkdir ~/ai
cd ~/ai
nano docker-compose.yml
```

Folgenden Inhalt einfügen (im Terminal: **Strg+Umschalt+V**):

```yaml
services:
  ollama:
    image: ollama/ollama
    container_name: ollama
    restart: unless-stopped
    volumes:
      - ollama:/root/.ollama
    ports:
      - "11434:11434"

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    depends_on:
      - ollama
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - open-webui:/app/backend/data
    ports:
      - "3000:8080"

volumes:
  ollama:
  open-webui:
```

Speichern und schließen in nano:

- **Strg+O**, dann **Enter** (speichern)
- **Strg+X** (beenden)

> **Achtung:** YAML ist pingelig. Einrückungen müssen erhalten bleiben, und Tippfehler (z. B. `enviroment` statt `environment`) führen zu Fehlern.

### Container starten

```bash
docker compose up -d
```

Beim ersten Mal werden mehrere GB heruntergeladen, das dauert ein paar Minuten. Am Ende sollten `ollama` und `open-webui` als **Started** angezeigt werden.

---

## 3. Open WebUI öffnen

Open WebUI läuft auf **Port 3000**.

`"3000:8080"` bedeutet: links der Port auf der VM, rechts der Port im Container. Man spricht immer die **3000** an.

- **Browser direkt in der VM:** `http://localhost:3000`
- **Browser auf einem anderen Rechner im Netz:** `http://<IP-der-VM>:3000`
  (IP herausfinden in der VM mit `hostname -I`)

Beim ersten Start kann es 1–2 Minuten dauern, bis die Seite erscheint. Dann einen **Admin-Account** anlegen.

---

## 4. Modell herunterladen

```bash
docker exec -it ollama ollama pull llama3.2
```

Danach die Seite neu laden, oben das Modell auswählen und loschatten.

Ohne GPU eignen sich vor allem kleine Modelle (z. B. `llama3.2` oder `qwen2.5:3b`). Große Modelle laufen auf der CPU sehr langsam.

---

## Nützliche Befehle

Alle Befehle im Ordner `~/ai` ausführen.

| Befehl | Wirkung |
|---|---|
| `docker compose ps` | Status der Container anzeigen |
| `docker compose logs -f open-webui` | Logs von Open WebUI live ansehen |
| `docker compose stop` | Container anhalten |
| `docker compose up -d` | Container (wieder) starten |
| `docker compose pull && docker compose up -d` | Auf neueste Versionen aktualisieren |
| `docker exec -it ollama ollama list` | Installierte Modelle anzeigen |
| `docker exec -it ollama ollama rm <modell>` | Modell löschen |

Daten (Modelle, Chats, Accounts) liegen in Docker-Volumes und bleiben bei Neustarts und Updates erhalten.

---

## Stolpersteine

- **Befehl mit `\` in einer Zeile kopiert:** Der Backslash bedeutet „geht in der nächsten Zeile weiter“. Steht alles in einer Zeile, den `\` entfernen.
- **`permission denied ... docker.sock`:** Die docker-Gruppe ist noch nicht aktiv. Ab- und wieder anmelden bzw. neu starten.
- **`newgrp` nicht gefunden:** Auf neueren Ubuntu-Versionen nicht vorinstalliert. Einfach neu starten statt `newgrp`.
- **Port 11434** muss nur nach außen freigegeben sein, wenn man Ollama auch von anderen Rechnern direkt ansprechen will. Sonst können die `ports`-Zeilen beim `ollama`-Dienst entfallen.
- **GPU in der VM:** Funktioniert nur mit GPU-Passthrough plus NVIDIA Container Toolkit, das ist ein eigenes Thema.
