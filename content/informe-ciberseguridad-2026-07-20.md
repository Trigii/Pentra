# Informe Semanal de Ciberseguridad — Semana del 14 al 20 de julio de 2026

*Orientado a red team / pentesting · Elaborado el lunes 20 de julio de 2026*

---

## Resumen ejecutivo

Semana dominada por el Patch Tuesday de julio más voluminoso de la historia y por la explotación activa de un 0-day de SharePoint on-prem (CVE-2026-58644, CVSS 9.8) ya en el catálogo KEV de CISA. En el frente de brechas, Accenture confirmó el robo de ~35 GB (código fuente, claves RSA/SSH y tokens de Azure) por el actor "888", mientras el ransomware volvió a demostrar impacto físico al forzar la parada de la producción estadounidense de Fairlife (Coca-Cola). En el plano ofensivo destaca una técnica sigilosa de enumeración de cuentas en Microsoft Entra vía *OAuth Client ID spoofing* que evade el logging y las políticas de acceso condicional. Regulatoriamente, EE.UU. congela la fase 2 de CMMC y prepara la norma final de CIRCIA para septiembre.

---

## 1. Vulnerabilidades críticas y CVEs

**CVE-2026-58644 — SharePoint Server RCE (CVSS 9.8) · EXPLOTADO EN LA WILD.** Deserialización de datos no confiables que permite ejecución remota de código sobre SharePoint on-prem (Subscription Edition, 2019 y 2016). CISA lo añadió al catálogo KEV el 16 de julio con plazo de remediación para agencias FCEB el 19 de julio. La cadena de post-explotación observada incluye robo de *machine keys* de IIS y despliegue de persistencia. Aplicar el parche del 14 de julio y verificar que la integración AMSI está activa en cada aplicación web. Vector de altísimo interés para red team por su exposición directa a internet y baja complejidad.

**CVE-2026-xxxx — SAP NetWeaver ABAP (CVSS 9.9).** SAP parcheó un fallo crítico en NetWeaver ABAP que podría permitir exponer o modificar datos de negocio. Prioridad alta en entornos SAP dado el histórico de explotación rápida de estos componentes.

**Patch Tuesday de julio (récord histórico).** Microsoft publicó uno de los ciclos más grandes registrados. Entre los fallos críticos sin explotación conocida pero de vector remoto sin interacción: CVE-2026-57092 (VMSwitch, use-after-free, CVSS 9.9), CVE-2026-56188 (driver de red de Windows Server, race condition, RCE no autenticado, CVSS 9.8) y CVE-2026-50518 / CVE-2026-56159 (DHCP Server, heap overflow, RCE no autenticado, CVSS 9.8). Además de CVE-2026-58644, se explota activamente CVE-2026-56155 (ADFS, elevación de privilegios).

**Parches de emergencia fuera de ciclo.** Se hizo público el código de explotación de un fallo de Firefox, mientras Chrome, Adobe y VMware publicaron parches de emergencia la misma semana. Priorizar navegadores y VMware en el ciclo de patching de esta semana.

---

## 2. Brechas y hackeos

**Accenture — ~35 GB exfiltrados (actor "888").** El atacante afirma haber robado código fuente, tokens de acceso personal de Azure, claves de cifrado RSA y claves SSH en una intrusión de principios de julio. Accenture minimizó el incidente ("asunto aislado, ya remediado, sin impacto en operaciones"), pero el material expuesto —secretos y código— habilita movimiento lateral hacia repositorios y almacenamiento cloud, con potencial efecto cascada sobre clientes Fortune Global 500. El mismo alias "888" ya reclamó una filtración de Accenture en 2024.

**Abbott — ciberataque a su división de diagnóstico oncológico.** La compañía divulgó un incidente en su negocio de *cancer diagnostics*. Sector salud, con implicaciones de continuidad y datos sensibles.

**Aseguradora de automóviles — brecha de datos.** Un proveedor de seguros de coche sufrió exposición de datos de clientes (reportado por Cybersecurity Dive). Continúa la tendencia de targeting al sector asegurador por el valor de PII y datos financieros.

---

## 3. Ransomware y malware

**Coca-Cola / Fairlife — parada de producción por ransomware.** El ataque suspendió temporalmente toda la producción estadounidense de la marca láctea Fairlife (~4.000 M$ en ventas); las operaciones en Canadá no se vieron afectadas. Fairlife detectó acceso no autorizado a sistemas de producción, activó su plan de respuesta y notificó a las autoridades. No se ha confirmado exfiltración ni grupo responsable. Ejemplo claro de impacto OT/negocio: el ransomware sigue traduciéndose en interrupción física de manufactura.

