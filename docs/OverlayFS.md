## (Overlay Cleaner Service einrichten)


### 1) Overlay Mount Service   
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



### 2) Aktivierung der Services

```bash
systemctl daemon-reload
systemctl enable --now overlay-cleaner.service
systemctl enable --now home-student.mount
```


#### Kurzlogik:

Overlay mounted - hängt von boot-run.service (und optional dem Stampfile und dem cleaner service)  ab.

