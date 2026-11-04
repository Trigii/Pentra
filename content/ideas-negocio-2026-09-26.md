# 💡 Ideas de Negocio del Día — 26 de septiembre de 2026

> Rotación de hoy: **1 SaaS de seguridad**, **1 servicio de consultoría/simulacro nicho**, **1 infoproducto en español**, **1 API/script interno de pentesting**.
> Se evitan deliberadamente los nichos ya cubiertos en ediciones anteriores (escáneres MCP, AI Act red team as-a-service, sprints NIS2/DORA, generadores de informes, auditoría NHI/CBOM, newsletters de pago sobre IA, DD de M&A, plantillas de informe, validación de alcance, bug bounty de nicho).

---

## 1. PageTrap — Escáner de "secuestro de navegadores agénticos" para sitios con contenido de usuario

**Concepto.** API/SaaS que rastrea el contenido público de un sitio web (reseñas, comentarios, campos de perfil, PDFs subidos, metadatos, invitaciones de calendario embebidas) buscando payloads de inyección de prompt ocultos —texto blanco sobre blanco, atributos `alt`, comentarios HTML, Unicode invisible— diseñados para secuestrar navegadores agénticos (Comet, Atlas y similares) que visitan la página en nombre de un usuario o de un agente de compra automatizado. Alerta en cuanto aparece contenido nuevo sospechoso, con puntuación de riesgo y PoC reproducible.

**Problema que resuelve.** Zenity Labs demostró en 2025-2026 vulnerabilidades reales en Comet: un simple contenido de calendario o página web controlado por el atacante puede hacer que el agente interprete instrucciones ajenas como órdenes del usuario, accediendo a archivos locales, exfiltrando datos o tomando el control del gestor de contraseñas, todo mientras muestra una salida inocente en pantalla. OpenAI ha admitido públicamente que este tipo de fallo es "improbable que se resuelva del todo". El problema de incentivos es que el ataque golpea al *visitante* (el usuario del navegador agéntico), no al operador del sitio — así que hoy nadie escanea su propio contenido de usuario en busca de esto, exactamente igual que nadie escaneaba XSS almacenado antes de que fuera obligatorio.

**Target.** E-commerce medianos/grandes con contenido generado por usuario (reseñas, foros, marketplaces) y SaaS B2B que empiezan a recibir tráfico de agentes de compra automatizados. Comprador: responsable de Trust & Safety, AppSec o Producto — no es un comprador de seguridad clásico todavía, lo que alarga el ciclo de venta pero también significa cero competencia establecida.

**Modelo de negocio.** Escaneo puntual gratuito (imán de leads) → monitorización continua 199–499 $/mes por dominio según volumen de contenido de usuario → API de integración en el pipeline de moderación desde 999 $/mes para marketplaces grandes.

**Ventaja de Tris.** Es la intersección exacta de dos habilidades que ya tiene documentadas en su propia base de conocimiento: cómo esconder un payload en una página HTTP (HTML injection, XSS, manipulación de metadatos — su día a día en explotación manual web) y cómo un agente LLM interpreta esa página como contexto de confianza. Nadie que venga solo del lado "AI safety" sabe ocultar el payload tan bien como un pentester web; nadie que venga solo del lado pentest clásico sabe qué hace un agente con ese HTML.

**Dificultad.** 🟡 Media — el motor de detección de patrones conocidos es un fin de semana; mantenerlo al día con las técnicas de evasión (Unicode, esteganografía en imágenes, PDFs) es el trabajo continuo y el verdadero foso.

**Potencial de ingresos estimado.** 800–3.000 €/mes en 6–9 meses. Categoría de compra todavía sin nombre en el mercado, así que el arranque es más lento de lo habitual, pero también sin competidor directo.

**Validación en 2-3 semanas.** Escanear gratis 20-30 sitios de e-commerce y foros españoles conocidos y publicar los hallazgos (con permiso o responsablemente anonimizados) como informe de investigación técnica. Genera cobertura en medios especializados y leads entrantes sin gastar en ads.

---

## 2. Simulacro de fraude por voz clonada (vishing con IA) para equipos financieros

