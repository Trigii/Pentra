# Informe Semanal de Ciberseguridad — Semana del 30 de junio al 7 de julio de 2026

## Resumen ejecutivo

La semana ha estado dominada por la explotación activa de una RCE en SharePoint (CVE-2026-45659) que CISA añadió a su catálogo KEV con plazo de parcheo federal, y por la consolidación de la campaña **FortiBleed**, ahora atribuida a las operaciones de ransomware INC y Lynx tras el robo de credenciales VPN en ~12.000 FortiGate. En el frente estatal, Kaspersky destapó un nuevo APT (**Armored Likho**) que ataca gobiernos y sector eléctrico con el infostealer BusySnake. A nivel de superficie de ataque, destacan nuevas técnicas de robo de sesión (**ConsentFix**, evolución de ClickFix que evade MFA) y la aparición de ransomware totalmente automatizado por IA (**JadePuffer**). El panorama regulatorio europeo entra en fase de supervisión activa de DORA y avanza hacia un "Cybersecurity Act 2.0".

---

## 1. Vulnerabilidades críticas y CVEs

**CVE-2026-45659 — Microsoft SharePoint Server RCE (explotación activa).** CISA la incorporó al catálogo KEV tras confirmar explotación en la wild, con plazo de parcheo para agencias federales el 4 de julio. Es una RCE por deserialización de datos no confiables (CVSS 8.8) que un atacante autenticado con permisos mínimos de *Site Member* puede disparar sin privilegios de administrador. Microsoft la corrigió en mayo de 2026 para Subscription Edition, 2019 y Enterprise 2016. **Prioridad de parcheo inmediata** para cualquier SharePoint on-premise expuesto.

**CVE-2026-40138 — BeyondTrust Remote Support / Privileged Remote Access (CVSS 9.2).** Vulnerabilidad de pre-autenticación en el subsistema de autenticación que permite a un atacante posicionado en red saltarse los controles de acceso. Dado que estos productos son objetivo de alto valor para pivoting hacia entornos privilegiados, conviene aplicar las actualizaciones sin demora.

**CVE-2026-20896 — Gitea (imágenes Docker, CVSS 9.8).** La plataforma confía en la cabecera `X-WEBAUTH-USER` desde cualquier IP de origen, lo que permite a un cliente no autenticado de Internet obtener acceso elevado. Ya se observan intentos de explotación. Revisar despliegues Gitea en contenedores y restringir el acceso a la interfaz.

**Menciones adicionales:** backdoor de autenticación no documentado en firmware Tenda (**CVE-2026-11405**) que permite control administrativo completo sin credenciales; y seis nuevas entradas KEV la semana previa afectando a PTC Windchill, Cisco Unified Communications Manager, Lantronix EDS5000 y Ubiquiti UniFi OS.

---

## 2. Brechas y hackeos

**Volcado masivo de 24.000 millones de registros.** Investigadores de Cybernews descubrieron una base de datos expuesta en Internet con credenciales agregadas de 36 fuentes (canales de Telegram, compilaciones de brechas previas y logs de infostealers). Aunque no es una brecha "nueva" sino una recopilación, alimenta directamente ataques de credential stuffing y account takeover a gran escala.

**KDDI — hasta 14,22 millones de correos y contraseñas expuestos.** La operadora japonesa reveló una brecha en la plataforma de email que provee a seis ISP (STNet, KDDI Web Communications, JCOM, Chubu Telecommunications, Nifty y Biglobe). Impacto directo sobre clientes de telecomunicaciones y riesgo de phishing dirigido subsiguiente.

**UNK_MassTraction — espionaje contra universidades vía Roundcube.** Un clúster alineado con China explota fallos críticos en el webmail open-source Roundcube de departamentos de física e ingeniería de universidades de EE.UU. y Canadá para robar credenciales y desplegar web shells o herramientas de post-explotación. Objetivo clásico de exfiltración de investigación de doble uso.

---

## 3. Ransomware y malware

**FortiBleed → INC y Lynx (el hallazgo de la semana).** La campaña de robo de credenciales FortiBleed, que usó un sniffer en Go ("FortiGate Sniffer") sobre ~12.000 FortiGate para interceptar credenciales VPN, ha quedado vinculada directamente a ransomware. SOCRadar documentó escaneo contra ~11.250 portales FortiGate en más de 150 países, acceso admin confirmado en 409 objetivos y cadena de ataque completa en 354. Un mismo operador aparecía logueado en los paneles de negociación de **INC Ransom** y **Lynx**, con víctimas solapadas. Al menos 12 despliegues de ransomware confirmados. **Recomendación red team/defensa:** auditar FortiGate en busca de binarios anómalos, rotar todas las credenciales VPN y revisar accesos admin históricos.