**Qilin consolida su dominio del mercado RaaS.** Qilin se mantiene como el grupo más activo entrando en 2026, con víctimas notables como el gigante alimentario Danone (filtración reclamada de ~90.000 archivos) y numerosas pymes. Analistas describen el ecosistema como un "monstruo de cuatro cabezas" (concentración en pocos grupos dominantes — Qilin, Play, Akira y afines) que absorbe la mayor parte de los ataques, con creciente uso de IA en las operaciones.

**Play y BlackCat siguen operativos.** Los roundups de la semana confirman actividad continuada de Play y ramas asociadas a BlackCat, junto con Qilin, como protagonistas del panorama de extorsión.

---

## 4. APTs y amenazas estatales

**China — CylindricalCanine / GoldenEyeDog vinculado al incidente DigiCert.** Investigadores atribuyeron el incidente de seguridad de DigiCert (abril 2026) al clúster CylindricalCanine, subgrupo de GoldenEyeDog (aka APT-Q-27 / Dragon Breath / Miuuti Group), grupo chino con histórico en los sectores de gaming y apuestas. Los APT chinos siguen ampliando objetivos y actualizando backdoors.

**Corea del Norte — Contagious Interview con esteganografía.** Actores norcoreanos ocultan payloads maliciosos dentro de imágenes SVG mediante esteganografía, dentro de la campaña de falsas ofertas de empleo y retos de programación dirigida a desarrolladores. TTP relevante para simulacros de ingeniería social a equipos técnicos.

**Rusia — ClickFix contra objetivos ucranianos.** Actores estatales rusos emplean la técnica ClickFix para inducir a víctimas ucranianas a autoinfectarse con infostealers. **Irán — Screening Serpens** continúa campañas de espionaje documentadas por Unit 42. En conjunto, entre febrero de 2025 y mediados de 2026 se documentan 297+ ataques a la cadena de suministro y un *breakout time* de referencia de 72 minutos (reducción de 4x).

---

## 5. Tendencias y herramientas (TTPs)

**OAuth Client ID Spoofing en Microsoft Entra — enumeración sigilosa.** Proofpoint documentó una técnica que aprovecha las respuestas diferenciadas de Entra ID según la validez del client ID para enumerar usuarios e inferir validez de contraseñas **sin generar eventos de inicio de sesión exitoso**. Los IDs falsificados dejan el campo de aplicación en blanco en los logs y no disparan políticas de acceso condicional acotadas a apps concretas. La campaña de enero usó 700.000+ IDs falsos contra 1M+ de cuentas en ~4.000 organizaciones. Herramienta ofensiva de referencia asociada: TeamFiltration (campaña UNK_SneakyStrike). Muy relevante para operaciones de *password spraying* evasivas contra M365.

**Cadena de suministro npm bajo presión continua.** Unit 42 actualizó (15 de julio) su seguimiento del panorama de amenazas en npm, con superficie de ataque creciente vía paquetes maliciosos y typosquatting. Reforzar controles de dependencias y SCA en pipelines.

---

## 6. Mundo corporativo, regulación y M&A

**EE.UU. congela la fase 2 de CMMC.** El Pentágono suspendió la expansión de la fase 2 del programa CMMC para una revisión de 60 días, citando costes, capacidad limitada de evaluación y barreras para pequeños contratistas de defensa.

**CIRCIA — norma final en septiembre.** CISA espera publicar la regla final de CIRCIA en septiembre de 2026: reporte de incidentes en 72 h y de pagos de ransomware en 24 h para entidades cubiertas de infraestructura crítica.

**Financiación y adquisiciones.** La británica Valarian levantó 70 M$ para su tecnología ACRA. Semana activa de M&A con movimientos de 1Password, Accenture, Cisco, F5, Rubrik y SailPoint. En Europa siguen vigentes las obligaciones de NIS2 y DORA como marco de referencia para el sector financiero y de infraestructura crítica.

---

## Para estar atento esta semana

- **Explotación masiva de CVE-2026-58644 (SharePoint):** con PoC y post-explotación conocida (robo de machine keys), es previsible un aumento de escaneo e intrusiones contra SharePoint on-prem no parcheado. Priorizar inventario y parcheo inmediato; cazar IIS machine keys comprometidas.
- **Caso Accenture:** vigilar posible efecto cascada hacia clientes si el código y los secretos filtrados son recientes. Rotar credenciales y revisar accesos de terceros vinculados a Accenture.
- **Fairlife/Coca-Cola:** pendiente de reivindicación de grupo y de confirmación de exfiltración; posible aparición en sitio de filtraciones en los próximos días.
- **OAuth Client ID spoofing:** revisar detecciones de enumeración en Entra (entradas con campo de aplicación en blanco, user-agents de TeamFiltration) antes de que la técnica se generalice.
- **Riesgo FIFA World Cup 2026:** roundups de la semana señalan aumento de riesgo cibernético asociado al evento; anticipar campañas de phishing y fraude temáticas.

---

## Fuentes

