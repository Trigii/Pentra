# Informe semanal de ciberseguridad — Semana del 12 al 19 de octubre de 2026

*Orientado a red team / pentesting. Generado el 19/10/2026. Los datos proceden de agregadores y blogs de terceros; las cifras discrepantes entre fuentes se indican donde aplica.*

## Resumen ejecutivo

La semana la dominan los equipos de acceso perimetral y gestión remota: el zero-day de BeyondTrust Remote Support (CVE-2026-102764, CVSS 9.3), explotado por el actor chino ARMORY CASTLE (UNC5221), junto con Ivanti Connect Secure, Cisco Secure Email Gateway, PAN-OS y Fortinet, todos añadidos al KEV de CISA. El Patch Tuesday de octubre fue récord (~1.4k CVE, con un zero-day de bypass de MotW). En incidentes, destacan la cadena de suministro MSP/SaaS (ConnectWise Cloud Backup, 173 tenants M365 con inicios de sesión no autorizados), el ataque OT a refrigeración de supermercados franceses y la brecha de Ahold Delhaize. Continúa la presión sobre infraestructura de borde con persistencia en memoria y manipulación de logs, lo que obliga a hacer forensia y no solo parchear.

## 1. Vulnerabilidades críticas y CVEs

- **BeyondTrust Remote Support — CVE-2026-102764 (CVSS v4 9.3, KEV 16/10).** Validación de origen incorrecta (CWE-290) que permite operaciones de archivo sin autenticar y root en el appliance. Explotado como 0-day desde el 29/08 por ARMORY CASTLE (UNC5221/Warp Panda): webshell PHP *Sawfly*, túnel SOCKS5 *Bareclaw*, borrado de logs, exfiltración vía el servicio nativo de transferencia de archivos. Afecta hasta 26.2.3; parche BT26-05-RS (≥24.3.1; versiones anteriores, actualizar primero). SaaS parcheado el 09/10. Sin PoC público conocido.
- **Ivanti Connect Secure — CVE-2026-104935 (KEV 13/10).** Bypass de autenticación explotado; permite comandos arbitrarios como root. Volexity (UAT0576) lo vio encadenado con otras dos vulnerabilidades desde el 03/10 contra appliances totalmente parcheados, sin malware y con alteración de logs. CISA recomienda forensia de memoria y red.
- **Cisco Secure Email Gateway — CVE-2026-76519 (CVSS 9.8, KEV 14/10).** Path traversal en AMP File Reputation; escritura de archivos y ejecución como root sin autenticar vía adjuntos manipulados, solo si File Reputation está activo en una política de correo. Segundo 0-day de SEG en 30 días. Relacionado: Cisco ISE CVE-2026-76460 (CVSS 10.0, KEV desde 16/09), bypass de autenticación en API con root.
- **Microsoft Patch Tuesday (13/14-oct, ~1.41k CVE según fuente).** CVE-2026-96951: bypass de Mark of the Web en Windows Shell (explotado, KEV): relevante para cadenas de phishing con SmartScreen/Protected View. CVE-2026-70242: RCE en Outlook vía Reading Pane (9.8). CVE-2026-105016: heap overflow en QUIC (9.8, potencial RCE de red sin auth). CVE-2026-108398: bypass de WDAC en .NET (8.8, divulgado públicamente).
- Otros: PAN-OS CVE-2026-0322 (CVSS 8.6, inyección de código en interfaz de gestión, explotación "muy limitada"); SonicWall SMA 1000 CVE-2026-102255 (SSRF pre-auth, CVSS 10 según Ivanti, hotfix 06/10, tráfico de ataque detectado); Ahsay AhsayCBS CVE-2026-105133/105134 (auth y command injection, KEV 14/10); Fortinet CVE-2025-61624 (KEV 16/10).

## 2. Brechas y hackeos

- **ConnectWise Cloud Backup (ex-SkyKick).** Una credencial de aplicación de Microsoft 365 comprometida dio acceso a entornos de clientes; Huntress observó inicios de sesión no autorizados en 173 tenants. ConnectWise deshabilitó siete identidades de aplicación (17/10) y suspendió el servicio (18/10). Revisar consentimientos OAuth y service principals de terceros.
- **Ahold Delhaize.** Sistemas desconectados como precaución en Stop & Shop, Food Lion, Hannaford, Albert Heijn, Delhaize, etc. Tiendas abiertas pero web, apps y logística afectadas. Sin atribución a ransomware por ahora (sufrió uno en 2024).
- **Burger King Japón:** ~11 M de registros de miembros (acceso 2–7/10, descubierto 13/10). **iRhythm:** ≥360k afectados tras ingeniería social. **Concesionarios VW (Alemania):** datos de pedidos/financiación expuestos vía proveedor IT y usados para fraude de cambio de cuenta bancaria.
- **Ataque OT a refrigeración de supermercados franceses:** cambio remoto de consignas hasta ~50 °C con pérdidas y fuegos; la fiscalía de París y ANSSI investigan.

## 3. Ransomware y malware