**JadePuffer — ransomware automatizado por IA.** El grupo desplegó un agente basado en LLM que ejecuta toda la cadena de ataque de forma autónoma: acceso inicial, robo de credenciales, movimiento lateral, cifrado y borrado de la base de datos de producción. Marca un cambio cualitativo en la velocidad y escalabilidad de las operaciones de extorsión.

**Contexto del mes:** junio cerró con 102 ataques de ransomware divulgados públicamente en 21 países, con **sanidad como sector más golpeado** (30 incidentes) y un ecosistema fragmentado en 31 grupos activos. Nuevas campañas de malware esta semana incluyen **Djinn Stealer** (contra servidores SimpleHelp expuestos, cazando credenciales de nube, claves Git y tokens Docker — orientado a desarrolladores) y **Atomic Stealer** distribuido mediante anuncios de cuentas verificadas en X.

---

## 4. APTs y amenazas estatales

**Armored Likho (nuevo APT, informe Kaspersky del 6 de julio).** Ataca organismos gubernamentales y del sector eléctrico en Rusia, Brasil y Kazajistán, combinando motivación financiera y ciberespionaje. Su arsenal incluye RATs modulares, el infostealer en Python **BusySnake Stealer** y herramientas como **Go2Tunnel** para túnel de red y acceso remoto. Vector inicial: spear-phishing suplantando comunicaciones oficiales de gobierno o asistencia social, con adjuntos ejecutables o LNK disfrazados de documentos.

**Salt Typhoon (China) sigue activo.** El grupo detrás del hackeo a las telecos de EE.UU. de 2024 persiste dentro de redes estadounidenses, con penetración confirmada este año de correos de comités de la Cámara de Representantes. Recordatorio de que las intrusiones estatales priorizan persistencia a largo plazo sobre impacto inmediato.

**Campaña China-nexus con software fiscal falso.** Operación de spear-phishing que usa utilidades falsas de declaración de impuestos india para distribuir **DcRAT**, atribuida a grupos con nexo chino.

---

## 5. Tendencias, técnicas y herramientas

**ConsentFix — la evolución de ClickFix que evade MFA.** A diferencia de ClickFix (que convierte a la víctima en el instalador), ConsentFix convierte a la víctima en el proveedor de identidad: muestra una página de inicio de sesión de Microsoft aparentemente legítima y pide arrastrar un enlace de callback a `localhost` al navegador, permitiendo al atacante obtener tokens de sesión que dan acceso a Microsoft 365 **sin contraseña ni MFA**. Técnica de alto valor para operaciones de phishing en engagements y una amenaza real de robo de sesión.

**TTPs de Active Directory al alza.** Ganan tracción **BadSuccessor** (escalada de privilegios de dominio), ataques cross-forest (Cross-Forest RBCD, AD CS) y abuso de AD Sites, además de técnicas actualizadas de identidad híbrida adaptadas a la nueva arquitectura de Entra Connect. El enfoque **living-off-the-land (LOTL)** sigue evadiendo detección más tiempo en entornos maduros.

**Tooling ofensivo.** Cobalt Strike (Beacon) continúa como estándar de simulación de adversario y C2 con alineación directa a MITRE ATT&CK; Outflank Security Tooling (OST) mantiene su relevancia para emulación de APTs y evasión de defensas.

---

## 6. Mundo corporativo y regulación

**DORA pasa de guía a supervisión activa.** Autoridades nacionales —BaFin (Alemania), AFM y DNB (Países Bajos), ACPR y AMF (Francia)— ya conducen revisiones y auditorías de resiliencia operativa digital sobre entidades financieras de la UE.

**"Cybersecurity Act 2.0".** Presentado en enero de 2026, plantea ajustar la Directiva NIS2, actualizar los marcos europeos de certificación y redefinir el rol de ENISA en el reporte de incidentes.

**NIS2 — implementación desigual.** Persiste un mosaico de obligaciones nacionales. En Alemania la ley de trasposición se publicó el 5 de diciembre de 2025 con registro obligatorio antes del 6 de marzo de 2026, pero solo alrededor de un tercio de las entidades obligadas se ha registrado. El Grupo de Cooperación NIS2 adoptó el 26 de mayo plantillas comunes de reporte de incidentes.

---

