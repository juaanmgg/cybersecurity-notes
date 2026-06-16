# Simple CTF

Conceptos clave: Anonymous FTP, CMS enumeration, Hashcat, Port scanning, Reconnaissance, SQL Injection (time-based), SSH (non-standard port), Searchsploit, Service enumeration, Sudo misconfiguration (vim), Web enumeration
Dificultad: Fácil
Fecha: 12 de junio de 2026

## Resumen

Compromiso completo de la máquina **Simple CTF** mediante enumeración de servicios, explotación de **SQL Injection time-based** en **Simple CMS 2.2.8**, crackeo de credenciales y escalada a root por configuración insegura de **sudo** sobre **vim**.

## 1. Reconocimiento (Recon)

### 1.1 Conectividad

- Verificación inicial de alcance:

```bash
ping -c 1 <IP>
```

### 1.2 Escaneo de puertos (full scan)

- Escaneo completo de puertos TCP para identificar superficie de ataque (exportado a fichero):

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn -oN allPorts <IP>
```

**Puertos abiertos identificados:**

- `21/tcp` (FTP)
- `80/tcp` (HTTP)
- `2222/tcp` (SSH en puerto no estándar)

### 1.3 Enumeración de servicios

- Enumeración con scripts por defecto y detección de versión sobre los puertos expuestos:

```bash
nmap -sC -sV -p 21,80,2222 -oN scanPorts <IP>
```

## 2. Acceso Inicial (Initial Access)

### 2.1 FTP (anonymous)

- Acceso al servicio FTP para comprobar ficheros públicos:

```bash
ftp <IP>
# user: anonymous
# pass: anonymous
```

Hallazgo relevante:

- Aparición de un posible usuario: **mitch**

### 2.2 Enumeración web

- Enumeración inicial de rutas y descubrimiento de:
    - `index.php`
    - Directorio `/simple/` con **Simple CMS v2.2.8**
- Enumeración recursiva del CMS para mapear rutas relevantes (panel/admin, uploads, etc.). Se localizaron directorios típicos como:
    - `assets/`
    - `admin/` (login de administración)
    - `uploads/`
    - `tmp/`
    - `content/`

### 2.3 Búsqueda y validación de exploit

- Búsqueda de exploits públicos para la versión identificada:

```bash
searchsploit "Simple CMS 2.2.8"
```

Resultado:

- Script en **Python** orientado a explotar una **SQL Injection (time-based)**.

Acciones realizadas antes de ejecutar:

- Revisión del código para entender el flujo y parámetros.
- Adaptación a compatibilidad (el script estaba en Python 2):
    - Ajuste de `print` (paréntesis) para ejecución.
- Ajuste del valor de `time` a `2` para acelerar el time-based y evitar esperas innecesarias.

### 2.4 Explotación (SQLi) y obtención de credenciales

Tras ejecutar el exploit, se obtuvo información de usuarios, incluyendo:

- Usuario: **mitch**
- `salt` y hash de contraseña
- Email asociado

### 2.5 Crackeo de hash (hashcat)

- Crackeo con **hashcat modo 20** (MD5 con salt):

```bash
hashcat -m 20 <hashes> <wordlist>
```

Credencial recuperada:

- Contraseña: `secret`

### 2.6 Acceso por SSH (puerto 2222)

- Acceso remoto con las credenciales obtenidas:

```bash
ssh mitch@<IP> -p 2222
```

Post-explotación:

- Obtención de la primera bandera `user.txt`.

## 3. Escalada de Privilegios (Privilege Escalation)

### 3.1 Enumeración de privilegios (sudo)

- Comprobación de permisos sudo:

```bash
sudo -l
```

Hallazgo:

- El usuario **mitch** podía ejecutar **vim** como **root**.

### 3.2 Explotación (GTFOBins: vim)

- Obtención de shell con privilegios mediante vim:

```bash
sudo vim -c ':sh'
```

Resultado:

- Shell como **root** y obtención de la bandera `root.txt`.

## 4. Flags

- `user.txt`: obtenida tras acceso SSH como `mitch`.
- `root.txt`: obtenida tras escalada a root mediante `sudo vim`.