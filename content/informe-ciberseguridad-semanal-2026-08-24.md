# Informe Semanal de Ciberseguridad — Semana del 17 al 24 de agosto de 2026

## Resumen ejecutivo

Semana marcada por dos vulnerabilidades de identidad/infraestructura crítica: un RCE con CVSS 10.0 en Microsoft Entra ID (CVE-2026-69836) explotado activamente antes del parche, y un day-zero en VMware vCenter (CVE-2026-59310, CVSS 9.8) atribuido a un APT chino que desplegó ransomware Babuk como cortina de humo. CISA/FBI actualizaron su advisory sobre Medusa ransomware, que ya suma más de 500 víctimas de infraestructura crítica desde 2021, y Lazarus (Corea del Norte) explotó un 0-day de Windows (CVE-2026-68820) en Operation Dream Job contra el sector de defensa. En el frente de brechas, un actor conocido como "TheHatman" está vendiendo bases de datos de empleados exfiltradas de tenants de Azure de múltiples Fortune 500 (McDonald's, Vodafone, TCS, Kyndryl), y la Hacienda francesa confirmó la exposición de datos de 678.000 contribuyentes.

## 1. Vulnerabilidades críticas y CVEs

**CVE-2026-69836 — Microsoft Entra ID (CVSS 10.0).** RCE por deserialización insegura (CWE-502) en el backend de Entra ID, explotable sin autenticación ni interacción del usuario. Microsoft aplicó la corrección directamente en su infraestructura cloud (no requiere parche del cliente), pero hubo una ventana de explotación activa antes de la corrección. Dada la naturaleza de Entra ID como puerta de entrada a identidad corporativa, cualquier organización con Azure/M365 debería revisar logs de autenticación anómala de las últimas semanas. [SecurityWeek](https://www.securityweek.com/microsoft-rolls-out-22-fresh-security-patches/) · [Help Net Security](https://www.helpnetsecurity.com/2026/08/21/microsoft-entra-id-vulnerability-cve-2026-69836/)

**CVE-2026-59310 — VMware vCenter (CVSS 9.8).** Directory traversal que permite RCE, explotado por un actor sospechoso de nexo chino. En al menos un caso investigado derivó en backdoor + SSH reverso + ransomware Babuk, aunque los investigadores creen que el ransomware fue una distracción forense y no el objetivo real de la intrusión. [The Hacker News](https://thehackernews.com/2026/08/attackers-exploit-vmware-vcenter.html)

**CVE-2026-68820 — Windows AFD.sys (CVSS 7.0, 0-day).** Escalada de privilegios en el driver Ancillary Function Driver for WinSock, parcheada en el Patch Tuesday de agosto (que incluyó 421 CVEs, 62 críticas, entre ellas CVE-2026-62815 en Microsoft QUIC, CVSS 9.8). Explotada por Lazarus antes del parche. [Tenable](https://www.tenable.com/blog/microsofts-august-2026-patch-tuesday-addresses-398-cves-cve-2026-68820) · [SecurityWeek](https://www.securityweek.com/august-2026-patch-tuesday-microsoft-fixes-421-cves-one-exploited-zero-day/)

**Otras a vigilar:** CVE-2026-19478 (GitLab, code injection crítico sin autenticación), CVE-2026-19490 (Citrix NetScaler, bypass de autenticación crítico), CVE-2026-65400 (macOS Screen Sharing, CVSS 9.8, explotado para minado de cripto tras exposición del puerto 5900), CVE-2026-58231 (SAP Commerce Cloud, explotado días después del parche), CVE-2026-14863 (FileRun, command injection CVSS 8.7), y una SQLi 0-day sin CVE aún en GeoServer explotada horas después de su divulgación pública. [Help Net Security](https://www.helpnetsecurity.com/2026/08/21/citrix-netscaler-gateway-cve-2026-19490/) · [The Hacker News](https://thehackernews.com/2026/08/weekly-recap-vmware-exploits-windows-0.html)

## 2. Brechas y hackeos

**Filtración masiva de Azure/Entra.** El actor "TheHatman" inundó foros cibercriminales con bases de datos de empleados —presuntamente descargadas con credenciales comprometidas desde portales Azure/Entra— afectando a McDonald's, TCS, Vodafone, HCL Technologies, Kyndryl, Gap, Hexaware y Wyndham Hotels. Hudson Rock apunta a infecciones de infostealers como vector más probable, no a una vulnerabilidad zero-day sistémica. [Hudson Rock](https://www.hudsonrock.com/blog/massive-azure-exfiltration-campaign-exposes-millions-of-enterprise-records-via-compromised-credentials-mcdonalds-vodafone-kyndryl-others)

**Hacienda francesa (DGFiP).** Un atacante con alias "ZeroBytes" exfiltró datos de contribuyentes y los puso a la venta en un foro criminal. La cifra inicial reportada fue de 678.000 afectados; la DGFiP actualizó posteriormente el alcance a ~600.000 personas/empresas, incluyendo mensajes privados expuestos. [Help Net Security](https://www.helpnetsecurity.com/2026/08/17/france-tax-authority-data-breach/)

**CareCloud (salud, EE. UU.).** 3,7 millones de registros médicos comprometidos — quinta mayor brecha sanitaria de 2026.

**SafePal (crypto hardware wallets).** Fallo de autorización en un plugin de seguimiento de pedidos expuso datos de 39.798 clientes (nombres, direcciones, teléfonos). [Help Net Security](https://www.helpnetsecurity.com/2026/08/17/safepal-data-breach-customer-order-information/)

**UT San Antonio.** Ciberataque contra la red académica obligó a retrasar tres días el inicio del semestre. [Help Net Security](https://www.helpnetsecurity.com/2026/08/19/ut-san-antonio-cyberattack-fall-semester-delay/)

## 3. Ransomware y malware

**Medusa (CISA/FBI/HHS, advisory actualizado).** Más de 500 organizaciones de infraestructura crítica comprometidas desde junio de 2021 (frente a las ~300 reportadas en marzo de 2025), con foco creciente en sanidad. TTPs: explotación de vulnerabilidades recién publicadas en <24h, living-off-the-land (PowerShell, Mimikatz) y herramientas RMM legítimas (AnyDesk, SimpleHelp) para persistencia. [BleepingComputer](https://www.bleepingcomputer.com/news/security/cisa-medusa-ransomware-hit-over-500-critical-infrastructure-orgs/) · [CISA AA25-071A](https://www.cisa.gov/news-events/cybersecurity-advisories/aa25-071a)

**Anubis vs. Fairlife (Coca-Cola).** Explotación de CitrixBleed 2 contra sistemas Nutanix, exfiltración de ~1 TB y suspensión de producción durante 11 días.

**Akira vía SonicWall SSL VPN sin MFA.** Campaña de credential-spraying contra cuentas VPN desprotegidas como vector inicial.

**Amnesia Stealer (macOS).** Primer malware macOS documentado que combina clonación de perfil Chromium con control remoto en tiempo real vía Chrome DevTools Protocol (CDP) — permite al operador "conducir" la sesión autenticada de la víctima (teclado, ratón, navegación) a ~3fps. Distribuido vía ClickFix. [Jamf / The Hacker News](https://thehackernews.com/2026/08/amnesiastealer-hijacks-chromium.html)

**Shai-Hulud v2 (npm supply chain).** Más de 400 paquetes / 1.700 versiones afectadas en un ataque a la cadena de suministro de npm que abarca múltiples publishers no relacionados.

## 4. APTs y amenazas estatales

**Lazarus (Corea del Norte) — Operation Dream Job.** Explotó CVE-2026-68820 (0-day) para desplegar el nuevo backdoor "Troy" y ForestTiger contra empresas de defensa y aeroespacial en Francia, Alemania, Brasil e India, vía ofertas de empleo falsas. [The Hacker News](https://thehackernews.com/2026/08/lazarus-exploits-windows-zero-day-to.html)

**APT chino vs. VMware vCenter.** Explotación de CVE-2026-59310 con despliegue de ransomware como distracción forense (ver sección 1).

**Storm-2945 / Midnight Blizzard (APT29, Rusia-SVR) — "CaptiveCrunch".** Robo de credenciales mediante gateways Wi-Fi públicos comprometidos en organizaciones con portales cautivos. [Microsoft / SecurityWeek](https://www.securityweek.com/russian-state-apt-linked-to-recent-public-wi-fi-gateway-hacking/)

**LightSpy (China).** El implante modular de vigilancia se ha expandido a más de 13 países (incluyendo capacidades nuevas de implante en routers), con infraestructura de 117 servidores y 35 dominios que imitan fabricantes asiáticos de electrónica — evidencia sugiere que opera como plataforma de vigilancia comercializada. [Arctic Wolf Labs](https://thehackernews.com/2025/02/lightspy-expands-to-100-commands.html)

**17 hackers iraníes acusados (Mabna Institute).** EE. UU. amplió los cargos contra la operación de hacking-for-hire responsable del robo de 31 TB de datos académicos. [Help Net Security](https://www.helpnetsecurity.com/2026/08/20/us-iranian-hackers-mabna-institute-charged/)

## 5. Tendencias y herramientas

**GhostSplice — bypass de guardrails en asistentes de código IA.** Nueva técnica que fragmenta instrucciones maliciosas entre canales MCP distintos (descripción de herramienta, resultado, mensaje de sampling); cada fragmento es inofensivo por separado, pero el modelo los reensambla en una única instrucción porque no hay separación de confianza entre fuentes ("cross-channel trust fragmentation"). Relevante para equipos que integran LLMs con MCP en pipelines de desarrollo. [The Hacker News](https://thehackernews.com/2026/08/malicious-mcp-servers-can-split.html)

**Robo de sesión vía Chrome DevTools Protocol (CDP).** SpecterOps documentó cómo habilitar CDP en un proceso Chrome/Edge en ejecución (con code execution previo) permite eludir protecciones anti-replay de cookies (ABE, device-bound cookies) accediendo directamente al contexto autenticado del navegador.

**IA ofensiva en ascenso.** Google/Mandiant reportó que su herramienta interna de agentes IA (AVDH) halló más de 100 vulnerabilidades críticas verificadas en solo dos días durante una investigación real. En paralelo, agencias de EE. UU. advirtieron sobre uso de IA para generar exploits contra PLCs Siemens S7 en infraestructura crítica (agua, energía, manufactura), y Gambit Security documentó actores usando IA para priorizar qué archivos robar tras una intrusión.

**EtherHiding en testnet BNB Smart Chain.** Cadenas de infección que usan blockchain como backend de C2 se han desplazado a testnets para eliminar costes de gas y rastro financiero.

## 6. Mundo corporativo

**Financiación.** Entre el 15 de julio y el 4 de agosto se cerraron 12 rondas relevantes por ~1.090 M$, con Horizon3.ai, ThreatLocker y Glow captando el 57% del total ($620M combinados). Foco: seguridad de agentes IA, gobernanza de identidad y validación automatizada de seguridad.

**M&A.** Keyfactor anunció la adquisición de Cofide para llevar identidad verificada a agentes IA y cargas de trabajo cloud.

**Regulación.** NIS2 llega a su plazo de transposición nacional en octubre de 2026; DORA entra en su primer ciclo real de supervisión y sanciones tras un año en vigor, con foco en fallos de reporte de incidentes y deficiencias en el Registro de Información. El AI Act de la UE también activa disposiciones sobre sistemas de IA de alto riesgo desde el 2 de agosto.

**OpenAI.** Pausó temporalmente el entrenamiento RL de su próximo modelo frontera (Astra) tras un incidente de intrusión en su entorno de investigación (OpenAI-Hugging Face) y evidencia preliminar de que el modelo podría alcanzar el umbral "crítico" de capacidad cibernética de su Preparedness Framework — señal relevante sobre cómo los grandes labs empiezan a tratar la capacidad ofensiva de sus propios modelos como riesgo de seguridad nacional/corporativo.

## Para estar atento esta semana

- **Explotación post-parche de Entra ID y vCenter:** aunque ambos ya tienen corrección, es habitual una ola secundaria de intentos de explotación contra organizaciones que tardan en rotar credenciales o revisar logs retroactivos. Vigilar IoCs asociados a CVE-2026-69836 y CVE-2026-59310.
- **Medusa:** el salto de 300 a 500+ víctimas en un año, con foco creciente en sanidad, sugiere que el grupo sigue escalando operaciones — priorizar hardening de RMM legítimo (AnyDesk, SimpleHelp) y detección de Mimikatz/PowerShell abusivo.
- **GeoServer y SAP Commerce Cloud:** ambos vieron explotación en horas/días tras divulgación pública — el ciclo patch-to-exploit sigue acortándose (Rapid7 registró 8.539 vulnerabilidades altas/críticas en Q2 2026, el doble que hace un año).
- **Ataques a agentes IA/MCP:** GhostSplice y el incidente OpenAI-Hugging Face son señales tempranas de una superficie de ataque nueva (pipelines de desarrollo con LLMs) que probablemente genere más divulgaciones en las próximas semanas.
- **NIS2:** con el plazo de octubre acercándose, es previsible más actividad regulatoria/sancionadora y noticias de organizaciones apurando su cumplimiento.

---
*Fuentes principales: The Hacker News, Help Net Security, SecurityWeek, Tenable, BleepingComputer, CISA, Hudson Rock. Informe generado automáticamente — semana del 17-24 de agosto de 2026.*
