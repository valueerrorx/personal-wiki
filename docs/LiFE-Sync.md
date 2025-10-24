## (Overlay Cleaner Service einrichten)

### Client

### 1) Verzeichnisse erstellen

```bash
mkdir -p /var/lib/boot-fetcher/queue /var/lib/boot-fetcher/done /var/lib/boot-fetcher/work
```


### 2) Fetch-Service 
(holt Dateien vom http server auf port 8080)

**Datei**: `/usr/local/bin/boot-fetch.sh`
```bash
`#!/usr/bin/env bash`
`set -euo pipefail                                   # fail fast`
`SERVER_IP="10.40.0.1"                               # <- fixe Server-IP`
`PORT="8080"                                         # <- HTTP-Port`
`REMOTE="queue"                                      # <- Server-Unterordner`
`BASE="http://${SERVER_IP}:${PORT}/${REMOTE}"        # base url`

`STATE="/var/lib/boot-fetcher"                       # state dir`
`QUEUE="${STATE}/queue"                              # queue for scripts`
`WORK="${STATE}/work"                                # temp dir`

`mkdir -p "${QUEUE}" "${WORK}"                       # ensure dirs`

`curl -fsS "${BASE}/scripts.txt" -o "${WORK}/scripts.txt"   # fetch manifest`
`mapfile -t FILES < <(grep -Ev '^\s*$|^\s*#' "${WORK}/scripts.txt" | sort -u)  # parse list`

`for f in "${FILES[@]}"; do`
  `curl -fsS "${BASE}/${f}" -o "${QUEUE}/${f}"      # download each listed file`
  `chmod +x "${QUEUE}/${f}"                         # ensure executable`
`done`
```

`chmod +x /usr/local/bin/boot-fetch.sh`


**Datei**: `/etc/systemd/system/boot-fetch.service`
```ini
[Unit]
Description=Fetch boot scripts from server (no execution)
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/boot-fetch.sh
TimeoutStartSec=30s                                         
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```


### 3) Run-Service 
(führt aus, pflegt „done“, signalisiert Fertig)

**Datei**: `/usr/local/bin/boot-run.sh`
```bash
#!/usr/bin/env bash
set -euo pipefail                                   # fail fast
STATE="/var/lib/boot-fetcher"                       # state dir
QUEUE="${STATE}/queue"                              # queue with scripts
DONE="${STATE}/done"                                # done dir
STAMP="/run/boot-runner.done"                       # per-boot completion stamp

mkdir -p "${QUEUE}" "${DONE}"                       # ensure dirs
shopt -s nullglob                                   # empty glob -> no iteration

for s in "${QUEUE}"/*.sh; do
  name="$(basename "$s")"                           # script base name
  [[ -e "${DONE}/${name}" ]] && { rm -f "$s"; continue; }   # skip if already done, clean queue
  if bash "$s"; then                                # execute as root
    mv -f "$s" "${DONE}/${name}"                    # move to done (no duplicate in queue)
  fi
done

touch "${STAMP}"                                    # signal: all done this boot

```

`chmod +x /usr/local/bin/boot-run.sh`


**Datei**: `/etc/systemd/system/boot-run.service`
```ini
[Unit]
Description=Execute queued boot scripts and mark them done
Wants=boot-fetch.service
After=boot-fetch.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/boot-run.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```


### 4) Overlay Mount Service   
(mounted das overlayFS nach /home/student - nachdem der Run-Service fertig ist. Auf diese Weise können auch im Home Verzeichnis Änderungen durchgeführt werden)


**Datei**: `/etc/systemd/system/home-student.mount`
```ini
[Unit]
Description=OverlayFS for /home/student
After=overlay-cleaner.service boot-run.service
Requires=overlay-cleaner.service boot-run.service
ConditionPathExists=/run/boot-runner.done
RequiresMountsFor=/mnt/upper /mnt/work

[Mount]
What=overlay
Where=/home/student
Type=overlay
Options=lowerdir=/home/student,upperdir=/mnt/upper,workdir=/mnt/work

[Install]
WantedBy=multi-user.target
```

**Datei**: `/etc/systemd/system/overlay-cleaner.service`
```ini
[Unit]
Description=Delete overlay upper/work before mount
After=local-fs.target
Before=home-student.mount

[Service]
Type=oneshot
ExecStart=/bin/rm -rf /mnt/upper /mnt/work || true
ExecStartPost=/bin/mkdir -p /mnt/upper /mnt/work || true
ExecStartPost=/bin/chown -R student:student /mnt/upper /mnt/work || true
RemainAfterExit=true    

[Install]
WantedBy=home-student.mount
``` 



### 5) Aktivierung der Services

```bash
systemctl daemon-reload
systemctl enable --now boot-fetch.service
systemctl enable --now boot-run.service
systemctl enable --now overlay-cleaner.service
systemctl enable --now home-student.mount
```


#### Kurzlogik:

Fetch holt → 
Run checkt ob Script neu ist →
Run führt aus wenn neu
Run kopiert Script nach ./done & setzt /run/boot-runner.done → 
Overlay mounted - hängt von boot-run.service (und optional dem Stampfile und dem cleaner service)  ab.




# <br>



# Server


### 0) Basis
Feste Server-IP notieren (z. B. 10.40.0.1)
Port wählen (z. B. 8080)
### 1) User & Verzeichnisse

```bash
sudo useradd -r -s /usr/sbin/nologin bootscripts      # system user (no shell)
sudo mkdir -p /srv/bootscripts/queue
sudo chown -R bootscripts:bootscripts /srv/bootscripts
sudo chmod -R 755 /srv/bootscripts
```


### 2) Skripte ablegen
`cd /srv/bootscripts/queue`

Beispiel:
```bash
sudo printf '#!/usr/bin/env bash\necho hello\n' > 001-init.sh
```


### 3) Manifest erzeugen
(nur die Skripte die in scripts.txt vermerkt sind werden vom Client auch heruntergeladen und ausgeführt)

```bash
cd /srv/bootscripts/queue
ls -1 *.sh 2>/dev/null | sort -V > scripts.txt            # one filename per line
```


### 4) HTTP-Server als systemd-Service

**Datei:** `/etc/systemd/system/bootscripts-http.service`
```ini
[Unit]
Description=Serve /srv/bootscripts via Python HTTP
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=bootscripts
Group=bootscripts
WorkingDirectory=/srv/bootscripts
ExecStart=/usr/bin/python3 -m http.server 8080 --bind 0.0.0.0
Restart=on-failure
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=true

[Install]
WantedBy=multi-user.target
```

### 5) Service aktivieren

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now bootscripts-http.service
```
