# Informe semanal de ciberseguridad — semana del 26 oct al 2 nov 2026

## Resumen ejecutivo
La semana estuvo marcada por tres ejes: (1) explotación activa de vulnerabilidades críticas en infraestructura de desarrollo y perímetro (GitLab CVE-2026-97229 CVSS 10.0, "PHP2Shell" CVE-2026-109618/109619 encadenado con bugs de Palo Alto, Ivanti Connect Secure, Cisco ASA/FTD); (2) el mayor robo cripto registrado, el de Binance (~6.300 M$), atribuido por el FBI a TraderTraitor (Lazarus); y (3) la primera campaña estatal agéntica a escala: Vapor Panda (China) usando el framework de pentesting con IA ARTEX contra +200 organizaciones. Se consolida además la tendencia de ataques a la cadena de suministro de desarrolladores (extensión Claude Code troyanizada en Cursor Marketplace, LiteLLM en PyPI, GhostAction) y de sabotaje físico vía OT/IoT.

## 1. Vulnerabilidades críticas y CVEs
- **GitLab CE/EE — CVE-2026-97229 (CVSS 10.0)**: RCE no autenticado por integer overflow, explotado mediante una regex manipulada en configuración CI/CD. Añadida a CISA KEV el 29/10. Alpine Linux sufrió un intento (20-21 y 25-26/oct): el atacante tumbó workers 8 veces, el exploit falló en el siguiente paso en 4; "casi funciona". Reconstruyeron la instancia. Plazo CISA FCEB: 1/nov.
- **PHP2Shell — CVE-2026-109618 / CVE-2026-109619 (9.2)**: fallos de memoria con RCE no autenticado (demostrado contra Apache mod_php). Fixes: 8.2.35, 8.3.36, 8.4.27, 8.5.12. CISA confirmó explotación (alerta 26/oct). Dream observó el encadenamiento con bugs de Palo Alto Networks para comprometer 25 servidores (uno vinculado a gobierno de EE.UU.); sospecha de actor chino con exploit propio.
- **Ivanti Connect Secure**: nuevas fallas en KEV (31/oct) que forman cadena con bypass de autenticación CVE-2026-104935, explotado por UTA0576.
- **Otros**: LXD/MicroCloud CVE-2026-112816 (CVSS 10.0, suplantación de clientes de confianza → root en host); Cisco ASA/FTD CVE-2026-76418 (8.6, DoS explotado); WordPress Core RCE (~63.000 intentos en 24 h, parcheado 28/oct); Drupal Commerce CVE-2026-118539 explotado; TeamViewer CVE-2026-111585 (8.8, fix 15.83); Node.js (10 vulns, parches 26.12.1/24.22.1/22.23.4); Red Hat RHSB-2026-016 crítico sin parche aún.

## 2. Brechas y hackeos
- **Binance (~6.300 M$)**: hot wallets vaciadas en ~20 min; vector inicial: portátil DevOps comprometido con acceso a contenedores. FBI atribuye a TraderTraitor (Corea del Norte); laundering principalmente a ETH. ~3.000 M$ congelados.
- **Polymarket (~2,5-2,7 M$)**: script de terceros (DebugBear) sustituido por JS malicioso que drenó wallets de usuarios conectados. Reembolsos en curso.
- **DHS "Icepick"**: datos de 40.000+ empleados (CBP, ICE) extraídos desde un sistema Okta, publicados por DDoSecrets.
- **Otros**: Chainalysis (intrusión con técnicas agénticas), Frontline Education (~50 M estudiantes, SSN), Florida DoE (~5 M), Dinamarca (8,8 M registros), Capcom (11,7 M), Moldavia (80+ webs caídas, 750.000 afectados), 123-Reg (scraping con IA), Railway (posible acceso a secretos de clientes).

## 3. Ransomware y malware
- **Cadena de suministro dev**: la extensión `Anthropic.claude-code` en Cursor Marketplace fue troyanizada ~12 h (backdoor SSH persistente, ≤5.000 usuarios); LiteLLM 1.107.0 (7 wheels PyPI con infostealer y hooks de arranque Python); GhostAction (workflows maliciosos en 790 repos); npm "SlugFest" (+1.500 paquetes expuestos a takeover).
- **Ransomware**: informes de +41 % en actividad al inicio del "golden quarter"; Qilin y Akira siguen entre los más activos. Incidentes: Del Monte Foods (SSN expuestos), Colmobil (Israel), AB InBev (atribuido a "Emperador", no verificado independientemente).
- **Nuevo malware**: NetamLoader (loader .NET oculto en paquete de actualización de Windows, gobiernos asiáticos, Kaspersky); infostealer VOLTA con priorización de víctimas mediante modelos (Talos); implante hardware en Trezor revendido por CryptoBilis.

## 4. APTs y amenazas estatales
- **Vapor Panda (China, CrowdStrike)**: desde 17/oct, +200 organizaciones en 14+ países; ~140 comprometidas en 11 días (~70 % confirmadas/moderada confianza). Usa ARTEX (framework de pentest agéntico) con DeepSeek: explotación masiva + web shell, múltiples rutas en paralelo, de ~40 a +1.500 comandos/hora. Objetivos: ministerios de exteriores, defensa, telecos (EE.UU., Bélgica, Filipinas), farma, semiconductores, nuclear. Ataques agénticos: ~45 en septiembre → 126 en octubre. Campaña activa.
- **Volt Typhoon**: Lumen documenta la cadena "Silent Train" (routers/IoT como pre-posicionamiento en infraestructura crítica de EE.UU.).
- **TraderTraitor/Lazarus**: Binance (ver arriba).
- **Rusia/Irán — sabotaje físico**: hackers alineados con Rusia causando disrupción en España y ~20 países; sospecha de Irán en fallos de refrigeración en bases de EE.UU. y supermercados franceses; acceso backend a ~250.000 sistemas solares Sungrow en Alemania.

