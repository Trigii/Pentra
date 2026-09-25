# 💡 Ideas de Negocio del Día — 20 de septiembre de 2026

> Rotación de hoy: **1 micro-SaaS de infraestructura**, **1 API/dev-tool**, **1 servicio de consultoría nicho**, **1 infoproducto en español**, **1 herramienta interna de firma de pentesting**.
> Se han evitado deliberadamente los nichos ya cubiertos en ediciones anteriores (escáner MCP, AI Act red team as-a-service, sprints NIS2, generador de informes, auditoría NHI, newsletters de pago sobre IA).

---

## 1. RevokeKit — La capa de *revocación* de secretos (no de detección)

**Concepto.** Todo el mercado vende *detección* de secretos filtrados. Nadie vende el paso siguiente: revocar y rotar la credencial concreta, en el proveedor concreto, en minutos. RevokeKit es una biblioteca viva de runbooks de revocación (200+ proveedores: AWS, OpenAI, Anthropic, Stripe, Supabase, Twilio, GitHub PAT, Slack, HubSpot…) con CLI y API que ejecuta la rotación cuando el proveedor tiene endpoint para ello.

**Problema que resuelve.** GitGuardian reprobó en enero de 2026 secretos que ya estaban confirmados como válidos en 2022: el **64% seguía sin revocar** — cuatro años de exposición. Cuando unas credenciales AWS aparecen en público, el atacante intenta usarlas en **17 minutos de media**, mientras que casi una cuarta parte de las organizaciones tarda **más de 24 horas** en rotarlas. El cuello de botella no es saber que se ha filtrado: es que el ingeniero de guardia no sabe dónde está el botón de revocar en el panel de un SaaS que usa una vez al año, y el proveedor de detección le deja solo justo ahí.

**Target.** Equipos de plataforma/DevSecOps de 10–150 ingenieros que ya pagan GitGuardian, TruffleHog Enterprise o el secret scanning de GitHub y tienen una cola de alertas sin cerrar. Comprador: Head of Platform o Security Engineer, no el CISO. También MSSPs que quieren ofrecer "remediación" y hoy solo entregan hallazgos.

**Modelo de negocio.** Open-core. Los runbooks en Markdown son públicos y gratis (motor SEO brutal: *"how to revoke a leaked \<proveedor\> key"* es una long tail enorme). El producto de pago es la ejecución automatizada + integración con el escáner que ya usan + registro de auditoría probatorio: 149 $/mes (hasta 25 devs), 499 $/mes (equipo), 1.500 $/mes (MSSP, multi-tenant).

**Ventaja de Tris.** Es el perfil exacto: DevSecOps que entiende IAM en cloud + pentester que sabe qué hace un atacante con esa clave en los primeros 17 minutos. Los runbooks de revocación solo los escribe bien alguien que ha abusado de esas credenciales en un ejercicio real. Además, el contenido lo puede ir produciendo desde su trabajo diario sin fricción.

**Dificultad.** 🟡 Media — el contenido es trabajo lineal y aburrido (ventaja: es un foso, nadie lo quiere hacer); la parte técnica de automatización es sencilla por proveedor.

**Ingresos estimados.** 1.500–8.000 €/mes en 12–18 meses. Techo bajo como SaaS, techo alto como activo de contenido adquirible.

**Validación en 3 semanas.** Publicar 30 runbooks de los proveedores más filtrados (claves de IA: +81% en 2025, 1,275 M de credenciales expuestas). Si ese sitio hace 2.000 visitas orgánicas/mes en el mes 2, hay negocio. Si no, no se ha perdido más que un fin de semana.

---

## 2. AgentRegress — Tests de inyección de prompt en el CI, no auditorías puntuales

**Concepto.** Una GitHub Action + API que, en cada pull request, lanza un corpus versionado de ataques (inyección de prompt directa e indirecta, abuso de herramientas, exfiltración vía tool-calling, escalada de permisos entre agentes) contra el endpoint del agente y **rompe el build** si una defensa que antes aguantaba ahora falla. No es un escáner de seguridad: es una suite de regresión.

