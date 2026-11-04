## 💡 Ideas de Negocio del Día — 24 de octubre de 2026

*Mix de hoy: 2 herramientas/API, 1 servicio nicho, 1 infoproducto, 1 contenido de pago. Evitadas las ideas ya publicadas (RevokeKit, regresión de agentes, verificación de scope, informes de pentest, PSIRT-as-a-service, CBOM, NHI audit, etc.).*

---

### 1. `mcp-lint` — Linter de seguridad para servidores MCP
1. **Concepto:** CLI open-source + GitHub Action que analiza un servidor MCP (herramientas expuestas, permisos, validación de entradas, secretos en config, descripciones de tools susceptibles de *tool poisoning*) y genera un informe con severidad y fixes. Versión de pago: monitorización continua + insignia "verificado".
2. **Problema:** Las empresas están desplegando servidores MCP internos y de terceros a toda velocidad, sin revisión de seguridad. Los equipos de plataforma no saben qué mirar y los equipos de seguridad no tienen tooling específico.
3. **Target:** Equipos de plataforma/DevEx de empresas de 50–1.000 personas que ya usan agentes de IA; desarrolladores que publican servidores MCP en registries.
4. **Modelo:** Open-core. CLI gratis (distribución y credibilidad) → plan Team 49–199 €/mes (CI, histórico, reglas custom) → auditorías manuales premium (2–4 k€) como upsell.
5. **Ventaja de Tris:** Mentalidad ofensiva: sabe qué reglas escribir porque sabe cómo se explota (SSRF, inyección de comandos, confused deputy, exfiltración vía tools). Ya trabaja en agentes de IA para red team.
6. **Dificultad:** 🟡 Media (MVP con 15–20 reglas en 3–4 semanas).
7. **Ingresos:** 1–6 k€/mes a 6–9 meses si el repo gana tracción; techo mayor con auditorías.

*Validación en semanas:* publicar el repo + un post con "hallazgos en los 50 servidores MCP más populares" (divulgación responsable). Mide estrellas y emails en lista de espera.

---

### 2. `ControlMap` — API que mapea hallazgos de pentest a controles NIS2 / DORA / ISO 27001
1. **Concepto:** API/plugin que recibe un hallazgo (CWE/OWASP/título + descripción) y devuelve los controles afectados en NIS2 (art. 21), DORA, ISO 27001 Anexo A y ENS, con redacción lista para el informe ("incumple el control X porque…").
2. **Problema:** Los clientes europeos piden cada vez más que el informe del pentest hable el idioma del auditor de cumplimiento. Los pentesters pierden horas mapeando a mano y los mapeos suelen ser inconsistentes.
3. **Target:** Boutiques de pentest y consultoras de 3–30 personas en España/UE; freelancers de pentesting.
4. **Modelo:** SaaS/API por asiento (29–99 €/usuario/mes) + plugins para Dradis, PwnDoc, SysReptor y exportación a Word/Markdown.
5. **Ventaja de Tris:** Pentester en activo en una consultora: conoce el dolor exacto del flujo de informes, y su paso por BBVA le da contexto regulatorio financiero (DORA).
6. **Dificultad:** 🟡 Media (la parte difícil es la curación de los mapeos, no el código).
7. **Ingresos:** 1–4 k€/mes con 30–60 usuarios; escalable vendiendo licencias de equipo.

*Validación:* generar 20 hallazgos mapeados de ejemplo y enseñarlos a 10 consultoras por LinkedIn; pedir preventa.

---

### 3. Pentest "CRA-Ready" para fabricantes pequeños de producto con software
1. **Concepto:** Servicio de precio cerrado (3–5 días) que prueba un producto (dispositivo IoT, app, firmware, API) y entrega un informe **estructurado por los requisitos esenciales del Anexo I del Cyber Resilience Act**: qué cumple, qué no y evidencia técnica para el expediente técnico.
2. **Problema:** Con el CRA ya en fase de obligaciones de reporte y la conformidad plena acercándose, los fabricantes pequeños (sobre todo de hardware conectado) no saben cómo demostrar "seguridad por diseño" y no quieren un pentest genérico que luego no encaje en su expediente.
3. **Target:** Fabricantes de producto con elementos digitales, 10–200 empleados, en UE (domótica, industrial ligero, wearables, SaaS embebido).
4. **Modelo:** Servicio por proyecto 3–8 k€ + retainer anual de retest (1–2 k€/trimestre).
5. **Ventaja de Tris:** OSCP + trabajo diario como pentester + perfil de ingeniero informático. Diferenciador: hablar el idioma del cumplimiento sin ser consultor "de PowerPoint".
6. **Dificultad:** 🟡 Media (conviene afinar la plantilla de Anexo I con asesoría legal; cuidado con el pluriempleo/cláusulas de exclusividad de su contrato actual).
7. **Ingresos:** 3–10 k€/mes con 1–2 proyectos/mes.