**Concepto.** Ejercicio de ingeniería social dirigido específicamente a fraude de "director financiero por voz clonada": con alcance y consentimiento firmados, se clona la voz de un directivo real a partir de un vídeo público (LinkedIn, webinar, rueda de prensa) y se ejecuta una llamada de fraude simulada contra el equipo financiero para medir si activan el protocolo de verificación fuera de banda. Cierra con informe + taller de 1 hora sobre cómo verificar antes de autorizar una transferencia.

**Problema que resuelve.** Los datos de 2025-2026 son contundentes: los intentos de fraude por deepfake han crecido un 1.300% interanual, los ataques de vishing con voz sintética un 1.633% solo entre el Q4 2024 y el Q1 2025, y clonar una voz de forma convincente hoy requiere solo 3 segundos de audio — la detección humana de un deepfake de calidad cae al 24,5% de aciertos. Solo el 32% de las organizaciones ha desplegado alguna herramienta de detección, y las pérdidas medias por incidente empresarial rondan los 680.000 $. El phishing simulado por email ya es un servicio estándar de mercado (KnowBe4 y similares); el simulacro de vishing con voz clonada por IA sigue siendo un hueco casi vacío, sobre todo en pymes españolas que nunca lo han probado.

**Target.** Pymes de 20-300 empleados con departamento financiero centralizado (una o dos personas con capacidad real de aprobar transferencias), idealmente empresas que ya han sufrido un intento de "fraude del CEO" clásico por email o conocen un caso cercano en su sector.

**Modelo de negocio.** Paquete cerrado de 1.500–3.000 € (simulacro + informe + taller), con retainer trimestral de 500 € para repetir el ejercicio con variantes nuevas del pretexto.

**Ventaja de Tris.** Ya tiene documentado el lado técnico de client-side attacks y pretexting; clonar una voz con las herramientas actuales es trivial técnicamente, así que el valor real está en diseñar el pretexto creíble y medir el protocolo — trabajo puro de red teamer, no de proveedor de herramienta de IA.

**Dificultad.** 🟢 Fácil montar el primer caso con una plantilla de pretexto reutilizable; 🟡 media escalarlo sin que cada cliente exija trabajo manual a medida.

**Potencial de ingresos estimado.** 1.500–4.500 €/mes con 2-3 clientes al mes.

**⚠️ Nota de conflicto — la más importante de hoy.** Esto es un servicio de red team social-engineering dirigido a clientes externos, exactamente el tipo de trabajo que hace una firma de seguridad ofensiva como la que se incorpora el 29 de septiembre. Es un conflicto más directo que el de la due diligence de M&A de la edición anterior: aquí sí se compite de lleno con el core business del futuro empleador. Con la incorporación a solo tres días vista, esta idea no se toca sin tener antes la cláusula de exclusividad delante y, si aplica, la conversación correspondiente — y probablemente lo más sensato sea aparcarla del todo mientras dure esa relación laboral, más que buscar cómo encajarla.

---

## 3. "De IT a OT" — Curso en español de fundamentos de pentesting industrial (ICS/SCADA) para pentesters IT

**Concepto.** Curso corto (6-8 h) + laboratorio simulado (Modbus/OPC-UA en contenedores, sin tocar hardware real) que enseña a un pentester de IT tradicional lo mínimo imprescindible para no ser peligroso en su primer proyecto OT: qué NO escanear ni tocar y por qué (un escaneo normal puede parar una línea de producción), terminología básica (modelo Purdue, PLC, HMI, SCADA), y cómo se estructura una auditoría OT frente a una de IT.

**Problema que resuelve.** NIS2 entra en su "acantilado de cumplimiento" en octubre de 2026 y obliga directamente a energía, agua potable, transporte, infraestructura financiera e industria manufacturera — sectores donde la seguridad depende de sistemas OT/ICS, un área que la propia normativa apenas detalla en controles específicos, dejando un vacío práctico enorme. Va a hacer falta mucha más gente que sepa auditar estos entornos de la que existe hoy, y el riesgo de tumbar un proceso industrial por desconocimiento hace que casi nadie se atreva a dar el salto sin formación previa. En español, la oferta formativa sobre esto es prácticamente inexistente: todo lo serio está en inglés, es caro y está pensado para el mercado norteamericano.