**Problema que resuelve.** Los equipos que construyen agentes cambian el prompt de sistema, el modelo o un tool wrapper cinco veces por semana, y cada cambio puede reabrir un bypass que ya se había parcheado — sin que nadie se entere hasta producción. Las descargas de frameworks de agentes superan a las de herramientas de seguridad **83 a 1** en PyPI, y esa brecha creció un 41% entre enero y mayo de 2026. Hoy la única respuesta del mercado es una auditoría manual de 4 semanas que queda obsoleta en el siguiente deploy.

**Target.** Startups de producto con IA, serie seed a serie B, que venden a empresa y ya han recibido su primer cuestionario de seguridad de un cliente grande. El comprador es el CTO o el ingeniero que lidera la plataforma de IA — presupuesto de herramientas de desarrollo, no de seguridad. Ciclo de venta de días, no de trimestres.

**Modelo de negocio.** PLG puro: gratis en repos públicos, 79 $/mes por repo privado, 299 $/mes con corpus de ataques actualizado semanalmente y panel de tendencias, 999 $/mes con corpus específico del sector del cliente. El corpus actualizado es la suscripción real; el runner es commodity.

**Ventaja de Tris.** Ya tiene el prototipo del **AI Red Team Agent** (bucle ReAct, orquestación LLM, wrappers de herramientas, backends Ollama/Groq/Anthropic). Esto es el envoltorio comercial correcto de ese motor: en lugar de vender "un agente que hace red team" (categoría confusa, comprador indefinido, ciclo largo), vende "tests que fallan el build" — un formato que todo ingeniero ya entiende y ya paga. El mismo código, un empaquetado con 10x menos fricción de venta.

**Dificultad.** 🟡 Media — el runner es un fin de semana; el valor está en mantener el corpus vivo, que es precisamente el trabajo que a él le divierte.

**Ingresos estimados.** 500–2.000 €/mes en 6 meses; 5.000–20.000 €/mes a 18 meses si el corpus se convierte en referencia.

**Validación en 3 semanas.** Publicar el corpus de ataques como repo abierto y medir estrellas + issues. Si 15 equipos lo integran a mano, el producto de pago se vende solo.

---

## 3. Due Diligence de seguridad para micro-adquisiciones de SaaS

**Concepto.** Auditoría técnica de seguridad de 3 días, precio cerrado, para compradores de SaaS pequeños (200 k€ – 3 M€) en Acquire.com, Flippa, search funds y micro-PE. Entregable: un informe de 12 páginas que responde a una sola pregunta — *¿qué pasivo de seguridad estoy comprando y cuánto cuesta arreglarlo?*

**Problema que resuelve.** Este mercado hace due diligence financiera y legal a fondo, y **cero** due diligence de seguridad. El comprador hereda secretos hardcodeados en el repo, un panel de administración sin MFA, backups que nadie ha restaurado nunca, dependencias de 2021, datos personales de clientes de la UE sin base legal clara, y un único desarrollador que se marcha en 30 días con todos los accesos. Descubrirlo después de firmar convierte una compra rentable en una crisis. Es un dolor de **dinero**, no de cumplimiento — por eso se paga rápido y sin comité.

**Target.** Compradores individuales y micro-fondos que cierran entre 2 y 10 operaciones al año. Nicho muy concreto y alcanzable: los intermediarios (brokers de Acquire, asesores de M&A de bootstrappers) son 20 personas en todo el mundo y concentran el flujo de operaciones. Se llega a ellos con dos cafés.

**Modelo de negocio.** Servicio productizado: 2.500 € por DD estándar (3 días), 4.500 € con revisión de código e infraestructura cloud. Upsell natural y recurrente: el comprador se convierte en *dueño* de un SaaS con deuda de seguridad → retainer de remediación a 800–1.500 €/mes. La verdadera palanca es la relación con los brokers: un acuerdo de referencia con dos de ellos genera flujo constante sin marketing.

