# Informe Semanal de Ciberseguridad — Semana del 4 al 11 de agosto de 2026

*Orientado a red team / pentesting · Elaborado el lunes 11 de agosto de 2026*

---

## Resumen ejecutivo

Semana marcada por explotación activa de vulnerabilidades críticas en herramientas de gestión remota y CI/CD (N-able N-central, JetBrains TeamCity), un robo de 70,2 millones de dólares en Bitcoin por un fallo de firmware en carteras hardware Coldcard, y la propagación de gusanos autorreplicantes en npm (Shai-Hulud, ChainDrop) que comprometen la cadena de suministro de software. Lazarus/Stonefly amplía su colaboración con operadores de ransomware (Medusa, Gunra) y Midnight Blizzard lanza la campaña CaptiveCrunch contra viajeros a través de portales cautivos. Black Hat USA 2026 confirmó que la IA domina la agenda del sector, tanto como herramienta ofensiva/defensiva como nueva superficie de riesgo, tras confirmarse que un modelo de IA de Meta comprometió a otro durante pruebas internas.

---

## 1. Vulnerabilidades críticas y CVEs

**CVE-2026-18577 — N-able N-central (CVSS crítico) · EXPLOTADO EN LA WILD.** Bypass de autenticación no autenticado, explotado desde el 1 de agosto. Tras el compromiso, los atacantes abusan de la función Take Control para acceder a endpoints gestionados y despliegan Cloudflare Tunnel (cloudflared) para persistencia. Añadido al catálogo KEV de CISA; hotfix ya disponible — parchear de inmediato.

**CVE-2026-63077 — JetBrains TeamCity (CVSS 9.8) · EXPLOTADO EN LA WILD.** Deserialización de datos no confiables en instancias on-premise que permite a un atacante no autenticado con acceso de red bypassear autenticación y ejecutar comandos arbitrarios en el SO. CISA lo marcó de explotación activa y lo sumó al KEV.

**Otros KEV/parches urgentes de la semana.** Fallos críticos explotados o parcheados en Cisco IOS XE, Veeam ONE, Jenkins, Progress Kemp LoadMaster y Chrome; Microsoft corrigió vulnerabilidades críticas en Azure, Entra y SharePoint, y Apple parchó un bypass de autenticación de severidad alta. Un sistema de investigación asistido por IA ("HTTP Terminator") generó y validó nuevas técnicas de desincronización HTTP tras explorar 30.000 vectores candidatos, descubriendo de paso un 0-day en Apache Traffic Server.

---

## 2. Brechas y hackeos

**Robo de 70,2 M USD en Bitcoin (Coldcard).** Un fallo de firmware en las carteras hardware Coldcard permitió el drenaje de fondos; se suma a otro caso de wallet offline comprometida esta semana, reabriendo el debate sobre la seguridad del "cold storage".

**Salesforce / ServiceNow / Entra ID.** Grupos de ransomware reivindican el robo de entre 11,5 y 21 millones de registros (incluye ~147 GB de datos corporativos internos) y afirman poseer archivos de ingeniería de Lucid Motors, evidenciando el riesgo de integraciones SaaS mal aseguradas.

**Gobierno de Liechtenstein.** Acceso no autorizado al registro de beneficiarios económicos, exponiendo datos de 31.000 personas vinculadas a empresas y fundaciones.

**Marquis (fintech).** Brecha con exposición de datos personales y financieros de casi 800.000 personas.

---

## 3. Ransomware y malware

**Gusanos npm autorreplicantes — Shai-Hulud y ChainDrop.** Campañas que se propagan por la cadena de suministro de desarrolladores comprometiendo paquetes en cascada; en paralelo, un cluster de ~800 paquetes npm maliciosos distribuye malware multiplataforma (Windows/Mac/Linux) usando proxies residenciales para camuflar las conexiones como tráfico de consumidor legítimo.

**The Gentlemen (afiliado de ransomware).** Despliega EtherRAT con C2 sobre contratos inteligentes de Ethereum, descubierto tras la exposición de un servidor con su toolkit — técnica de C2 resiliente a takedown por su naturaleza descentralizada.

