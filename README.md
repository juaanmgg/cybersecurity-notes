# 🛡️ TryHackMe Writeups & Cybersecurity Notes

<div align="center">

![Cybersecurity](https://img.shields.io/badge/Cybersecurity-eJPTv2_Prep-000000?style=for-the-badge&logo=tryhackme&logoColor=white)
![Status](https://img.shields.io/badge/Status-Active-success?style=for-the-badge)
![Writeups](https://img.shields.io/badge/Writeups-11-blue?style=for-the-badge)

</div>

Repositorio personal donde documento mis resoluciones, metodologías y apuntes de máquinas de **TryHackMe** mientras me preparo para la certificación **eJPTv2**. 

El objetivo de estos *writeups* no es solo capturar la bandera, sino entender el **porqué** de cada vulnerabilidad, documentar comandos precisos y afianzar metodologías de Penetration Testing (Reconocimiento, Explotación, Post-Explotación).

## 🎯 Máquinas Resueltas

He clasificado las máquinas por sistema operativo y entorno. Puedes hacer clic en "Leer" para ver la metodología completa paso a paso.

### 🐧 Entornos Linux

| Máquina | Dificultad | Fecha | Conceptos Clave & Técnicas | Writeup |
| :--- | :--- | :--- | :--- | :--- |
| **RootMe** | 🟢 Fácil | 05/06/2026 | File Upload Bypass, Reverse Shell, PrivEsc (SUID), Gobuster, GTFOBins | [👉 Leer](./Linux/RootMe.md) |
| **Simple CTF** | 🟢 Fácil | 12/06/2026 | SQLi (Time-based), CMS Enum, Hashcat, Sudo Misconfig (vim), FTP Anon | [👉 Leer](./Linux/SimpleCTF.md) |
| **Kenobi** | 🟢 Fácil | 21/06/2026 | SMB & NFS Enum, ProFTPD (mod_copy), SSH Key Theft, PATH Variable Hijacking | [👉 Leer](./Linux/Kenobi.md) |
| **Bounty Hacker** | 🟢 Fácil | 22/06/2026 | Active Mode FTP (Raw netcat), SSH Brute-Force (Hydra), Sudo PrivEsc (tar checkpoint) | [👉 Leer](./Linux/BountyHacker.md) |
| **Pickle Rick** | 🟢 Fácil | 23/06/2026 | OSINT (Source Code), Hardcoded Creds, Authenticated RCE, Reverse Shell, Sudo (ALL) | [👉 Leer](./Linux/PickleRick.md) |
| **Agent Sudo** | 🟢 Fácil | 25/06/2026 | HTTP Header Spoofing, Steganography (Binwalk/Steghide), Zip Hash Cracking, CVE-2019-14287 | [👉 Leer](./Linux/AgentSudo.md) |

### 🪟 Entornos Windows & Active Directory

| Máquina | Dificultad | Fecha | Conceptos Clave & Técnicas | Writeup |
| :--- | :--- | :--- | :--- | :--- |
| **Blue** | 🟢 Fácil | 13/06/2026 | MS17-010 (EternalBlue), Metasploit, Hash dumping, NTLM cracking (John The Ripper) | [👉 Leer](./Windows/Blue.md) |
| **Ice** | 🟢 Fácil | 17/06/2026 | Icecast Exploit, Local Exploit Suggester, Process Migration (spoolsv.exe), Kiwi/Mimikatz | [👉 Leer](./Windows/Ice.md) |
| **Blaster** | 🟢 Fácil | 19/06/2026 | OSINT, RDP, CVE-2019-1388 (UAC Bypass), Metasploit Web Delivery, Persistence | [👉 Leer](./Windows/Blaster.md) |
| **Attacktive Directory**| 🟡 Media | 27/06/2026 | Active Directory, Kerbrute, AS-REP Roasting (Impacket), DCSync, Pass-the-Hash (Evil-WinRM)| [👉 Leer](./Windows/AttacktiveDirectory.md)|
| **Relevant** | 🟡 Media | 27/06/2026 | SMB-to-Web Mapping, IIS Misconfig, ASPX Payload, Token Impersonation (PrintSpoofer) | [👉 Leer](./Windows/Relevant.md) |

---

## 🛠️ Metodología y Stack Técnico

La resolución de las máquinas sigue un ciclo de auditoría profesional estructurado, combinando explotación manual y automatizada:

* 🔍 **Reconocimiento & OSINT:** Análisis de superficie de ataque, enumeración de infraestructura con `nmap` (scripts NSE), `gobuster`, `enum4linux` y monturas `NFS`. Detección de fallos de arquitectura (SMB-to-Web Mapping) y auditoría de cabeceras HTTP.
* 💥 **Acceso Inicial & Web:** Explotación de vulnerabilidades (SQLi, *Authenticated RCE*), generación de *payloads* nativos (`msfvenom` .aspx/.php), manipulación de protocolos (*Raw Active Mode* FTP), *Fuerza Bruta* (`Hydra`) y ejecución *fileless* vía `Metasploit`.
* 🏢 **Active Directory (AD):** Enumeración de dominio mediante `Kerbrute`, recolección de *tickets* vulnerables (*AS-REP Roasting*), extracción de base de datos NTDS (`Impacket secretsdump` / DCSync) y acceso vía WinRM (`Evil-WinRM`).
* ⬆️ **Escalada de Privilegios:** * *Linux:* Abuso de SUID/Sudo (GTFOBins, `$PATH`, *tar checkpoints*, CVE-2019-14287).
  * *Windows:* *Bypasses* de UAC (CVE-2019-1388), migración de procesos (`spoolsv.exe`) y suplantación de *tokens* locales abusando de `SeImpersonatePrivilege` (`PrintSpoofer`).
* 🔑 **Post-Explotación & Cracking:** Extracción de credenciales en memoria (`Kiwi/Mimikatz`), técnica *Pass-the-Hash* (PtH), persistencia en registro y ruptura offline (*NTLM/MD5*, *ZIP Hashes*, Base64, *Kerberos etype 23*) con `Hashcat` y `John the Ripper`.

---

*Desarrollado por Juan Merino Garrido - Estudiante de Ingeniería Informática*