**Ventaja de Tris.** Perfil híbrido raro y justo el que hace falta: pentester (encuentra los agujeros) + DevSecOps (sabe estimar el coste real de arreglarlos, que es lo que el comprador necesita para negociar el precio). Un pentester puro entrega hallazgos; un auditor GRC entrega políticas. Aquí el entregable es una **cifra de remediación** defendible en una negociación, y eso exige las dos mitades.

**Dificultad.** 🟢 Fácil de arrancar — no hay producto que construir, el proceso es repetible desde la primera operación y se puede hacer en fines de semana.

**Ingresos estimados.** 2.000–6.000 €/mes con 1–2 operaciones mensuales; escalable a 10 k€+/mes solo si se sistematiza y se delega.

**⚠️ Nota de conflicto.** Con la incorporación a One eSecurity el 29 de septiembre, cualquier idea de consultoría hay que contrastarla contra la cláusula de exclusividad y no competencia del contrato **antes** de facturar un euro. Esta idea es la menos conflictiva de las de servicio (compradores individuales de micro-SaaS no son clientes potenciales de una firma de seguridad ofensiva corporativa), pero la conversación con Carmen o con el contrato en la mano hay que tenerla igual.

---

## 4. "Del laboratorio al informe" — Infoproducto en español sobre el entregable, no sobre el exploit

**Concepto.** Curso corto (4–5 h) + pack de plantillas sobre lo único que el cliente compra realmente en un pentest: **el informe**. Cómo escribir un hallazgo que sobreviva a la revisión del cliente, cómo puntuar CVSS sin que te lo discutan, cómo redactar un resumen ejecutivo que lea un director no técnico, cómo encadenar hallazgos de bajo riesgo en una narrativa de impacto real, cómo defender los hallazgos en la reunión de cierre.

**Problema que resuelve.** Existe una industria entera enseñando a explotar (HTB, TryHackMe, OSCP, cientos de canales de YouTube) y prácticamente nadie enseñando a entregar. El resultado: juniors que rootean la máquina y suspenden la prueba técnica por el informe; freelancers que hacen un trabajo excelente y pierden el cliente porque el PDF parece un volcado de Nessus; y todo ello en español, donde el vacío es aún mayor que en inglés. Es la habilidad con mayor retorno por hora de estudio de toda la profesión y nadie la vende.

**Target.** Dos segmentos con el mismo producto: (a) juniors en España y LatAm preparando pruebas técnicas y primeros meses de trabajo — volumen; (b) pentesters freelance que facturan directamente y necesitan que el entregable justifique su tarifa — disposición a pagar. Mercado hispanohablante, poco atendido, con menos competencia y menor coste de adquisición que el anglosajón.

**Modelo de negocio.** Escalera clásica: plantilla de informe gratuita (imán de emails) → pack de plantillas + biblioteca de 40 hallazgos redactados y reutilizables, 47 € → curso completo con revisión de un informe real, 197 € → revisión 1:1 de informes, 150 €/informe (limitada, sirve de investigación de mercado y de prueba social). Coste marginal cero, sin soporte, sin infraestructura.

**Ventaja de Tris.** Credibilidad recién ganada y verificable: está escribiendo **ahora mismo** un informe de pentest para una evaluación técnica real (laboratorio del portal de reclamaciones de seguros, entrega el lunes 21) y se incorpora a un equipo ofensivo el 29 de septiembre. El contenido se genera como subproducto del trabajo diario — cada informe real que escriba durante el próximo año es material del curso, anonimizado. Es la posición ideal: lo bastante junior para recordar exactamente dónde duele, lo bastante bueno para que le contraten por ello.

**Dificultad.** 🟢 Fácil — es la idea con menor riesgo técnico y menor conflicto laboral de toda la lista (formación genérica, sin clientes, sin competencia con el empleador).

**Ingresos estimados.** 300–1.500 €/mes de forma pasiva tras 6 meses de contenido constante; 3.000 €/mes+ si la lista de correo crece por encima de 5.000 suscriptores.

