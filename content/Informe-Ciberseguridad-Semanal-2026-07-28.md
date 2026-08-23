# Informe Semanal de Ciberseguridad — Semana del 21 al 28 de julio de 2026

## Resumen ejecutivo

La semana estuvo marcada por dos zero-days activamente explotados en AD FS y SharePoint (Patch Tuesday de julio, 570 CVEs corregidos), una oleada de brechas corporativas vía terceros y phishing de voz (Estée Lauder, Abbott, Ernst & Young, Aflac), y una consolidación del ecosistema ransomware en torno a Qilin, LockBit 5.0 y The Gentlemen, que están operando casi como un cártel. En paralelo, se confirma la tendencia de fondo de 2026: agentes de IA autónomos usados tanto para ataques (post-explotación totalmente automatizada) como para defensa (MDASH de Microsoft), y un endurecimiento regulatorio real en la UE con las primeras sanciones bajo NIS2 y supervisión activa de DORA. Para un perfil red team/pentest, la prioridad inmediata es CVE-2026-56164 (SharePoint, sin autenticación) y CVE-2026-57092 (Windows VMSwitch, CVSS 9.9).

## 1. Vulnerabilidades críticas y CVEs

**Patch Tuesday de julio — 570 CVEs, 2 zero-days explotados.** Microsoft corrigió 570 vulnerabilidades (57 críticas). Dos ya se explotaban activamente antes del parche:
- **CVE-2026-56164** (SharePoint Server): fallo de autenticación faltante que permite a un atacante remoto no autenticado escalar privilegios sin interacción del usuario. CISA fijó plazo de parcheo para agencias federales al 17 de julio.
- **CVE-2026-56155** (AD FS): escalada de privilegios local por control de acceso insuficiente. Plazo CISA: 28 de julio.
- **CVE-2026-57092** (Windows VMSwitch): CVSS 9.9, elevación de privilegios crítica, también explotada in-the-wild.

**Zoom — CVE-2026-53412.** Fallo crítico en Windows que permitiría a un atacante no autenticado tomar control de la cuenta; tres fallos adicionales de severidad alta habilitan escalada de privilegios.

**Linux kernel — CVE-2026-53264 (CVSS 7.8).** Use-after-free en el subsistema de traffic-control de red que permite escalar de usuario local a root en CentOS Stream 9. El investigador (Lee Jia Jie) señaló que usó IA para acelerar el descubrimiento y desarrollo del exploit — dato relevante como indicador de tendencia en investigación ofensiva asistida por IA.

Ambos CVEs de Microsoft ya están en el catálogo KEV de CISA; recomendable priorizar su remediación sobre cualquier otro hallazgo de esta semana.

## 2. Brechas y hackeos

- **Ernst & Young:** brecha vía compromiso de un sistema de tickets de soporte de un tercero, con exposición de datos personales y financieros usados en preparación de declaraciones fiscales.
- **Estée Lauder:** explotación de una falla en Oracle E-Business Suite (módulo de RR.HH.).
- **Abbott Laboratories:** comprometida por el grupo **ShinyHunters** mediante una campaña de vishing (phishing de voz) dirigida a empleados.
- **Aflac (filial Japón):** robo de datos personales y bancarios.
- **Chick-fil-A:** cuentas de clientes comprometidas por credential stuffing.
- **RevolutionParts:** exposición de más de 5 millones de registros en plataforma de e-commerce para concesionarios automotrices.

Tendencia: el vector dominante de esta semana no fue la explotación directa de software, sino compromiso de terceros/proveedores e ingeniería social (vishing, credential stuffing), reforzando la necesidad de auditar la cadena de suministro y los controles de verificación de identidad en mesas de ayuda.

## 3. Ransomware y malware