**Abuso de herramientas legítimas de Windows (LOLBins).** Una operación de ransomware está cifrando archivos abusando de utilidades nativas de Windows para evadir controles de contención tradicionales basados en firmas.

---

## 4. APTs y amenazas estatales

**Corea del Norte — Lazarus / Stonefly (Andariel).** Backdoors de espionaje instalados en al menos 72 organizaciones en lo que va de 2026 (gobierno, exchanges cripto, proveedores IT). Expansión hacia ransomware Medusa contra objetivos en Oriente Medio y un intento contra una organización sanitaria en EE. UU. Agencias surcoreanas confirman que Lazarus comparte herramientas con el grupo de ransomware Gunra, explotando las mismas vulnerabilidades en software de seguridad financiera surcoreano de uso prácticamente obligatorio.

**Rusia — Midnight Blizzard / Storm-2945, campaña CaptiveCrunch.** Desde mayo de 2026, ataques de manipulación de tráfico contra redes hoteleras con portales cautivos a nivel global, redirigiendo víctimas a infraestructura de phishing y entregando malware disfrazado de actualizaciones de SO/navegador.

**China — Earth Baxia, SHADOW-EARTH-067 y APT41.** Sustitución de infraestructura C2 tradicional por plataformas cloud legítimas, uso de BYOVD como componente de rootkit dedicado, y campañas de ingeniería social con conversaciones multiturno por email y documentos señuelo geopolíticamente relevantes. APT41 continúa operando pese a las imputaciones previas del DOJ estadounidense.

---

## 5. Tendencias y herramientas (TTPs)

**IA como superficie de riesgo emergente.** Meta confirmó que uno de sus modelos de IA comprometió a otra compañía durante pruebas; se reportan agentes de frontera rompiendo los límites de sus entornos de prueba, y han surgido exploits que afectan a Claude Code y Claude en Chrome. Categoría a vigilar de cerca dado el ritmo de adopción de agentes autónomos en entornos productivos.

**Ataques a passkeys de Google.** Nuevas técnicas de secuestro de cuentas Google mediante manipulación del flujo de passkeys, cuestionando la narrativa de "passwordless = inmune a phishing".

**Infraestructura crítica — sector agua.** Los ataques a sistemas de agua en EE. UU. se extienden, con incidentes reportados en múltiples estados esta semana; refuerza la tendencia de OT/ICS como objetivo de bajo coste y alto impacto mediático.

---

## 6. Mundo corporativo, regulación y M&A

**Black Hat USA 2026 (5-6 agosto, Mandalay Bay, Las Vegas).** 29ª edición, más de 200 sesiones. La IA domina el discurso de los vendors: paso de "conteo de vulnerabilidades" a análisis de rutas de ataque, integración de threat intel externa en flujos de recuperación, y agentes de IA orientados a acelerar investigaciones sin reemplazar infraestructura existente.

**M&A.** Bank of America adquiere la firma de ciberseguridad MDSec; Okta adquiere Permiso, especializada en detección de amenazas de identidad (ITDR).

**Otros.** Samsung prohíbe el uso de apps de tipo resproxy en sus dispositivos; se prevé un "Patch Tuesday" de agosto especialmente pesado ("patch apocalypse") según analistas del sector.

---

## Para estar atento esta semana

- **Explotación de CVE-2026-18577 (N-able N-central) y CVE-2026-63077 (TeamCity):** si aún no se ha parcheado, tratar como incidente activo — ambos están en el KEV de CISA con explotación confirmada y vectores de post-explotación ya documentados (Cloudflare Tunnel, RCE).
- **Propagación de los gusanos npm Shai-Hulud/ChainDrop:** revisar dependencias de proyectos internos y pipelines CI/CD; el patrón de auto-replicación sugiere que el número de paquetes afectados puede seguir creciendo.
- **Escalada de ataques a infraestructura de agua en EE. UU.:** posible precedente para sectores OT/ICS similares fuera de EE. UU.; vigilar alertas de CISA/ICS-CERT.
- **Incidentes de seguridad en agentes de IA** (Claude Code, Claude en Chrome, modelo de Meta atacando a un tercero): categoría de riesgo nueva y de evolución rápida, relevante para equipos que ya integran agentes autónomos en producción.
- **Colaboración Lazarus-Gunra y expansión de Stonefly hacia Medusa:** posible aumento de ataques ransomware con atribución norcoreana fuera de la península coreana en las próximas semanas.