**Validación en 2 semanas.** Publicar la plantilla de informe gratis en LinkedIn y en r/hacking_es / comunidades hispanas de ciberseguridad. Si consigue 300 descargas y 15 respuestas del tipo *"¿tienes algo más?"*, hay producto.

---

## 5. ScopeSentinel — Validación de alcance antes de disparar el primer paquete

**Concepto.** API + interfaz web que toma la lista de alcance que envía el cliente (dominios, rangos IP, apps) y verifica automáticamente, antes de que empiece el proyecto, que **cada activo pertenece realmente al cliente**: RDAP/WHOIS, propiedad de ASN, transparencia de certificados, detección de rangos compartidos de cloud, CDN y terceros (Cloudflare, Shopify, hosting compartido, SaaS de terceros). Salida: un documento de alcance limpio, con cada activo marcado como confirmado / dudoso / de terceros, listo para adjuntar a la carta de autorización.

**Problema que resuelve.** Toda firma de pentesting empieza cada proyecto con una lista de alcance proporcionada por el cliente que está mal: contiene IPs que ya no son suyas, activos alojados por terceros que no han autorizado nada, y rangos compartidos donde tocar es ilegal. Las consecuencias son de dos tipos y ambas caras: días de trabajo facturable quemados en activos muertos, y riesgo legal real si se ataca infraestructura de un tercero. Hoy esto se resuelve con un analista haciendo `whois` a mano durante medio día por proyecto, o directamente no se resuelve y se confía en la carta de autorización.

**Target.** Firmas de pentesting pequeñas y medianas (5–50 consultores) y pentesters freelance con varios clientes. Es un dolor **interno y operativo**, no de cumplimiento: se vende al director técnico o al líder del equipo ofensivo, que decide sin comité de compras. Mercado pequeño pero de altísima conversión, porque todos tienen exactamente el mismo problema y nadie ha montado la herramienta.

**Modelo de negocio.** SaaS por asiento o por proyecto: 99 €/mes hasta 5 proyectos, 349 €/mes ilimitado por firma, o 29 € por validación suelta para freelancers. Complemento evidente: generación de la carta de autorización con el anexo de alcance verificado. Precio anclado en horas ahorradas — media jornada de analista por proyecto se paga solo.

**Ventaja de Tris.** A partir del 29 de septiembre va a vivir este problema **en cada proyecto**, en un equipo ofensivo de nueva creación (hoy una sola persona con 2 años) que todavía no tiene procesos internos consolidados. Eso significa dos cosas: (a) acceso directo al dolor y a la iteración real, y (b) los compradores son sus propios colegas de profesión, a los que puede entrevistar sin fingir. Es el caso de libro de *scratch your own itch* aplicado a un nicho de herramientas internas que las grandes plataformas ignoran por pequeño.

**Dificultad.** 🟡 Media — técnicamente es agregación de fuentes públicas (RDAP, CT logs, rangos de cloud publicados); la dificultad real está en la precisión de la clasificación, que es donde está el valor.

**Ingresos estimados.** 1.000–5.000 €/mes con 15–30 firmas. Nicho pequeño y bien delimitado, pero con retención altísima: una vez que está en el proceso de arranque de proyecto, no se quita.

**⚠️ Nota de conflicto.** Construir herramientas basadas en procesos observados en el empleador es zona gris contractual. Si esta idea avanza, hay que desarrollarla con fuentes públicas exclusivamente, fuera del horario y el equipo de trabajo, y sin usar datos, clientes ni código de la empresa. Es perfectamente viable, pero hay que hacerlo limpio desde el primer commit.

---

## 🧭 Reflexión del día

**La ventana de la "brecha de remediación" está abierta y se está cerrando.**

Los datos de este año apuntan todos en la misma dirección: el mercado de seguridad ha resuelto la *detección* y ha fracasado por completo en la *acción*. El 64% de los secretos confirmados como válidos en 2022 seguía sin revocar en enero de 2026. Las organizaciones testean solo el 32% de su superficie de ataque, dejando el 68% sin tocar. El 92% dice que sus herramientas de IAM no pueden gestionar identidades de agentes. Los frameworks de agentes se descargan 83 veces más que las herramientas de seguridad, y esa brecha creció un 41% en cuatro meses.