- **Majin (nueva familia, Windows):** extensión `.majin`, doble extorsión con plazo de 7 días, borrado de shadow copies y WMI para reconocimiento/persistencia. Víctimas en múltiples países (Colombia, India, Francia, Italia, EE. UU., Alemania…).
- **Qilin (RaaS):** Windows/Linux/ESXi; nueva víctima en Japón (construcción/logística). **The Gentlemen** y **Krybit** (~203 GB publicados de una metalúrgica india) siguen activos.
- **AOMEI Backupper:** infraestructura de actualización manipulada en ventanas de 5–10 min entre mayo y julio; los endpoints que actualizaron pueden estar backdoorizados (GTIG/Mandiant).
- **MALFEX (npm):** paquete `imgdecode` (979 descargas) carga una DLL que obtiene shellcode cifrado desde GitHub e inyecta en `dllhost.exe`. **RuRAT:** abuso de Remote Utilities con persistencia por tareas programadas/servicios.

## 4. APTs y amenazas estatales

- **ARMORY CASTLE (China, UNC5221):** zero-day de BeyondTrust; objetivo SaaS, tecnología, gobierno, legal y BPOs. Y **CL-STA-0954** (Unit 42): intrusión de 25 meses en un MSP de telecom vía FortiManager (CVE-2024-47575) y CVE-2025-61624, implante ARM64 en memoria *PEERSTITCH* (se elimina al reiniciar) y robo de configuraciones/credenciales de >500 FortiGate.
- **Flax Typhoon (China):** el DOJ/FBI incautó dominios de *Microscan* (escáner vía botnet tipo Mirai) y *FishHub* (plataforma de spear-phishing, ~20 universidades taiwanesas).
- **UAT-11985 (Cisco Talos):** AiTM y QR phishing contra investigadores vinculados a Taiwán para robar credenciales Google y MFA; Talos estima uso de IA en los señuelos. **Irán (Screening Serpens)** y **trabajadores TI norcoreanos** (alerta FBI 13/10) siguen en aumento.
- **ShinyHunters (UNC6240):** vishing a help desk, registro de MFA no autorizado, robo de datos SaaS; explotación de Oracle PeopleSoft CVE-2026-35273. Arrestos en septiembre y uno anunciado por el FBI el 9/10; cifras (>140 organizaciones, ≥70 M$) no verificadas.

## 5. Tendencias y herramientas (TTPs)

- Los 0-day en appliances de acceso remoto se encadenan con **antiforense** (borrado de logs, implantes solo en memoria): parchear no basta, hay que buscar compromiso previo.
- **Cadena de suministro y confianza en terceros:** apps M365 de proveedores, actualizaciones de software, npm.
- **Ingeniería social con identidad:** help desk, MFA rogue, AiTM y QR phishing, con contenido asistido por IA.
- MotW bypass sigue siendo un vector recurrente (séptimo 0-day explotado desde 2022) para entrega inicial.

## 6. Mundo corporativo y regulación

- **ConnectWise** suspende Cloud Backup (impacto reputacional para proveedores MSP).
- **UE:** octubre de 2026 marca un hito de cumplimiento de NIS2 y refuerzo de la aplicación de DORA, con énfasis en responsabilidad de la dirección y riesgo de terceros (fuentes secundarias de cumplimiento; verificar con la transposición nacional).
- EE. UU.: acciones judiciales contra esquemas de trabajadores TI norcoreanos e infraestructura de China. No se encontraron adquisiciones relevantes confirmadas esta semana.

## Para estar atento esta semana

1. **BeyondTrust/Ivanti/Cisco SEG:** posible publicación de PoC y escaneo masivo; en tests, comprobar exposición y versiones. Probable más victimología de ARMORY CASTLE.
2. **ConnectWise Cloud Backup:** pueden aparecer más tenants afectados y detalles del abuso de credenciales de aplicación.
3. **Ahold Delhaize y refrigeración francesa:** posible atribución (ransomware u OT/hacktivismo).
4. **SonicWall SMA 1000 (CVE-2026-102255, CVSS 10):** confirmación de compromisos.
5. **CVE-2026-70242 (Outlook) y QUIC:** vigilar PoC tras el Patch Tuesday.
6. **ShinyHunters:** posibles represalias o nuevas filtraciones tras los arrestos.

## Fuentes

- [Cybersecurity Week in Review: Oct 12–18, 2026 (Senthorus)](https://blog.senthorus.ch/posts/weekly_reviews/19_10_2026)
- [Cyber Roundup: week of Oct 12 (CybelAngel)](https://cybelangel.com/blog/cyber-roundup-week-of-october-12/)
- [Actively exploited this week: 9 new CISA KEV entries (DEV Community)](https://dev.to/devaland/actively-exploited-this-week-9-new-cisa-kev-entries-12-october-to-18-october-2026-3dno)
- [Chinese APT Exploited BeyondTrust Vulnerability as Zero-Day (SecurityWeek)](https://www.securityweek.com/chinese-apt-exploited-beyondtrust-vulnerability-as-zero-day/)
- [CISA: patching critical BeyondTrust vulnerability](https://www.cisa.gov/news-events/alerts/2026/10/16/cisa-urges-patching-critical-vulnerability-beyondtrust-remote-support)
- [Cisco ISE CVE-2026-76460 (Safeguard)](https://safeguard.sh/resources/blog/cisco-ise-cve-2026-76460-authentication-bypass-kev)
- [CYFIRMA Weekly Intelligence Report, 16 Oct 2026](https://www.cyfirma.com/news/weekly-intelligence-report-16-oct-2026/)
- [Unit 42: Screening Serpens](https://unit42.paloaltonetworks.com/tracking-iran-apt-screening-serpens/)
- [DORA/NIS2 October 2026 (ComplianceHub)](https://compliancehub.wiki/dora-nis2-2026-enforcement-eu-financial-cyber-resilience-compliance/)