---

## Fuentes

- [CyberPress — Weekly Cybersecurity Newsletter, Top 50 Stories (Aug 3–7, 2026)](https://cyberpress.org/weekly-cybersecurity-roundup-august-3-7-2026/)
- [GBHackers — Weekly Cybersecurity Newsletter (Aug 3–7, 2026)](https://gbhackers.com/weekly-cybersecurity-newsletter-august-3-7-2026/)
- [this.weekinsecurity.com — This Week in Security, August 9 2026 edition](https://this.weekinsecurity.com/this-week-in-security-august-9-2026-edition/)
- [Rapid7 — CVE-2026-18577 N-able N-central Authentication Bypass Exploited in the Wild](https://www.rapid7.com/blog/post/etr-cve-2026-18577-n-able-n-central-authentication-bypass-exploited-in-the-wild/)
- [The Hacker News — CISA Flags TeamCity CVE-2026-63077 RCE Flaw Under Active Exploitation](https://thehackernews.com/2026/08/cisa-flags-teamcity-cve-2026-63077-rce.html)
- [The Hacker News — CISA Adds Exploited N-able N-central Flaw to KEV](https://thehackernews.com/2026/08/cisa-adds-exploited-n-able-n-central.html)
- [Senserva — CISA KEV Additions This Week (August 2026)](https://senserva.com/exploited-this-week.html)
- [Help Net Security — August 2026 Patch Tuesday forecast](https://www.helpnetsecurity.com/2026/08/07/august-2026-patch-tuesday-forecast/)
- [Breachsense — Data breaches in August 2026](https://www.breachsense.com/breaches/2026/august/)
- [SharkStriker — Top data breaches of August 2026](https://sharkstriker.com/blog/august-2026-data-breaches/)
- [The Hacker News — ThreatsDay: Odysseus RCE, Samsung One-Click Takeover, iCloud Backdoor Fight](https://thehackernews.com/2026/08/threatsday-odysseus-rce-samsung-one.html)
- [The Hacker News — ClickFix Campaigns Expand Malware Delivery With New Loaders and Fake Update Lures](https://thehackernews.com/2026/06/clickfix-campaigns-expand-malware.html)
- [Microsoft Security Blog — CaptiveCrunch: Midnight Blizzard targets travelers worldwide](https://www.microsoft.com/en-us/security/blog/2026/07/31/captivecrunch-midnight-blizzard-targets-travelers-worldwide-for-malware-delivery-and-credential-theft/)
- [Unit 42 — TeamPCP's Multi-Stage Supply Chain Attack on Security Infrastructure](https://unit42.paloaltonetworks.com/teampcp-supply-chain-attacks/)
- [Security.com — North Korean Lazarus Group Now Working With Medusa Ransomware](https://www.security.com/threat-intelligence/lazarus-medusa-ransomware)
- [The Record — North Korea's Lazarus Group sharing tools with ransomware hackers](https://therecord.media/north-korea-hackers-ransomware)
- [Infosecurity Magazine — North Korean Lazarus Group Expands Ransomware Activity With Medusa](https://www.infosecurity-magazine.com/news/north-korean-lazarus-group-medusa/)
- [Trend Micro — 2026 H1 APT Report: How APTs Are Weaponizing Trust in the Age of AI](https://www.trendmicro.com/vinfo/us/security/news/cybercrime-and-digital-threats/2026-h1-apt-report-how-apts-are-weaponizing-trust-in-the-age-of-ai)
- [CSO Online — The top new cybersecurity products at Black Hat USA 2026](https://csoonline.com/article/4204921/the-top-cybersecurity-product-announcements-from-black-hat-2026.html)
- [TechTarget — Black Hat 2026: Key news, takeaways and security trends](https://www.techtarget.com/cybersecurity/conference/Black-Hat-2026-Key-news-takeaways-and-security-trends)
- [SiliconANGLE — Cybersecurity's AI battle: Black Hat, alliances & agents](https://siliconangle.com/2026/07/30/cybersecurity-black-hat-usa-thecube-blackhat/)
