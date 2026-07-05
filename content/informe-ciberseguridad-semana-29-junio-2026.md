# 🛡️ Informe Semanal de Ciberseguridad
## Semana del 23 al 29 de junio de 2026

---

## Resumen Ejecutivo

Una semana marcada por la hipersactividad del grupo ShinyHunters, que aprovechó un zero-day crítico en Oracle PeopleSoft (CVSS 9.8) para comprometer más de 300 instancias y filtrar datos de universidades, Eastman Kodak y Madison Square Garden Sports. Paralelamente, el mayor Patch Tuesday de la historia de Microsoft (206 CVEs, 6 zero-days) y la explotación activa de la vulnerabilidad "RoguePlanet" en Windows Defender mantienen los equipos de respuesta en alerta máxima. En el plano operacional, la campaña **FortiBleed** ha expuesto credenciales de más de 86.000 firewalls Fortinet en 194 países, y la proliferación de herramientas ofensivas con IA (+340% de ataques AI-asistidos en Q1 2026) está redefiniendo el panorama de amenazas.

---

## 1. Vulnerabilidades Críticas y CVEs

### 🔴 CVE-2026-35273 — Oracle PeopleSoft RCE (CVSS 9.8) — **EXPLOTADO ACTIVAMENTE**
El grupo ShinyHunters explotó esta vulnerabilidad de ejecución remota de código en el componente **Environment Management** de PeopleSoft Enterprise PeopleTools entre el 27 de mayo y el 9 de junio. Sin autenticación previa y con acceso remoto, los atacantes comprometieron más de 300 instancias en 100+ organizaciones (68% sector educativo en EE.UU.). Oracle ha publicado parche de emergencia. **Prioridad de parcheo: inmediata.**