**Target.** Pentesters y consultores IT de España y LatAm con 1-4 años de experiencia que ven venir la demanda derivada de NIS2 en la cartera de clientes de su empresa y quieren especializarse antes de que el hueco se llene.

**Modelo de negocio.** Curso grabado + acceso al laboratorio, 147–247 €, con una edición en directo trimestral de grupo reducido (20 plazas) a 397 € que incluye revisión de un caso propio del alumno.

**Ventaja de Tris.** Su formación técnica en redes, protocolos y post-explotación (ya documentada extensamente en su propia base de conocimiento de pentesting clásico) es exactamente la base desde la que se explica bien el salto a OT a alguien que parte de cero en ese mundo — y si su nuevo puesto acaba tocando este terreno, puede ir documentando el aprendizaje genérico (nunca material ni datos de cliente) como materia prima del curso.

**Dificultad.** 🟢 Fácil–🟡 media — el contenido teórico es accesible; montar el laboratorio simulado con contenedores lleva más tiempo que un curso puramente teórico, pero es un trabajo de una sola vez.

**Potencial de ingresos estimado.** 500–2.000 €/mes de forma pasiva a partir del sexto mes; con recorrido para convertirse en referencia en español si se publica antes de que el grueso del mercado reaccione a la fecha límite de NIS2.

**Validación en 2 semanas.** Publicar gratis un mini-PDF tipo "5 errores que tumban una planta al hacer tu primer pentest OT sin saberlo" y medir descargas y respuestas en LinkedIn y comunidades hispanas de ciberseguridad.

---

## 4. FixCheck — Re-verificación automática de hallazgos de pentest tras remediación

**Concepto.** API/CLI ligera que, dado un hallazgo estructurado de pentest (tipo de vulnerabilidad, host/endpoint, PoC original), ejecuta una comprobación no intrusiva cuando el cliente avisa de que "ya está arreglado": vuelve a comprobar la cabecera, el banner de versión, el endpoint expuesto o el parámetro inyectable sin relanzar el pentest completo. Devuelve un veredicto — corregido / no corregido / no verificable automáticamente — con evidencia (captura, respuesta HTTP) lista para pegar en el informe de cierre.

**Problema que resuelve.** La fase de reverificación tras remediación es pura repetición para el 60-70% de los hallazgos típicos de un pentest (versiones desactualizadas, cabeceras de seguridad ausentes, paneles expuestos, endpoints sin autenticar), y hoy se hace a mano, uno por uno, con las mismas herramientas del pentest original — horas que ni firmas ni freelancers suelen poder facturar del todo.

**Target.** Freelancers y firmas pequeñas de pentesting: el mismo comprador que una herramienta de validación de alcance, pero en el extremo opuesto del proyecto (cierre, no arranque).

**Modelo de negocio.** Por proyecto (9,99 € por lote de hasta 50 hallazgos) o suscripción: 49 €/mes ilimitado para freelancers, 199 €/mes multiusuario para firmas pequeñas.

**Ventaja de Tris.** Es literalmente el trabajo manual que hace cada vez que cierra un hallazgo, y los patrones de verificación por tipo de vulnerabilidad ya están enumerados en su propia base de conocimiento de pentesting — convertir esa base en un motor de reglas de verificación es una traducción directa, no una investigación desde cero.

**Dificultad.** 🟢 Fácil — motor de reglas + peticiones HTTP, sin necesidad de IA para una primera versión.

**Potencial de ingresos estimado.** 300–1.200 €/mes en 6 meses. Nicho pequeño pero con ciclo de venta muy corto: se vende con una demo de 5 minutos al mismo tipo de comprador que ya conoce el problema de primera mano.

**Validación en 2 semanas.** Publicar 15 reglas de verificación como script abierto en GitHub dirigido a la comunidad de pentest y medir estrellas/forks/issues en las dos primeras semanas.