- **Qilin vs. The Gentlemen:** ambos grupos compiten por el primer puesto de actividad ransomware en 2026; Qilin ya acumula más de 500 víctimas solo este año (+443% interanual), operando en más de 50 países.
- **LockBit 5.0:** repunte notable tras el colapso de ALPHV/BlackCat; 163 víctimas en Q1 2026, cuarto puesto global. Indicios de "cartelización" con Qilin y Akira — alianzas entre operadores para compartir infraestructura y afiliados.
- **Anubis / Fairlife:** exfiltración reportada de 1 TB de datos confidenciales, con plazo de filtración fijado a esta semana bajo el modelo clásico de doble extorsión.
- **Pago de rescate por agencia gubernamental de EE.UU.:** ~9.44 BTC (~1M USD) pagado al grupo "Kairos" para recuperar archivos robados — caso a seguir por sus implicaciones en política de no-pago.

## 4. APTs y amenazas estatales

- **Lazarus Group (Corea del Norte):** continúa siendo el actor más prolífico en robo de criptoactivos; en 2026 los actores vinculados a la RPDC acumulan cientos de millones de dólares robados (incluyendo el caso Kelp DAO, ~290M USD, vía compromiso de servidores de verificación de LayerZero), sumándose al histórico acumulado de 6.75B USD. La ONU vincula estos fondos al programa de misiles balísticos y armamento nuclear norcoreano.
- **Salt Typhoon / Volt Typhoon (China):** persiste el pre-posicionamiento en infraestructura crítica y telecomunicaciones de EE.UU. Salt Typhoon (MSS) mantiene foco en espionaje sobre operadoras; Volt Typhoon construye infraestructura de sabotaje para uso en un eventual conflicto geopolítico. En respuesta, EE.UU. mantiene un programa de 3.000M USD para retirar hardware de origen chino de redes críticas.
- **APT29/Cozy Bear:** actividad histórica reciente con el malware WINELOADER contra diplomáticos europeos sigue como referencia de TTPs activos del grupo (spear-phishing + loaders en memoria).

## 5. Tendencias, TTPs y herramientas

- **Post-explotación 100% autónoma por IA:** documentado el primer caso de un agente LLM ejecutando de forma completamente autónoma una cadena de post-explotación (conexión WebSocket a un notebook marimo expuesto, movimiento lateral, cosecha de credenciales cloud y exfiltración de clave SSH vía pool de Cloudflare Workers) — sin intervención humana en el bucle.
- **Supply chain / agentes ofensivos IA:** julio se vio marcado por el "poisoning" de la GitHub Action de Claude Code y "GuardFall", un fallo de diseño de shell injection universal que afecta a más de 500.000 despliegues open-source.
- **TOCTOU contra agentes de código/computer-use:** Johann Rehberger demostró un ataque time-of-check/time-of-use contra agentes de IA con acceso a herramientas — vector relevante para quienes evalúan seguridad de copilots y agentes de codificación.
- **Defensa asistida por IA:** Microsoft lanzó su primer modelo específico de ciberseguridad dentro de MDASH (identificación y remediación de vulnerabilidades), con 95.95% en el benchmark CyberGym.

## 6. Mundo corporativo y regulación

- **M&A:** el mercado de ciberseguridad sigue muy activo — 37 operaciones anunciadas en junio, entre ellas Cisco/WideField Security (detección de amenazas de identidad para el Agentic SOC de Splunk), 1Password/Apono (gobernanza de acceso just-in-time, 250-300M USD) y A10 Networks/TrojAI (red-teaming de IA).
- **NIS2:** primeras sanciones administrativas ya emitidas en la UE (Q1 2026); obligaciones se completan en octubre de 2026.
- **DORA:** entra en su primer ciclo real de supervisión activa por parte de BaFin, AFM/DNB y ACPR/AMF, con foco en entidades financieras.

## Para estar atento esta semana

