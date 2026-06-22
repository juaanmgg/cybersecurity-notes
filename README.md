# 🛡️ TryHackMe Writeups & Cybersecurity Notes

<div align="center">

![Cybersecurity](https://img.shields.io/badge/Cybersecurity-eJPTv2_Prep-000000?style=for-the-badge&logo=tryhackme&logoColor=white)
![Status](https://img.shields.io/badge/Status-Active-success?style=for-the-badge)
![Writeups](https://img.shields.io/badge/Writeups-7-blue?style=for-the-badge)

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
| **Kenobi** | 🟢 Fácil | 21/06/2026 | SMB & NFS Enum, ProFTPD (mod_copy), SSH Key Theft, PATH Variable Hijacking | [👉 Leer](./Linux/Kenobi.md) |
| **Bounty Hacker** | 🟢 Fácil | 22/06/2026 | Active Mode FTP (Raw netcat), SSH Brute-Force (Hydra), Sudo PrivEsc (tar checkpoint) | [👉 Leer](./Linux/BountyHacker.md) |

### 🪟 Entornos Windows

| Máquina | Dificultad | Fecha | Conceptos Clave & Técnicas | Writeup |
| :--- | :--- | :--- | :--- | :--- |
| **Blue** | 🟢 Fácil | 13/06/2026 | MS17-010 (EternalBlue), Metasploit, Hash dumping, NTLM cracking (John The Ripper) | [👉 Leer](./Windows/Blue.md) |
| **Ice** | 🟢 Fácil | 17/06/2026 | Icecast Exploit, Local Exploit Suggester, Process Migration (spoolsv.exe), Kiwi/Mimikatz | [👉 Leer](./Windows/Ice.md) |
| **Blaster** | 🟢 Fácil | 19/06/2026 | OSINT, RDP, CVE-2019-1388 (UAC Bypass), Metasploit Web Delivery, Persistence | [👉 Leer](./Windows/Blaster.md) |

---

## 🛠️ Metodología y Stack Técnico

La resolución de las máquinas sigue un ciclo de auditoría profesional estructurado, combinando explotación manual y automatizada:

* 🔍 **Reconocimiento & OSINT:** Análisis de superficie de ataque y enumeración de infraestructura con `nmap` (scripts NSE), `gobuster`, `enum4linux`, `smbclient` y monturas `NFS`.
* 💥 **Acceso Inicial:** Explotación de vulnerabilidades web (SQLi, *File Uploads*), manipulación de servicios FTP a bajo nivel (ej. *Raw Active Mode*, *mod_copy*), *Fuerza Bruta* con diccionarios customizados (`Hydra`), validación de CVEs (`searchsploit`) y ejecución *fileless* en memoria vía `Metasploit` (`web_delivery`).
* ⬆️ **Escalada de Privilegios:** Abuso de SUID/Sudo en Linux (GTFOBins, variables `$PATH`, *tar checkpoints*) y *bypasses* de UAC en Windows (ej. CVE-2019-1388), incluyendo inyección y migración de procesos (`spoolsv.exe`).
* 🔑 **Post-Explotación & Cracking:** Extracción de credenciales en memoria (`Kiwi/Mimikatz`), volcado de SAM, sustracción de claves SSH, persistencia en registro y ruptura offline (*NTLM/MD5*) con `Hashcat` y `John the Ripper`.

---

*Desarrollado por Juan Merino Garrido - Estudiante de Ingeniería Informática*
