# Silent Monitor — mi write-up

Room de **TryHackMe** (medium). Cadena: enum web → SQL injection (bypass login) → command injection (`%0A`) → credenciales → SSH → privesc por kernel (`copy_fail`).

Room: https://tryhackme.com/room/silentmonitor

---

## Cómo empecé

Escaneo de puertos:

```bash
nmap -sV -sC -Pn -T4 <IP>
```

![nmap](./silentMonitor(nmap2).png)

Salieron **SSH** y **HTTP**. La web es el portal del NOC (CorpNet); a primera vista no hay mucho.

![página](./silentMonitor(pagina).png)

Enumeré directorios y apareció **`/internal`**:

![fuzz](./silentMonitor(Fuzz).png)

---

## Bypass del login (SQL injection)

`/internal` pide login. Con inyección SQL clásica en el formulario el bypass funciona y entras al panel interno.

Ejemplo de payload (username):

```sql
' OR 1=1 --
```

![login SQLi](./silentMonitor(login).png)

---

## Command injection en Host Health

Dentro hay una sección para verificar conectividad a un host (ping / health check).

![Host Health](./silentMonitor(HostHealth).png)

Los separadores típicos (`;`, `|`, `&`) están filtrados. Con **Burp** inyecté un salto de línea URL-encoded (`%0A`) después del host y encadené un comando. 

Payload (idea):

```http
target=127.0.0.1%0Als
```

![command injection](./silentMonitor(burpsuite2).png)

Listando archivos se ve `secret.config`. Lo leí por la misma inyección y sacó credenciales:

![secret.config](./silentMonitor(burpsuite3).png)

---

## User flag (SSH)

Con esas credenciales entré por SSH como **sysadmin** y leí la primera flag.

```bash
ssh sysadmin@<IP>
```

![user flag](./silentMonitor(sysadmin2).png)

---

## Root (kernel / copy_fail)

En reconocimiento local miré la versión del kernel y busqué CVEs aplicables.

![kernel](./silentMonitor(sysadmin3).png)

Encontré el exploit **copy_fail** para esa versión. Lo llevé a la máquina, lo ejecuté y obtuve shell como **root** — ahí está la segunda flag.

![root](./silentMonitor(root).png)

---

## Resumen

| Paso | Técnica |
|------|---------|
| Enum | nmap + fuzz de directorios → `/internal` |
| Acceso web | SQL injection en el login |
| RCE / lectura | Command injection vía `%0A` en Host Health |
| User | Creds en `secret.config` → SSH |
| Root | Exploit de kernel `copy_fail` |