## Para estar atento esta semana

- **Explotación en cadena de CVE-2026-45659 (SharePoint):** al estar ya en KEV con exploit activo, es previsible un aumento de intentos oportunistas contra instancias sin parchear. Vigilar telemetría de deserialización y web shells en SharePoint on-prem.
- **Fan-out de FortiBleed hacia más despliegues de ransomware:** con 354 cadenas completadas y solo 12 despliegues confirmados hasta ahora, existe un gap de accesos potencialmente monetizables por INC/Lynx en las próximas semanas.
- **Adopción de ConsentFix:** al evadir MFA vía tokens de sesión, puede escalar rápido en campañas de phishing masivo. Revisar políticas de consentimiento OAuth y detección de callbacks a localhost.
- **Ransomware automatizado por IA (JadePuffer):** si el modelo se replica, cabe esperar una reducción drástica del *dwell time* y mayor volumen de ataques con menos operadores humanos.

---

## Fuentes

- [The Hacker News — SharePoint RCE CVE-2026-45659 añadido a CISA KEV](https://thehackernews.com/2026/07/sharepoint-rce-cve-2026-45659-added-to.html)
- [SecurityWeek — CISA advierte de la vulnerabilidad de SharePoint explotada activamente](https://www.securityweek.com/cisa-warns-of-actively-exploited-microsoft-sharepoint-vulnerability/)
- [Hackerstorm — Weekly CISA KEV Updates 29 June 2026](https://www.hackerstorm.com/articles/our-blog/government-regulatory-cyber-alerts/weekly-cisa-kev-updates-29-june-2026-six-new-known-exploited-vulnerabilities-added)
- [The Hacker News — FortiBleed vinculado a INC y Lynx](https://thehackernews.com/2026/07/fortibleed-credential-theft-linked-to.html)
- [BleepingComputer — FortiBleed credential-theft campaign linked to Lynx ransomware](https://www.bleepingcomputer.com/news/security/fortibleed-credential-theft-campaign-linked-to-lynx-ransomware/)
- [Cybersecurity Dive — FortiBleed traced to INC and Lynx](https://www.cybersecuritydive.com/news/fortibleed-campaign-traced-to-inc-and-lynx-ransomware-operations/824348/)
- [The Hacker News — Armored Likho / BusySnake Stealer](https://thehackernews.com/2026/07/armored-likho-targets-government.html)
- [Dark Reading — BusySnake Stealer en infraestructura crítica](https://www.darkreading.com/cyberattacks-data-breaches/busysnake-infostealer-critical-infrastructure-networks)
- [SecurityWeek — Armored Likho APT](https://www.securityweek.com/armored-likho-apt-targeting-government-electric-power-entities/)
- [Malwarebytes — Verified X ad spreads Mac malware / ConsentFix](https://www.malwarebytes.com/blog/news/2026/07/verified-x-ad-spreads-mac-malware-while-consentfix-steals-microsoft-accounts)
- [Malwarebytes — 24 mil millones de registros expuestos](https://www.malwarebytes.com/blog/news/2026/06/24-billion-stolen-records-found-in-giant-data-dump-check-if-youre-affected)
- [PrivacyGuides — Data Breach Roundup (26 jun – 2 jul 2026)](https://www.privacyguides.org/news/2026/07/03/data-breach-roundup-june-26-july-2-2026/)
- [CM-Alliance — June 2026 Biggest Cyber Attacks & Ransomware](https://www.cm-alliance.com/cybersecurity-blog/june-2026-biggest-cyber-attacks-data-breaches-ransomware-attacks)
- [Integrity360 — Cyber News Roundup July 3rd 2026](https://www.integrity360.com/cyber-news-roundup-july-3rd-2026)
- [ENISA — DORA is now alive and kicking](https://www.enisa.europa.eu/news/eu-financial-entities-cybersecurity-upgrade-dora-is-now-alive-and-kicking)
- [Reed Smith — EU cybersecurity regulatory update for 2026](https://www.reedsmith.com/our-insights/blogs/viewpoints/102mnj2/eu-cybersecurity-regulatory-update-for-2026-and-beyond/)
- [AlteredSecurity — CRTE Updated for 2026 (TTPs AD)](https://www.alteredsecurity.com/post/crte-red-team-certification-update)

---

*Informe generado automáticamente el 7 de julio de 2026. Nota: la explotación en curso y las atribuciones pueden evolucionar; verificar los avisos oficiales (CISA, ENISA, vendors) antes de tomar decisiones operativas.*