---

## 🧭 Reflexión del día

**Se está abriendo una superficie de ataque nueva que todavía no tiene ni categoría de compra ni proveedor establecido: el navegador como agente.**

Comet y Atlas llevan meses en producción y ya hay exploits reales y reproducibles publicados por investigadores serios (Zenity Labs) — no son hipótesis de laboratorio, es acceso a archivos locales y toma de gestores de contraseñas mediante contenido web normal y corriente. Y la propia OpenAI ha dicho abiertamente que probablemente nunca se resuelva del todo. Eso es exactamente el patrón histórico de las mejores oportunidades de nicho en seguridad: un vector de ataque demostrado, sin categoría de producto todavía, y sin proveedor dominante — la misma ventana en la que nacieron el escaneo de contenedores o el CSPM hace unos años, ahora un ciclo más temprano.

En paralelo, dos señales más confirman que octubre-noviembre de 2026 es buen momento para sembrar, no para recoger todavía: el fraude por voz clonada se ha multiplicado más de 10 veces en intentos interanuales mientras solo un tercio de las empresas tiene alguna defensa desplegada, y el "acantilado de cumplimiento" de NIS2 en octubre va a obligar a sectores con OT/ICS (energía, agua, industria) a auditar entornos para los que casi no hay formación en español. Ninguna de las tres cosas es una moda pasajera de IA generativa — son consecuencias directas de tecnología ya desplegada (navegadores agénticos, clonación de voz) y de una fecha legal ya fijada (NIS2), lo que las hace más predecibles que la media de "ideas de negocio con IA".

**Y una nota sobre el momento personal, otra vez.** Quedan tres días para la incorporación del 29 de septiembre a un equipo ofensivo nuevo. Eso deja el orden de prioridad bastante claro:

- La idea 2 (simulacro de vishing) es la que más se parece al trabajo diario de una firma de seguridad ofensiva — es la que hay que aparcar primero, no la que hay que validar esta semana.
- Las ideas 1 (PageTrap) y 4 (FixCheck) son contenido/producto sin cliente propio: se pueden avanzar en fin de semana sin pisar terreno del nuevo empleador, y el escaneo gratuito de PageTrap se puede lanzar como investigación pública sin comprometer nada.
- La idea 3 (curso OT/IT) es la de mejor encaje con el momento vital: no compite con nadie, y si el nuevo puesto acaba tocando entornos OT, el aprendizaje genérico del día a día alimenta el curso sin usar una sola línea de material del cliente.
- El patrón se repite edición tras edición: lo que no depende de un cliente propio avanza estas semanas; lo que sí depende de un cliente, espera a tener la cláusula de exclusividad delante.

---

### Fuentes

- [Researchers discover suite of agentic AI browser vulnerabilities — CyberScoop](https://cyberscoop.com/agentic-ai-browsers-allow-hijacking-zenity-labs-comet/)
- [AI browsers wide open to attack via prompt injection — The Register](https://www.theregister.com/2025/10/28/ai_browsers_prompt_injection/)
- [Six Demonstrated Exploits That Prove Agentic Browser Security Is Not Theoretical — SoftwareSeni](https://www.softwareseni.com/six-demonstrated-exploits-that-prove-agentic-browser-security-is-not-theoretical/)
- [AI Voice Cloning Fraud Statistics 2026 — SQ Magazine](https://sqmagazine.co.uk/ai-voice-cloning-fraud-statistics/)
- [How vishing and AI voice cloning attacks target businesses — ThreatLocker](https://www.threatlocker.com/blog/ai-voice-cloning-and-vishing-attacks-what-every-business-must-know)
- [NIS2 And DORA Readiness: What European Businesses Must Do Before October 2026 — Cyber Threat Defense](https://ctdefense.com/nis2-dora-readiness-october-2026/)
- [Cloud Security Statistics 2026: Key Data & Trends — StationX](https://app.stationx.net/articles/cloud-security-statistics)
- [How to Automate Vulnerability Remediation in 2026 — Endor Labs](https://www.endorlabs.com/learn/vulnerability-remediation-automation)