- [Zero Day Initiative — The July 2026 Security Update Review](https://www.thezdi.com/blog/2026/7/14/the-july-2026-security-update-review)
- [CrowdStrike — July 2026 Patch Tuesday Analysis](https://www.crowdstrike.com/en-us/blog/patch-tuesday-analysis-july-2026/)
- [The Hacker News — CISA Adds Exploited SharePoint RCE Zero-Day CVE-2026-58644 to KEV](https://thehackernews.com/2026/07/cisa-adds-exploited-sharepoint-rce-zero.html)
- [CISA — Urges SharePoint Hardening After New Exploitations](https://www.cisa.gov/news-events/alerts/2026/07/14/cisa-urges-sharepoint-hardening-after-new-exploitations)
- [Rapid7 — CVE-2026-58644 SharePoint Unauthenticated RCE Exploited in the Wild](https://www.rapid7.com/blog/post/etr-cve-2026-58644-microsoft-sharepoint-server-unauthenticated-remote-code-execution-vulnerability-exploited-in-the-wild/)
- [The Hacker News — SAP Patches CVSS 9.9 NetWeaver ABAP Flaw](https://thehackernews.com/2026/07/sap-patches-cvss-99-netweaver-abap-flaw.html)
- [TechTimes — Firefox Exploit Code Goes Public as Chrome, Adobe, VMware Ship Emergency Patches](https://www.techtimes.com/articles/320970/20260719/firefox-exploit-code-goes-public-chrome-adobe-vmware-ship-emergency-patches.htm)
- [Cybersecurity Dive — Accenture faces massive data breach](https://www.cybersecuritydive.com/news/accenture-data-breach-access-keys-source-code/824694/)
- [BleepingComputer — Accenture confirms breach after hacker offers stolen data for sale](https://www.bleepingcomputer.com/news/security/accenture-confirms-breach-after-hacker-offers-stolen-data-for-sale/)
- [Cybersecurity Dive — Abbott discloses cyberattack on cancer diagnostics business](https://www.cybersecuritydive.com/news/abbott-discloses-cyberattack-on-cancer-diagnostics-business/825552/)
- [Cybersecurity Dive — Data breach hits car insurance provider](https://www.cybersecuritydive.com/news/data-breach-car-insurance-provider/824835/)
- [BleepingComputer — Coca-Cola says Fairlife ransomware attack halts US dairy production](https://www.bleepingcomputer.com/news/security/coca-cola-says-fairlife-ransomware-attack-halts-us-dairy-production/)
- [Cybersecurity Dive — Ransomware attack forces Coca-Cola to suspend US production](https://www.cybersecuritydive.com/news/ransomware-attack-coca-cola-suspend-production-dairy/825540/)
- [Cybersecurity Dive — Ransomware ecosystem grows, but 'four-headed monster' dominates](https://www.cybersecuritydive.com/news/ransomware-concentrated-ai-guidepoint/824828/)
- [Cybernews — Qilin ransomware claims global food giant Danone](https://cybernews.com/news/danone-evian-silk-international-delight-qilin-ransomware-attack/)
- [SecurityWeek — Chinese APTs Expand Targets, Update Backdoors in Recent Campaigns](https://www.securityweek.com/chinese-apts-expand-targets-update-backdoors-in-recent-campaigns/)
- [Cybersecurity Dive — Hackers find a new trick to collect Microsoft Entra user data](https://www.cybersecuritydive.com/news/microsoft-entra-user-enumeration-bypass-proofpoint/825052/)
- [The Hacker News — OAuth Client ID Spoofing Lets Attackers Validate Stolen Entra Credentials](https://thehackernews.com/2026/07/oauth-client-id-spoofing-lets-attackers.html)
- [Proofpoint — TeamFiltration / UNK_SneakyStrike Account Takeover Campaign](https://www.proofpoint.com/us/blog/threat-insight/attackers-unleash-teamfiltration-account-takeover-campaign)
- [Unit 42 — The npm Threat Landscape (Updated July 15)](https://unit42.paloaltonetworks.com/monitoring-npm-supply-chain-attacks/)
- [Check Point Research — 20th July Threat Intelligence Report](https://research.checkpoint.com/2026/20th-july-threat-intelligence-report/)
- [Federal News Network — CIRCIA, other big cyber rules expected to get finalized this fall](https://federalnewsnetwork.com/cybersecurity/2026/07/circia-other-big-cyber-rules-expected-to-get-finalized-this-fall/)
- [Hipther — Cybersecurity Roundup July 14, 2026 (CMMC review, funding, M&A)](https://hipther.com/latest-news/2026/07/14/115096/cybersecurity-roundup-pentagon-cmmc-review-nsa-router-guidance-new-jersey-cyber-grants-anthropic-ai-controls-shinyhunters-and-the-mit-cybersecurity-clinic-july-14-2026/)
