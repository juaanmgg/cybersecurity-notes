# 🛡️ TryHackMe Writeups & Cybersecurity Notes

<div align="center">

![Cybersecurity](https://img.shields.io/badge/Cybersecurity-eJPTv2_Prep-000000?style=for-the-badge&logo=tryhackme&logoColor=white)
![Status](https://img.shields.io/badge/Status-Active-success?style=for-the-badge)
![Writeups](https://img.shields.io/badge/Writeups-5-blue?style=for-the-badge)

</div>

Repositorio personal donde documento mis resoluciones, metodologías y apuntes de máquinas de **TryHackMe** mientras me preparo para la certificación **eJPTv2**. 

El objetivo de estos *writeups* no es solo capturar la bandera, sino entender el **porqué** de cada vulnerabilidad, documentar comandos precisos y afianzar metodologías de Penetration Testing (Reconocimiento, Explotación, Post-Explotación).

## 🎯 Máquinas Resueltas

He clasificado las máquinas por sistema operativo. Puedes hacer clic en "Leer" para ver la metodología completa paso a paso.

### 🐧 Entornos Linux

| Máquina | Dificultad | Fecha | Conceptos Clave & Técnicas | Writeup |
| :--- | :--- | :--- | :--- | :--- |
| **RootMe** | 🟢 Fácil | 05/06/2026 | File Upload Bypass, Reverse Shell, PrivEsc (SUID), Gobuster, GTFOBins | [👉 Leer](./Linux/RootMe.md) |
| **Simple CTF** | 🟢 Fácil | 12/06/2026 | SQLi (Time-based), CMS Enum, Hashcat, Sudo Misconfig (vim), FTP Anon | [👉 Leer](./Linux/SimpleCTF.md) |

### 🪟 Entornos Windows

| Máquina | Dificultad | Fecha | Conceptos Clave & Técnicas | Writeup |
| :--- | :--- | :--- | :--- | :--- |
| **Blue** | 🟢 Fácil | 13/06/2026 | MS17-010 (EternalBlue), Metasploit, Hash dumping, NTLM cracking (John The Ripper) | [👉 Leer](./Windows/Blue.md) |
| **Ice** | 🟢 Fácil | 17/06/2026 | Icecast Exploit, Local Exploit Suggester, Process Migration (spoolsv.exe), Kiwi/Mimikatz | [👉 Leer](./Windows/Ice.md) |
| **Blaster** | 🟢 Fácil | 19/06/2026 | OSINT, RDP, CVE-2019-1388 (UAC Bypass), Metasploit Web Delivery, Persistence | [👉 Leer](./Windows/Blaster.md) |

---

## 🛠️ Herramientas y Metodología

Para la resolución de estas máquinas, aplico un enfoque estructurado simulando un entorno real de Penetration Testing, utilizando el siguiente arsenal técnico:

* 🔍 **Reconocimiento & Enumeración:** 
  * Escaneo de red y vulnerabilidades: `nmap` (TCP/UDP, Scripts NSE).
  * Enumeración de directorios web y OSINT: `gobuster`, análisis manual de fuentes.
  * Enumeración de servicios SMB/RPC: `enum4linux`, `smbmap`.
* 💥 **Acceso Inicial & Explotación:** 
  * Búsqueda e implementación de CVEs públicos (`searchsploit`).
  * Explotación de servicios web (Inyecciones SQL, *File Upload Bypasses*).
  * Interacción con servicios de escritorio remoto (`xfreerdp`).
  * Uso avanzado de `Metasploit Framework`, incluyendo ataques *fileless* en memoria (`web_delivery`).
* ⬆️ **Post-Explotación & Escalada de Privilegios (PrivEsc):** 
  * **Entornos Linux:** Enumeración manual de binarios SUID y configuraciones Sudo (GTFOBins), estabilización interactiva de TTY.
  * **Entornos Windows:** *Bypass* de controles UAC (ej. CVE-2019-1388), automatización de vectores locales (`local_exploit_suggester`), inyección y migración de procesos a servicios críticos (`spoolsv.exe`).
* 🔑 **Extracción de Credenciales y Persistencia:**
  * Volcado de bases de datos SAM y hashes locales (`hashdump`).
  * Extracción de credenciales cacheadas en memoria RAM mediante `Kiwi` (Mimikatz).
  * Creación de persistencia automatizada en el registro de Windows.
* 🔓 **Cracking:**
  * Ruptura de hashes NTLM y MD5 (con/sin *salt*) utilizando `Hashcat` y `John the Ripper`.

---

*Desarrollado por Juan Merino Garrido - Estudiante de Ingeniería Informática*