La lectura estratégica: durante una década el dinero estuvo en *encontrar* problemas — escáneres, gestión de superficie de ataque, plataformas de detección. Ese espacio está saturado y consolidándose. El dinero de los próximos tres años está en **cerrar el bucle**: revocar, rotar, verificar, probar en cada deploy, demostrar con evidencia. Tres de las cinco ideas de hoy (RevokeKit, AgentRegress, ScopeSentinel) son deliberadamente herramientas de *acción*, no de *hallazgo* — y ninguna requiere competir con un producto financiado por venture capital, porque los fondos siguen financiando detección.

**Y una nota sobre el momento personal.** Esta semana el contexto cambia: incorporación a un equipo ofensivo el 29 de septiembre, remoto, en un equipo nuevo. Eso reordena las prioridades de esta lista de forma bastante clara:

- **Las primeras 6–8 semanas no son para construir SaaS.** Son para aprender el oficio a tiempo completo y para tener la conversación de exclusividad/no competencia con el contrato delante.
- **La idea 4 (infoproducto) es la única que se puede empezar esta misma semana sin fricción alguna** — no compite con el empleador, no requiere clientes, y el informe que se entrega mañana ya es la primera pieza de contenido.
- **Las ideas 1, 2 y 5 son para el mes 2 en adelante**, cuando haya datos reales del día a día para decidir cuál duele de verdad. Concretamente: la 5 se valida sola en los primeros cinco proyectos — si en el arranque de cada uno se pierde media jornada limpiando el alcance, hay producto; si no, se descarta sin haber escrito una línea de código.
- **La idea 3 (due diligence) es la de mejor relación esfuerzo/ingreso**, pero es la que más depende de lo que diga el contrato.

El error clásico en este momento vital es arrancar un SaaS la misma semana que se empieza un trabajo nuevo y exigente. El movimiento correcto es empezar por lo asíncrono y sin dependencias — contenido — y dejar que los primeros meses de trabajo real seleccionen la herramienta que merece construirse.

---

### Fuentes

- [MCP Security Statistics 2026: CVEs, Vulnerabilities & Breach Data — Practical DevSecOps](https://www.practical-devsecops.com/mcp-security-statistics-2026-report/)
- [GitGuardian Report: Non-Human Identity Sprawl as Primary Security Risk 2026](https://nhimg.org/nhi-news/non-human-identity-sprawl-enterprise-security-2026)
- [The Agent Identity Problem: Non-Human Identities Outnumber Humans 45 to 1 — Security Boulevard](https://securityboulevard.com/2026/07/the-agent-identity-problem-non-human-identities-outnumber-humans-45-to-1-and-ai-agents-are-making-it-worse/)
- [Agentic AI identity sprawl is outpacing enterprise governance in 2026 — NHI Mgmt Group](https://nhimg.org/articles/agentic-ai-identity-sprawl-is-outpacing-enterprise-governance-in-2026/)
- [EU AI Act red teaming: what you must do before 2 August 2026 — Red Team Partners](https://redteampartner.com/field-notes/eu-ai-act-red-teaming-guide/)
- [NIS2 And DORA Readiness: What European Businesses Must Do Before October 2026 — Cyber Threat Defense](https://ctdefense.com/nis2-dora-readiness-october-2026/)
- [Best Penetration Testing Tools in 2026 — ComplyJet](https://www.complyjet.com/blog/best-penetration-testing-tools)
- [The State of Paid Newsletters 2026 — beehiiv](https://www.beehiiv.com/blog/the-state-of-paid-newsletters-2026)
- [Solopreneur Income Report: Real Revenue Benchmarks 2026 — Goal Group](https://goal-group.com/articles/tools-comparisons/solopreneur-income-report-real-revenue-benchmarks-/)