- **Explotación adicional de CVE-2026-56155 y CVE-2026-56164** más allá del sector gobierno de EE.UU. — vigilar PoCs públicos y su uso por afiliados de ransomware.
- **Escalada de la rivalidad Qilin/The Gentlemen**, que podría traducirse en un aumento de volumen de ataques a corto plazo por presión competitiva entre afiliados.
- **Filtración de datos de Fairlife (Anubis)** — el plazo de extorsión vence esta semana; posible publicación completa del terabyte exfiltrado.
- **Nuevos PoCs de post-explotación asistida por IA** — GuardFall y el vector de shell injection en despliegues open-source siguen sin remediación completa en gran parte del ecosistema afectado.
- **Primeras sanciones NIS2 adicionales** conforme se acerca el plazo de octubre 2026, especialmente en sectores de infraestructura crítica no financiera.

---

### Fuentes

- [Tenable — Microsoft's July 2026 Patch Tuesday](https://www.tenable.com/blog/microsofts-july-2026-patch-tuesday-addresses-569-cves-cve-2026-56155-cve-2026-56164)
- [SecurityOnline — July 2026 Patch Tuesday](https://securityonline.info/july-2026-patch-tuesday/)
- [Senserva — Exploited This Week: CISA KEV Additions](https://senserva.com/exploited-this-week.html)
- [eSecurity Planet — Weekly Roundup](https://www.esecurityplanet.com/weekly-roundup/ai-agents-trust-abuse-and-breaches-define-cybersecurity-news-this-week-of-july-2026/)
- [Privacy Guides — Data Breach Roundup (July 17-23, 2026)](https://www.privacyguides.org/news/2026/07/24/data-breach-roundup-july-17-23-2026/)
- [TechCrunch — The worst hacks and breaches of 2026 so far](https://techcrunch.com/2026/07/07/the-worst-hacks-and-breaches-of-2026-so-far/)
- [SharkStriker — July 2026 Data Breaches](https://sharkstriker.com/blog/july-2026-data-breaches/)
- [GBHackers — 2026 Ransomware Report](https://gbhackers.com/2026-ransomware-report/)
- [Industrial Cyber — Ransomware sector reconsolidating Q1 2026](https://industrialcyber.co/ransomware/ransomware-sector-reconsolidating-as-qilin-lockbit-and-the-gentlemen-expand-influence-in-q1-2026/)
- [Dark Web Informer — Ransomware Attack Update July 14](https://darkwebinformer.com/ransomware-attack-update-july-14th-2026/)
- [Sanctions.io — Lazarus Group and DPRK Crypto Theft 2026](https://www.sanctions.io/blog/the-lazarus-group-and-dprk-crypto-theft-in-2026)
- [Androidheadlines — Lazarus $290M Kelp DAO heist](https://www.androidheadlines.com/2026/04/north-koreas-lazarus-steals-290m-in-large-crypto-heist.html)
- [Congress.gov — Salt Typhoon CRS report](https://www.congress.gov/crs_external_products/IF/HTML/IF12798.web.html)
- [NJCCIC — Volt Typhoon](https://www.cyber.nj.gov/threat-landscape/nation-state-threat-analysis-reports/china-linked-cyber-operations-targeting-us-critical-infrastructure/volt-typhoon)
- [Adversa AI — Top Agentic AI Security Resources, July 2026](https://adversa.ai/blog/top-agentic-ai-security-resources-july-2026/)
- [Neteye Blog — The AI Cyber Attacks Explosion in 2026](https://www.neteye-blog.com/blog/2026/07/03/the-ai-cyber-attacks-explosion-in-2026-emerging-threats/)
- [SecurityWeek — Cybersecurity M&A Roundup, June 2026](https://www.securityweek.com/cybersecurity-ma-roundup-37-deals-announced-in-june-2026/)
- [ComplianceHub.Wiki — DORA Enforcement and NIS2 2026](https://compliancehub.wiki/dora-nis2-2026-enforcement-eu-financial-cyber-resilience-compliance/)