- Afecta: PeopleSoft Enterprise PeopleTools (múltiples versiones)
- Impacto: RCE sin autenticación → robo masivo de datos → extorsión
- Referencias: [The Hacker News](https://thehackernews.com/2026/06/shinyhunters-exploits-oracle-peoplesoft.html) · [BleepingComputer](https://www.bleepingcomputer.com/news/security/oracle-peoplesoft-servers-hacked-in-shinyhunters-data-theft-attacks/)

### 🔴 CVE-2026-47281 "RoguePlanet" — Windows Defender / VS Code (CVSS 9.6) — **ZERO-DAY ACTIVO**
Escalada de privilegios a nivel SYSTEM mediante la interacción entre Windows Defender y Visual Studio Code. Explotación confirmada en la wild. Microsoft la incluyó en el Patch Tuesday de junio. Afecta a entornos de desarrolladores con VS Code instalado.

- Impacto: Escalada de privilegios local → SYSTEM access
- Referencias: [Threat-Modeling.com](https://threat-modeling.com/windows-defender-rogueplanet-zero-day-cve-2026-47281/)

### 🔴 CVE-2026-55255 — Langflow IDOR (CVSS 9.9) — **EXPLOTACIÓN ACTIVA DESDE 25 JUNIO**
Primera explotación activa observada el 25 de junio. Se trata de un **Insecure Direct Object Reference (IDOR) cross-tenant** en la plataforma de orquestación de agentes de IA Langflow. Permite acceder a flujos y datos de otros tenants. Crítico para organizaciones que usen Langflow en entornos multi-tenant.

- Impacto: Acceso no autorizado entre tenants → filtración de lógica de IA y datos
- Referencias: [Sysdig Blog](https://www.sysdig.com/blog/understanding-langflow-cve-2026-55255-and-why-higher-cvss-vulnerabilities-arent-always-the-most-exploited)

### 🟠 Microsoft Patch Tuesday Junio 2026 — Récord histórico: 206 CVEs
El mayor Patch Tuesday desde el lanzamiento del programa en 2003. Destacan:
- **54 vulnerabilidades RCE**, con clúster de 11 CVEs en Remote Desktop Client
- **65 Elevation of Privilege** (EoP), incluyendo 3 bypasses de BitLocker (CVE-2026-45585 "YellowKey", CVE-2026-50507 "Bitskrieg")
- **8 bypasses de Secure Boot** + múltiples UEFI-level bypasses
- **CVE-2026-47291**: RCE crítico en HTTP.sys sin autenticación (integer overflow + heap overflow)
- 6 zero-days en total; 1 explotado activamente en la wild

- Referencias: [Arctic Wolf](https://arcticwolf.com/resources/blog/microsoft-patch-tuesday-security-recap-june-2026-edition/) · [Malwarebytes](https://www.malwarebytes.com/blog/bugs/2026/06/microsofts-biggest-ever-patch-tuesday-fixes-206-bugs-including-3-zero-days)

### 🟠 CVE-2026-55200 — libssh2 RCE (PoC público)
Vulnerabilidad en `ssh2_transport_read()` que permite integer wrap en `packet_length`, resultando en escritura out-of-bounds en heap. PoC público disponible. Afecta libssh2 ≤ 1.11.1. Actualizar a commit `97acf3d` o versión parcheada.

- Referencias: [CybersecurityNews](https://cybersecuritynews.com/poc-exploit-libssh2-rce-vulnerability/)

### 🟠 SAP — Múltiples CVEs críticos (Patch Tuesday SAP, junio)
- **CVE-2026-44748** (CVSS 9.9): XML Signature Wrapping en SAML en SAP NetWeaver ABAP
- **CVE-2026-27671** (CVSS 9.8): Memory corruption sin autenticación en AS ABAP — compromete la instancia completa
- **Adobe Campaign Classic**: 2 CVEs con CVSS 10 (APSB26-66)

---

## 2. Brechas y Hackeos

### 💀 ShinyHunters — Campaña de extorsión masiva
La semana ha estado dominada por ShinyHunters, que ha pasado de comprometer sistemas a escalar directamente hacia extorsión pública:

- **Oracle PeopleSoft** (≥100 organizaciones, 300+ instancias): datos de cientos de miles de estudiantes universitarios. CVE-2026-35273 como vector.
- **Eastman Kodak**: acceso confirmado a datos corporativos; amenaza de publicar 2,2 millones de registros.
- **Madison Square Garden Sports**: 45 GB de datos publicados después de negarse a pagar el rescate.

### 💀 Klue → LastPass / BeyondTrust — Ataque de cadena de suministro
El grupo **Icarus** comprometió **Klue** (plataforma de inteligencia de mercado) usando credenciales legacy, robó los **tokens OAuth** que Klue mantenía en nombre de sus clientes empresariales, y accedió a instancias de Salesforce de más de una docena de organizaciones. LastPass y BeyondTrust confirmaron el impacto:

- **LastPass**: nombres, teléfonos, emails, direcciones físicas y contenido de tickets de soporte comprometidos. Vaults y master passwords **no afectados**.
- **BeyondTrust**: datos CRM expuestos.
- Vector: credenciales legacy de proveedor → OAuth token theft → lateral movement a Salesforce de clientes.

- Referencias: [SecurityWeek](https://www.securityweek.com/beyondtrust-lastpass-impacted-by-klue-salesforce-incident/) · [LastPass Blog](https://blog.lastpass.com/posts/klue-supply-chain-incident-and-lastpass-response)

### 🔥 FortiBleed — 86.644 firewalls Fortinet expuestos (194 países)
Campaña activa de credential harvesting contra dispositivos **FortiGate** expuestos a Internet. No es una vulnerabilidad nueva: los atacantes explotan el hashing de credenciales débil heredado de versiones pre-2025 de FortiOS, extraen ficheros de configuración y crackean los hashes. CISA emitió alerta el 18 de junio urgiendo hardening inmediato.

- Acción recomendada: rotar todas las credenciales de FortiGate, actualizar FortiOS a versión reciente, revisar configuraciones expuestas.
- Referencias: [Arctic Wolf](https://arcticwolf.com/resources/blog/active-fortibleed-campaign-impacting-fortinet-devices-across-194-countries/) · [CISA](https://www.cisa.gov/news-events/alerts/2026/06/18/cisa-urges-hardening-fortinet-devices-after-reports-credential-exposure)

---

## 3. Ransomware y Malware

### ⚠️ Operación LE — Takedown de Amadey y StealC (24 junio 2026)
El 24 de junio, operación coordinada de fuerzas del orden desmanteló la infraestructura criminal detrás de los malware **Amadey** (loader) y **StealC** (infostealer), ampliamente utilizados como precursores de despliegues de ransomware. El impacto real sobre los grupos que los empleaban está por determinarse, pero representa un golpe logístico a múltiples actores.

### ⚠️ Semana activa en el ecosistema ransomware
Incidentes documentados en ransomware.live para la semana:
- **Vienna Airport (Flughafen Wien AG)**: ataque detectado el 23 de junio.
- **MagMutual Insurance Company**: comprometida por el grupo **LeakNet**.
- **Nachlass Nord**: comprometida por **ANUBIS**.
- **IH Engineers**: ataque revelado el 23 de junio (ocurrido el 29 de mayo).

### 📌 Estado actual de grupos de ransomware
- **LockBit 5.0**: activo y evolucionado. Alianza consolidada con **Qilin** y **DragonForce** desde octubre 2025. Soporte para Windows, Linux y ESXi. Objetivo primario: sector empresarial estadounidense.
- **ALPHV/BlackCat**: desarticulado. Sus afiliados han migrado principalmente a LockBit y RansomHub.
- **Campaña CyberStrikeAI**: herramienta ofensiva con IA que comprometió 600+ firewalls FortiGate en 55 países mediante credential harvesting y reconocimiento de red totalmente automatizados.

---

## 4. APTs y Amenazas Estatales

### 🇨🇳 China — Salt Typhoon sigue expandiendo alcance
Salt Typhoon ha comprometido redes en más de **80 países**, con foco en telecomunicaciones, transporte y gobierno. En febrero 2026 se documentó una campaña que afectó a 50+ telecoms y agencias gubernamentales en 42 países, usando Google Sheets como canal C2 encubierto. Sigue siendo el actor con mayor alcance geográfico activo en 2026.

### 🇮🇷 Irán — APT34 en modo reactivo
Tras la **Operación Epic Fury** (ataque militar EE.UU.-Israel contra instalaciones nucleares iraníes, 2 de marzo 2026), APT34 incrementó su tempo operacional en un 72 horas. Previamente, el grupo había suspendido operaciones entre el 8 y el 27 de enero, coincidiendo con un apagón de Internet impuesto por el gobierno iraní. El patrón sugiere un grupo directamente subordinado a decisiones gubernamentales.

### 🇰🇵 Corea del Norte — Lazarus / Grupo Kimsuky
Tras robar ~2.02B USD en criptomonedas durante 2025, Lazarus mantiene campañas multi-vector activas en 2026 orientadas a high-value targets: exchanges, DeFi, y contratistas de defensa. Énfasis en social engineering y compromisos de larga duración.

---

## 5. Tendencias y Herramientas

### 🤖 IA ofensiva: explosión de herramientas y técnicas
- **+340% de ataques AI-asistidos en Q1 2026** respecto a 2025 (Boldsmedia).
- **70 herramientas ofensivas de IA open-source** catalogadas a marzo 2026, frente a menos de 5 antes de GPT-4. Cubren desde descubrimiento de vulnerabilidades hasta generación de exploits y agentes autónomos end-to-end.
- **Deepfakes en tiempo real**: tasa de éxito >95% en CEO fraud usando clonación de voz con 10-15 segundos de audio original. Incidentes documentados en videollamadas ejecutivas con transferencias autorizadas.
- **Prompt injection**: identificado como el mayor riesgo actual en entornos con LLMs desplegados, junto con agentes de IA autónomos con acceso a herramientas.
- **CyberStrikeAI**: primer ejemplo documentado de herramienta ofensiva puramente AI-driven comprometiendo 600+ dispositivos de forma autónoma.

### 🛡️ Defensivo destacado
- Takdown de Amadey/StealC reduce temporalmente capacidad de distribución de loaders para varios grupos de ransomware.
- CISA amplió el catálogo KEV con CVE-2026-12569 (PTC Windchill, CVSS 9.3) y emitió alerta sobre FortiBleed.
- White House NSM (16 junio): nuevo memorándum de seguridad nacional establece requisitos base para redes clasificadas y militares, con nuevas capacidades de intervención para el director de la NSA.

---

## 6. Mundo Corporativo y Regulatorio

### 📋 NIS2 — Deadline de octubre 2026 se acerca
- Los Países Bajos exigieron a entidades esenciales e importantes completar auto-evaluaciones antes de **junio 2026**.
- El 26 de mayo, el NIS2 Cooperation Group adoptó plantillas comunes para reporte de incidentes, reduciendo la carga administrativa.
- La Comisión Europea propuso enmiendas en enero 2026 para mayor claridad legal. El deadline final para entidades en sectores críticos: **octubre 2026**.

### 📋 DORA — Primera enforcement en marcha
DORA es aplicable desde enero 2025. Los supervisores europeos han señalado que la primera enforcement por fallos en reporting de incidentes graves y deficiencias en el Registro de Información comenzará en el **ciclo supervisorio de 2026**. Las entidades financieras que no hayan completado sus programas de resiliencia digital están en riesgo de sanciones inminentes.

### 📋 Microsoft Patch Tuesday: nuevo "record"
206 vulnerabilidades en un solo ciclo marca un punto de inflexión. Analistas de Arctic Wolf y CrowdStrike señalan que el aumento sostenido de superficie de ataque en el ecosistema Microsoft (especialmente en Remote Desktop, HTTP.sys y Hyper-V) requiere replantear los ciclos de parcheo desde mensuales a continuos.

---

## ⚡ Para Estar Atento Esta Semana

| Riesgo | Por qué puede escalar |
|--------|----------------------|
| **CVE-2026-35273 (Oracle PeopleSoft)** | ShinyHunters aún activos; organismos que no hayan parcheado siguen siendo objetivos. Esperar más víctimas en la próxima semana. |
| **FortiBleed** | Más de 86.000 dispositivos expuestos. Los hashes crackeados comenzarán a monetizarse vía acceso inicial broker o ransomware. Revisar VPN/firewall logs urgentemente. |
| **CVE-2026-55200 (libssh2 PoC público)** | PoC disponible → weaponization inminente en servicios SSH en Internet. Alta superficie de ataque en sistemas Linux y embebidos. |
| **Cadena de suministro OAuth (Klue/Icarus)** | El patrón "comprometer SaaS menor → robar tokens OAuth → pivotar a servicios enterprise" replicable. Auditar integraciones OAuth de terceros. |
| **Langflow CVE-2026-55255** | Explotación activa + comunidad AI que despliega instancias multi-tenant sin hardening. Esperar aumento de incidentes en entornos de IA empresarial. |
| **LockBit 5.0 + alianza Qilin/DragonForce** | Infraestructura consolidada post-BlackCat. El ecosistema de afiliados está estabilizado: volumen de ataques en ascenso. Foco en ESXi y Linux. |

---

## Fuentes

- [The Hacker News — ShinyHunters Oracle PeopleSoft](https://thehackernews.com/2026/06/shinyhunters-exploits-oracle-peoplesoft.html)
- [BleepingComputer — Oracle PeopleSoft breach](https://www.bleepingcomputer.com/news/security/oracle-peoplesoft-servers-hacked-in-shinyhunters-data-theft-attacks/)
- [SecurityWeek — BeyondTrust/LastPass Klue incident](https://www.securityweek.com/beyondtrust-lastpass-impacted-by-klue-salesforce-incident/)
- [LastPass Blog — Klue supply chain response](https://blog.lastpass.com/posts/klue-supply-chain-incident-and-lastpass-response)
- [Arctic Wolf — FortiBleed campaign](https://arcticwolf.com/resources/blog/active-fortibleed-campaign-impacting-fortinet-devices-across-194-countries/)
- [CISA — FortiBleed alert](https://www.cisa.gov/news-events/alerts/2026/06/18/cisa-urges-hardening-fortinet-devices-after-reports-credential-exposure)
- [Malwarebytes — Microsoft Patch Tuesday June 2026](https://www.malwarebytes.com/blog/bugs/2026/06/microsofts-biggest-ever-patch-tuesday-fixes-206-bugs-including-3-zero-days)
- [Arctic Wolf — Patch Tuesday recap](https://arcticwolf.com/resources/blog/microsoft-patch-tuesday-security-recap-june-2026-edition/)
- [Threat-Modeling.com — CVE-2026-47281 RoguePlanet](https://threat-modeling.com/windows-defender-rogueplanet-zero-day-cve-2026-47281/)
- [Sysdig — CVE-2026-55255 Langflow](https://www.sysdig.com/blog/understanding-langflow-cve-2026-55255-and-why-higher-cvss-vulnerabilities-arent-always-the-most-exploited)
- [CybersecurityNews — libssh2 PoC](https://cybersecuritynews.com/poc-exploit-libssh2-rce-vulnerability/)
- [Hadrian — 70 AI offensive tools](https://hadrian.io/blog/the-ai-offensive-security-boom-seventy-tools-in-eighteen-months)
- [ComplianceHub — DORA/NIS2 enforcement 2026](https://compliancehub.wiki/dora-nis2-2026-enforcement-eu-financial-cyber-resilience-compliance/)
- [eSecurity Planet — Weekly roundup June 2026](https://www.esecurityplanet.com/weekly-roundup/massive-breaches-ai-risks-and-critical-vulnerabilities-define-this-week-in-cybersecurity-in-june-2026/)
- [Ransomware.live — Incident tracker](https://www.ransomware.live/)
- [CybelAngel — Chinese APTs 2026](https://cybelangel.com/blog/cyber-espionage-apts/)
- [Trellix — Iranian cyber capability 2026](https://www.trellix.com/blogs/research/the-iranian-cyber-capability-2026/)

---

*Informe generado automáticamente el 29 de junio de 2026 · Pentra Red Team Intelligence*