## 5. Tendencias y herramientas
- Ofensiva agéntica: ARTEX, intrusiones asistidas por IA (Chainalysis, 123-Reg), infostealers con triaje por LLM. El tiempo de operación pasa de minutos-humano a segundos-máquina.
- Supply chain de desarrollador y de terceros en frontend (script de tercero en Polymarket) como vector dominante.
- Ataques a OT/IoT con efecto físico (refrigeración, solar, hornos, climatización).
- Cloudflare: 18,2 M de ataques DDoS L3 en Q3 (vs 13,2 M en Q2), impulsados por booters.
- Spyware mercenario: notificaciones de Apple en 99 países; Discover-Tech dirigiendo malvertising a usuarios de Starlink.
- Fallo en ChatGPT (enlaces compartidos → código en sesión de la víctima), corregido como P1.
- eMudhra: 8 claves de CA intermedias recuperables por generación defectuosa; revocadas por navegadores; abandona su negocio TLS.

## 6. Mundo corporativo y regulación
- Consolidación continua: Accenture adquiere Dragos, runZero y NetRise (OT/exposición); Sophos adquiere Arco Cyber (IA agéntica); Cyderes–Lucidum. (Fechas exactas no verificadas; ver fuentes.)
- Regulación UE: NIS2, DORA y CRA siguen empujando requisitos de cadena de suministro y SBOM; el calendario 2027 (AI Act, CRA) se acerca. No se identificó una nueva norma específica esta semana.
- Character.ai en quiebra: venta de activos/datos de usuarios bajo escrutinio de fiscales generales. Wyden/Sección 702 en EE.UU.

## Para estar atento esta semana
1. **GitLab CVE-2026-97229**: con exploit parcialmente funcional visto en la naturaleza, espera exploit público/fiable pronto. Parchear y revisar logs de crashes de workers y regex en `.gitlab-ci.yml`.
2. **PHP2Shell + Palo Alto**: cadena ya usada; revisar PAN-OS expuestos y mod_php; probable explotación masiva oportunista.
3. **Vapor Panda/ARTEX**: más víctimas y posible filtración de IOCs/TTPs; considerar simulaciones de adversario con ritmo agéntico.
4. **Binance**: investigación forense en curso; posibles nuevos puntos de entrada y eventos de laundering.
5. **Cadena de suministro dev**: nuevas extensiones/paquetes comprometidos (Cursor, VS Code, PyPI, npm); rotar secretos, fijar versiones.
6. **Red Hat RHSB-2026-016** sin parche; **Ivanti/UTA0576**; **LXD CVE-2026-112816**.
7. Sabotaje OT físico en Europa: posible escalada con tensión geopolítica.

## Fuentes
- [This Week in Security, 1 nov 2026](https://this.weekinsecurity.com/this-week-in-security-november-1-2026-edition/)
- [The Hacker News](https://thehackernews.com/)
- [CISA añade GitLab RCE y Cisco DoS a KEV — THN](https://thehackernews.com/2026/10/cisa-adds-actively-exploited-gitlab-rce.html)
- [Hacktober 2026 — Cyberpress](https://cyberpress.org/hacktober-2026-cyberattacks/)
- [Vapor Panda / ARTEX — CyberScoop](https://cyberscoop.com/china-vapor-panda-agentic-ai-artex-campaign-crowdstrike/)
- [CISA: PHP critical vulnerabilities](https://www.cisa.gov/news-events/alerts/2026/10/21/critical-vulnerabilities-php-require-immediate-patching) (no se pudo abrir; datos vía otras fuentes)
- [Del Monte Foods — Morningstar/PR Newswire](https://www.morningstar.com/news/pr-newswire/20261030ph86031/del-monte-foods-data-breach-investigation-edelson-lechtzin-llp-probes-class-action-claims-after-a-ransomware-attack-exposes-social-security-numbers)
- [Emperador/AB InBev — Rescana](https://www.rescana.com/post/emperador-ransomware-attack-analysis-ab-inbev-azure-cloud-data-breach-and-pki-certificate-exposure-october-2026)
- [Ransomware +41 %](https://www.enterprisesecuritytech.com/post/ransomware-surges-41-as-cybercriminals-kick-off-the-golden-quarter)
- [VOLTA infostealer — Talos](https://blog.talosintelligence.com/appraisal-by-ai-model-driven-victim-prioritization-in-the-volta-infostealer)
- [Accenture adquiere Dragos, runZero y NetRise](https://threatcluster.io/cluster/accenture-acquires-dragos-runzero-and-netrise-for-ot-cyberse-560b08d1)
- [Sophos adquiere Arco Cyber](https://techeconomy.ng/sophos-acquires-arco-cyber-to-integrate-agentic-ai-into-its-cybersecurity-ecosystem/)
- [Calendario de cumplimiento 2027 (NIS2, DORA, CRA)](https://pasqualepillitteri.it/en/news/29268/digital-compliance-calendar-2027-ai-act-nis2-dora-cra)