*Validación:* una landing + 30 mensajes directos a fabricantes en Catalunya; objetivo: 2 llamadas de descubrimiento en 3 semanas.

---

### 4. Curso "Mac en la empresa: atacar flotas macOS" (en español)
1. **Concepto:** Curso práctico (8–10 h) + laboratorio sobre pentest/red team en entornos macOS corporativos: abuso de MDM (Jamf/Intune), TCC, Keychain, persistencia, movimiento lateral Mac→AD, evasión de EDR en macOS.
2. **Problema:** Casi todo el contenido ofensivo está centrado en Windows/AD. Los pentesters que se encuentran flotas Mac en startups y empresas tech no tienen un camino estructurado, y menos en español.
3. **Target:** Pentesters y red teamers junior-mid de habla hispana, y equipos blue que quieren entender el ataque.
4. **Modelo:** Infoproducto 149–249 € (Gumroad/Teachable) + edición con cohorte en directo 399 € + licencias para equipos.
5. **Ventaja de Tris:** Ya trabaja en una línea de macOS red team y prepara OSEP; puede documentar mientras investiga, convirtiendo su estudio en producto.
6. **Dificultad:** 🟡 Media (el laboratorio con Macs es el reto: virtualización limitada; se puede basar en Macs físicos + guías reproducibles).
7. **Ingresos:** 2–8 k€ por lanzamiento (40–50 alumnos), recurrente con actualizaciones.

*Validación:* 3 hilos/posts gratuitos sobre un truco concreto de macOS; medir lista de espera antes de grabar nada.

---

### 5. Newsletter de pago "Red Team Notes ES": un write-up técnico al mes + digest semanal
1. **Concepto:** Newsletter semanal gratuita (digest de técnicas, CVEs y herramientas ofensivas, en español) con una capa de pago: un análisis técnico profundo mensual con PoC y checklist reutilizable.
2. **Problema:** El contenido ofensivo de calidad está en inglés y disperso; la comunidad hispana busca curación fiable de nivel intermedio-avanzado, no titulares.
3. **Target:** Pentesters, estudiantes de OSCP/OSEP y equipos de seguridad hispanohablantes.
4. **Modelo:** Suscripción 7–12 €/mes o 70–100 €/año + patrocinios (herramientas, formaciones, bolsas de empleo) cuando la lista supere los 2–3 k suscriptores.
5. **Ventaja de Tris:** Mantiene el vault Pentra y ya genera informes semanales; el contenido es una reutilización directa de su flujo de trabajo actual.
6. **Dificultad:** 🟢 Fácil (el coste es constancia, no tecnología).
7. **Ingresos:** 300–1.500 €/mes al año con 1–2 k suscriptores gratuitos y 5–8 % de conversión; patrocinios suman 200–800 €/edición después.

*Validación:* publicar 4 números gratuitos en Substack/Beehiiv; si llegas a 300 suscriptores en 6 semanas, abre la capa de pago.

---

**Reflexión del día:** El patrón común de hoy es que la regulación europea (CRA, NIS2, DORA) y la fiebre de los agentes de IA están creando demanda de *traducción*: alguien que convierta un hallazgo técnico en evidencia de cumplimiento, o un servidor MCP desplegado a toda prisa en un riesgo medible. Quien combine criterio ofensivo con lenguaje regulatorio ocupa un hueco que las plataformas grandes aún no cubren bien, y puede validarse con landing, outreach y un repo público en pocas semanas.

---
### Fuentes
- [Regulatory wave 2026: NIS2, DORA, AI Act & CRA](https://www.advisori.de/en/blog/regulatory-wave-2026-nis2-dora-ai-act-cra-what-companies-need-to-do-now)
- [CRA September 2026: Vulnerability Reporting Starts](https://www.advisori.de/en/blog/cra-september-2026-vulnerability-reporting-manufacturers-check)
- [EU CRA 2026 Readiness](https://sam-solutions.com/blog/eu-cra-2026-readiness)
- [Your Enterprise Deployed MCP Servers. Has Anyone Audited Them?](https://dev.to/easterndev/your-enterprise-deployed-mcp-servers-has-anyone-audited-them-26l1)
- [MCP Security: 30 CVEs in 60 Days](https://www.noqta.tn/en/blog/securite-serveurs-mcp-vulnerabilites-protection-guide-2026)
- [Synack: AI pentesting coverage gap 2026](https://www.synack.com/press-releases/ai-pentesting-coverage-gap-2026/)
