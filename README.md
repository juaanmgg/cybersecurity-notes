# 🛡️ TryHackMe Writeups & Cybersecurity Notes

<div align="center">

![Cybersecurity](https://img.shields.io/badge/Cybersecurity-eJPTv2_Prep-000000?style=for-the-badge&logo=tryhackme&logoColor=white)
![Status](https://img.shields.io/badge/Status-Active-success?style=for-the-badge)
![Writeups](https://img.shields.io/badge/Writeups-3-blue?style=for-the-badge)

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

---

## 🛠️ Herramientas y Metodología

Para la resolución de estas máquinas, aplico un enfoque estructurado utilizando el siguiente arsenal técnico:

* **Reconocimiento & Enumeración:** `nmap` (TCP/UDP, Scripts NSE), `gobuster`, `enum4linux`.
* **Explotación:** Búsqueda manual de exploits (`searchsploit`), manipulación de peticiones, `Metasploit Framework`.
* **Post-Explotación & PrivEsc:** Enumeración manual de SUID/Sudo, migración de procesos, volcado de hashes (`hashdump`), estabilización de TTY.
* **Cracking:** `Hashcat`, `John the Ripper`.

---
*Desarrollado por Juan Merino Garrido - Estudiante de Ingeniería Informática*
