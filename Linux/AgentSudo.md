# Agent Sudo

Conceptos clave: HTTP Headers Spoofing (User-Agent), FTP Brute-Force (Hydra), Raw FTP Interaction, Steganography (Binwalk, Steghide), Zip Hash Cracking (John The Ripper), CVE-2019-14287 (Sudo Privilege Escalation)
Dificultad: Fácil
Fecha: 25 de junio de 2026

## Resumen
Compromiso de la máquina **Agent Sudo** (Linux) mediante el encadenamiento de enumeración web avanzada y análisis esteganográfico. El acceso inicial comenzó con la manipulación de cabeceras HTTP (`User-Agent`) para descubrir vectores ocultos. Tras obtener el nombre de un empleado, se realizó un ataque de fuerza bruta al servicio FTP para extraer material digital incrustado (imágenes y textos). El análisis de los ficheros (`binwalk`, `steghide`) y el crackeo de *hashes* de archivos comprimidos (`zip2john`) revelaron credenciales válidas para acceso por SSH. La escalada a `root` se logró explotando la vulnerabilidad **CVE-2019-14287** que afectaba a la configuración de Sudo en el sistema.

---

## 1. Reconocimiento (Recon)

### 1.1 Escaneo de Puertos y Servicios
Se mapeó la superficie de ataque para identificar los servicios expuestos:

```bash
nmap -p- --open -T4 -n -sS -v <IP>
nmap -p 21,22,80 -sC -sV <IP>
```

**Hallazgos:**
* `21/tcp`: FTP (vsftpd 3.0.3).
* `22/tcp`: SSH (OpenSSH 7.6p1).
* `80/tcp`: HTTP (Apache 2.4.29).

### 1.2 Enumeración Web y User-Agent Spoofing
Al acceder al servidor web en el puerto 80, la página principal instaba al usuario a utilizar su propio "User-Agent". Para investigar este comportamiento, se interceptó la petición HTTP `GET` utilizando **Burp Suite**.

En la pestaña *Repeater*, se manipuló la cabecera `User-Agent`. Tras varias pruebas con el abecedario, cambiar el valor a `C` generó una redirección a una página oculta que revelaba un nombre de usuario: `chris`.

---

## 2. Acceso Inicial (Initial Access)

### 2.1 Fuerza Bruta a FTP
Teniendo un nombre de usuario válido (`chris`), se procedió a realizar un ataque de fuerza bruta de diccionario contra el servicio FTP utilizando **Hydra**:

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://<IP>
```
**Resultado:** Se descubrió la contraseña `crystal`.

### 2.2 Exfiltración mediante Raw FTP (Modo Activo)
Se inició sesión en el servidor FTP. Demostrando control sobre el protocolo a bajo nivel, se interactuó con `netcat` configurando un canal en **Modo Activo** (`PORT`) en lugar del modo pasivo habitual.

Se exfiltraron con éxito tres ficheros hacia la máquina atacante para su análisis local:
* `To_agent_J.txt`
* `cutie.png`
* `cute-alien.jpg`

---

## 3. Análisis Esteganográfico y Obtención de Credenciales

### 3.1 Binwalk y Crackeo de ZIP
Al no encontrar información evidente ejecutando `strings` sobre las imágenes, se procedió a analizar sus firmas con `binwalk`.

```bash
binwalk -e cutie.png
```
El análisis extrajo varios componentes (zlib) y un archivo comprimido `.zip` oculto en la imagen, el cual estaba protegido por contraseña. Para romper esta seguridad, se extrajo el *hash* del archivo y se procedió a su crackeo offline:

```bash
zip2john 8702.zip > hash_zip.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash_zip.txt
```
**Resultado:** Se crackeó el archivo descubriendo la contraseña `alien`. 

### 3.2 Steghide y Decodificación Base64
Al descomprimir el archivo con la contraseña obtenida, se reveló un texto que contenía una cadena codificada en Base64 (`QXJlYTUx`), la cual se decodificó a `Area51`.

Esta cadena resultó ser la *passphrase* necesaria para extraer la información oculta en la segunda imagen utilizando `steghide`:

```bash
steghide extract -sf cute-alien.jpg -p Area51
```
El proceso extrajo el archivo `message.txt`, el cual contenía la contraseña en texto plano (`hackerrules!`) del usuario `james`.

### 3.3 Acceso SSH
Con las credenciales (`james` : `hackerrules!`), se estableció conexión SSH, logrando el acceso al sistema y capturando la bandera `user.txt`.

---

## 4. Escalada de Privilegios (Privilege Escalation)

### 4.1 Enumeración Sudo
Se evaluaron los permisos administrativos del usuario `james`:

```bash
sudo -l
# Output relevante: (ALL, !root) /bin/bash
```
La configuración indicaba que `james` podía ejecutar `/bin/bash` como cualquier usuario, **excepto** como `root`.

### 4.2 Explotación de CVE-2019-14287 (Sudo Security Bypass)
Esta restricción específica (`!root`) es vulnerable a una falla lógica en el manejo de IDs de usuario en Sudo (CVE-2019-14287). Al pasar el ID de usuario `-1` (o `4294967295`), la función encargada de comprobar los permisos falla, y Sudo trata internamente el valor como `0` (el ID de `root`), evadiendo la política restrictiva.

Se ejecutó el siguiente *payload*:

```bash
sudo -u#-1 /bin/bash
```

**Resultado:** El *bypass* fue exitoso, proporcionando una *shell* interactiva con privilegios completos de `root`. Se accedió al directorio principal y se capturó la bandera `root.txt`, completando la máquina.
