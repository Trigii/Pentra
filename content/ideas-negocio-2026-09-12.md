# 💡 Ideas de Negocio del Día — 12 de septiembre de 2026

> Rotación de hoy: **servicio recurrente (retainer)**, **consultoría técnica con herramienta propia**, **servicio de validación**, **auditoría exprés de precio fijo** e **infoproducto con laboratorios**.
> Evito repetir los ejes ya cubiertos en semanas anteriores (escáneres MCP, generadores de informes de pentest, auditorías NIS2/DORA genéricas, newsletters de seguridad IA).

---

## 1. PSIRT Fraccional — Guardia de reporte CRA 24h para fabricantes pequeños

**Nombre / Concepto:** Servicio externalizado que actúa como el PSIRT (Product Security Incident Response Team) de fabricantes pequeños de hardware/software, cubriendo la obligación de reportar vulnerabilidades explotadas a ENISA en 24h/72h/14 días.

**Problema que resuelve:** Desde **ayer, 11 de septiembre de 2026**, cualquier fabricante que ponga un "producto con elementos digitales" en el mercado de la UE debe enviar un aviso temprano a ENISA y al CSIRT nacional **en 24 horas** desde que sabe de una vulnerabilidad activamente explotada, notificación completa a las 72h e informe final a los 14 días. Incluye productos ya vendidos antes de la aplicación plena del CRA (dic 2027). Las multas llegan a 15 M€ o el 2,5% de la facturación global. Un fabricante de 30-80 personas (sensores IoT, PLCs, routers industriales, software embebido) **no tiene a nadie de guardia** que sepa redactar ese reporte ni decidir si un hallazgo cae dentro del umbral. El reloj corre en fin de semana y en agosto.

**Target:** Fabricantes europeos de 20-200 empleados con producto conectado: IoT industrial, dispositivos médicos clase baja, domótica, fabricantes de software embebido y firmware. Especialmente los que ya tienen marcado CE por otras directivas y descubren que el CRA les aplica sin tener equipo de seguridad. Foco inicial: España, Cataluña (clúster industrial cercano), Alemania e Italia.

**Modelo de negocio:** Retainer mensual de **800-2.500 €/mes** por fabricante que incluye: buzón de divulgación coordinada (`security@`) gestionado, política de CVD publicada, runbook de decisión, plantillas pre-rellenadas del portal único de reporte y disponibilidad de guardia. Extra por incidente real gestionado (1.500-4.000 €). Upsell: preparación de SBOM y documentación técnica de cara a diciembre 2027.

**Ventaja de Tris:** Un DevSecOps con máster en ciberseguridad entiende las dos mitades que aquí no se juntan casi nunca: el análisis técnico (¿esto es realmente explotable en producción? ¿hay evidencia de explotación activa?) y el proceso de pipeline/versionado del fabricante (¿en qué release entró, qué dependencia lo introdujo, cuándo sale el parche?). La parte legal es plantilla; la parte difícil es el juicio técnico bajo presión de 24 horas, que es exactamente su terreno.

**Dificultad de implementación:** 🟡 Media — sin producto que construir, pero requiere runbooks sólidos, disponibilidad real y credibilidad comercial ante fabricantes.

**Potencial de ingresos estimado:** 8-30 k€/mes con 10-15 clientes en retainer. Un solo cliente ancla paga el coste de arranque. Validable en 4-6 semanas: 20 llamadas a fabricantes industriales del área de Barcelona preguntando "¿quién os cubre el reporte de 24 horas del CRA?".

---

## 2. CryptoSight — Inventario criptográfico (CBOM) como sprint de 3 semanas

**Nombre / Concepto:** Escáner propio + servicio que recorre repos, contenedores, pipelines y certificados de una empresa y entrega un **Cryptographic Bill of Materials**: qué algoritmos, claves, librerías y certificados hay, cuáles son cuánticamente frágiles y en qué orden migrar.

**Problema que resuelve:** Entre finales de 2026 y enero de 2027 convergen tres fechas (transición de FIPS 140-2 a histórico, hito de estrategia nacional PQC en la UE, puerta de adquisición CNSA 2.0). Pero solo el **38%** de las organizaciones está migrando de verdad y apenas un **7%** tiene criptografía cuántico-segura desplegada en la mayoría de su parque de certificados. El cuello de botella no es el algoritmo nuevo: es que **nadie sabe qué criptografía tiene**. El NIST NCCoE calcula 12-24 meses solo para la fase de descubrimiento en una empresa grande. Los proveedores actuales de CBOM venden plataformas caras orientadas a banca; el mediano mercado no tiene nada asequible.

**Target:** Empresas de 200-2.000 empleados en sectores con obligación de resiliencia (fintech, salud, SaaS B2B con clientes regulados, proveedores de administraciones públicas). También consultoras pequeñas que necesitan subcontratar la parte técnica del descubrimiento. El comprador es el CISO o el responsable de arquitectura, no el CTO.

**Modelo de negocio:** Doble capa. (a) **Herramienta open source** que genera CBOM en formato CycloneDX a partir de código, imágenes de contenedor y escaneo de TLS — captación y credibilidad. (b) **Sprint de pago de 3 semanas, 8-15 k€**: ejecución sobre el entorno real, priorización por riesgo "harvest now, decrypt later" y hoja de ruta de migración. (c) Suscripción de re-escaneo trimestral 300-800 €/mes para mantener el CBOM vivo (crypto-agility no es un proyecto, es un estado).

**Ventaja de Tris:** Este trabajo es 80% recorrer pipelines, imágenes, configuraciones de TLS y dependencias — exactamente lo que hace un DevSecOps a diario — y 20% criptografía aplicada, que cubre el máster. La mayoría de consultoras de PQC son criptógrafos que no saben moverse por un CI/CD real; el diferencial de Tris es lo contrario y ahí está el cuello de botella. Además, el escáner es construible en semanas: la detección criptográfica es *pattern matching* sofisticado, no investigación.

**Dificultad de implementación:** 🟡 Media — la herramienta es abarcable; lo duro es vender un proyecto cuyo dolor aún es futuro.

**Potencial de ingresos estimado:** 60-150 k€/año con 8-12 sprints. El open source es el canal de entrada, no el producto.

---

## 3. Segunda Opinión — Validación humana de hallazgos de pentest autónomo

**Nombre / Concepto:** Servicio de verificación independiente que coge el output de una plataforma de pentesting con IA (XBOW, Escape, Stingrai y similares) y devuelve, por cada hallazgo, un veredicto reproducible: explotable / no explotable / explotable con condiciones, con PoC y contexto de negocio.

**Problema que resuelve:** El output crudo de agentes ofensivos corre entre un **18% y un 45% de hallazgos inválidos**. Las plataformas venden validadores automáticos, pero el equipo que recibe el informe sigue sin poder firmarlo ante auditoría, ante un cliente o ante su propio comité de riesgo. Se está creando un rol nuevo: alguien que ponga su nombre debajo del hallazgo. Hoy eso lo hace un ingeniero interno saturado que quema 6-10 horas por informe, o directamente nadie — y el backlog se llena de ruido que erosiona la confianza en la herramienta que la empresa acaba de pagar.

**Target:** Dos perfiles claros. (a) Empresas de 100-800 empleados que compraron una plataforma de pentesting autónomo en 2026 y no tienen equipo ofensivo propio para triar el resultado. (b) MSSPs y consultoras pequeñas que usan estas plataformas para escalar entregables y necesitan una capa de calidad antes de mandar el informe a su cliente. El segundo segmento es más fácil de cerrar: es su reputación la que está en juego.

**Modelo de negocio:** Precio por hallazgo validado (**40-90 € por finding**, con mínimo por lote) o retainer mensual por volumen (**1.500-4.000 €/mes** hasta X hallazgos). Entrega en 48-72h. Escalable: el propio trabajo genera un corpus de patrones de falsos positivos por plataforma, que con el tiempo se convierte en producto (reglas de descarte automático) o en informe de mercado vendible.

**Ventaja de Tris:** Es literalmente pentesting sin la parte comercial ni el descubrimiento — empiezas con el hallazgo ya sobre la mesa y tu trabajo es reproducirlo. Es la forma más limpia que existe de facturar horas de pentesting mientras se profundiza en ese perfil, y cada validación es entrenamiento pagado. Además, el DevSecOps le da el contexto de "¿esto es explotable *en este entorno concreto*?", que es justo donde el agente se equivoca.

**Dificultad de implementación:** 🟢 Fácil — cero producto, cero infraestructura. Se arranca con un documento de metodología y dos clientes piloto.

**Potencial de ingresos estimado:** 2-6 k€/mes en paralelo al empleo actual; 8-15 k€/mes si se convierte en dedicación completa con 4-6 clientes recurrentes. La vía más rápida a ingreso real de toda la lista.

---

## 4. Agent Keys Audit — Auditoría de identidades no humanas para startups de IA

**Nombre / Concepto:** Auditoría exprés de precio fijo que inventaría todas las identidades no humanas de una startup — claves de API de LLMs, tokens de MCP, service accounts, secretos de CI, credenciales de agentes — y entrega un mapa de a qué puede acceder cada agente y qué pasa si se filtra.

**Problema que resuelve:** Solo el **15%** de las organizaciones se siente capaz de prevenir ataques basados en identidades no humanas, y más del **16% ni siquiera registra** la creación de identidades relacionadas con IA. En una startup que ha metido agentes en producción durante 2026, la realidad típica es: una clave de OpenAI en el `.env` de tres desarrolladores, un token de servidor MCP con permisos de escritura en producción, un service account de CI con rol de administrador "temporal" desde marzo, y cero rotación. "Identity Assurance for an AI World" es la segunda prioridad de los CISOs para 2026, pero las soluciones que existen (Oasis, Okta y compañía) están diseñadas y tarificadas para empresas de 5.000 personas.

**Target:** Startups de 10-80 empleados con producto que usa agentes o LLMs en producción, típicamente entre seed y Serie A. El disparador de compra no es el miedo: es el **cuestionario de seguridad de su primer cliente enterprise** o la due diligence de la siguiente ronda. Vender en ese momento exacto es la clave.

**Modelo de negocio:** Auditoría de precio fijo **3.500-6.000 €**, entregada en 5-7 días laborables: inventario, matriz de exposición por identidad, hallazgos priorizados y un documento de una página que pueden adjuntar al cuestionario del cliente. Recurrencia: re-auditoría semestral al 50% del precio. Producto derivado natural: convertir el proceso en un script/CLI que se ejecute en su repo y su cloud, y venderlo después como suscripción.

**Ventaja de Tris:** Gestionar secretos, rotación, permisos de pipelines y service accounts **es el trabajo diario de un DevSecOps**. Aquí no hay que aprender un dominio nuevo: hay que empaquetar lo que ya hace en un entregable con precio y nombre. Y el ángulo ofensivo — "esta clave filtrada te da esto" — es lo que convierte un inventario aburrido en un informe que el fundador lee entero.

**Dificultad de implementación:** 🟢 Fácil — metodología + scripts existentes. El reto es comercial, no técnico.

**Potencial de ingresos estimado:** 3-5 k€ por auditoría, 2-4 al mes son alcanzables trabajando el ecosistema de startups de Barcelona y los inversores que hacen due diligence técnica. 70-120 k€/año si se convierte en dedicación principal.

---

## 5. "Pentesting de la Cadena de Suministro" — Infoproducto con laboratorios reales

**Nombre / Concepto:** Curso de pago con entorno de laboratorio propio que enseña a pentesters clásicos a atacar lo que no saben atacar: pipelines de CI/CD, registries de contenedores, runners de GitHub Actions, artefactos firmados, infraestructura como código y dependencias.

**Problema que resuelve:** El debate de 2026 en las comunidades de pentesting es de supervivencia: quien solo sabe ejecutar escáneres está siendo comoditizado por las plataformas autónomas, mientras quien combina explotación manual, razonamiento de arquitectura, rutas de ataque en cloud, abuso de identidad y análisis de código ocupa una capa que no se automatiza. El pentester de aplicaciones web con OSCP sabe hacer SQLi y XSS, pero no sabe qué hacer frente a un `workflow_run` mal configurado, un registry interno sin autenticación o un runner self-hosted compartido entre repos públicos y privados. La formación existente sobre esto es defensiva ("cómo asegurar tu pipeline") o son charlas de conferencia sueltas: **no existe un curso ofensivo estructurado con laboratorios**.

**Target:** Pentesters junior-mid con 1-4 años de experiencia, gente con OSCP que quiere diferenciarse, y equipos internos de red team que empiezan a recibir encargos sobre la plataforma de desarrollo. Mercado global en inglés, arranque posible en español para reducir competencia y validar rápido.

**Modelo de negocio:** Curso autoguiado con laboratorios en **199-349 €** (pago único), cohortes en vivo de 4 semanas a **600-900 €** dos veces al año, y suscripción opcional al laboratorio a **19 €/mes**. Validación previa sin construir nada: vender el acceso anticipado de la primera cohorte antes de grabar una sola lección. Distribución: escribir 5-6 writeups técnicos gratuitos de ataques reales a pipelines y dejar que hagan el trabajo de captación.

**Ventaja de Tris:** Aquí la ventaja es casi injusta. Es DevSecOps — vive dentro de los pipelines que el curso enseña a atacar — con máster en ciberseguridad y orientación a pentesting. Prácticamente nadie que enseñe pentesting tiene ese lado defensivo/operativo real, y prácticamente nadie que construya pipelines sabe atacarlos. Además, construir el curso es construir su propia especialización: el contenido sale del trabajo diario y, de paso, es el mejor argumento de autoridad posible en entrevistas y ante clientes de las otras cuatro ideas de esta lista.

**Dificultad de implementación:** 🟡 Media — el contenido es fácil para él, pero los laboratorios reproducibles cuestan tiempo y el marketing de infoproducto es un oficio en sí mismo.

**Potencial de ingresos estimado:** 5-15 k€ en el primer lanzamiento con lista de 800-1.500 suscriptores; 30-60 k€/año en régimen con lanzamientos recurrentes. Ingreso más pasivo que las ideas 1-4, pero con rampa más lenta.

---

## 🔭 Reflexión del día

**Ayer, 11 de septiembre de 2026, se abrió la ventana más concreta del año.** Las obligaciones de reporte del Cyber Resilience Act ya están vivas: miles de fabricantes europeos que nunca habían tenido un equipo de seguridad de producto ahora tienen 24 horas legales para hacer algo que no saben hacer. A diferencia de NIS2 o DORA — donde el cumplimiento es un proyecto largo con presupuesto anual y ciclos de compra lentos — el CRA crea una **necesidad operativa de guardia**, y las necesidades operativas se compran rápido y se pagan mensualmente. Ese es el tipo de demanda que un profesional independiente puede capturar antes de que las grandes consultoras monten la oferta.

El patrón de fondo que conecta las cinco ideas: **2026 es el año en que la automatización de seguridad generó más output del que nadie puede verificar**. Agentes ofensivos que producen entre un 18% y un 45% de hallazgos inválidos. Inventarios criptográficos que nadie tiene. Identidades de agentes que el 16% de las empresas ni siquiera registra. SBOMs incompletos que llegan de proveedores. En todos los casos, la máquina genera volumen y falta alguien que ponga **juicio técnico y su firma** encima. Ese rol — verificador, no ejecutor — es donde se está moviendo el valor del perfil ofensivo, y es accesible sin construir producto ni levantar capital.

**Recomendación de esta semana:** la idea 3 (Segunda Opinión) es la de menor fricción absoluta — cero producto, se puede facturar el mes que viene, y cada hora facturada es entrenamiento en pentesting pagado por otro. Si hay que elegir una sola para probar en 30 días, es esa.

---

## Fuentes

- [The CRA's 24-Hour Rule: Preparing for the Cyber Resilience Act's September 2026 Reporting Obligations — DLA Piper](https://www.dlapiper.com/en-us/insights/publications/2026/08/the-cras-24-hour-rule-preparing-for-the-cyber-resilience-acts-september-2026-reporting-obligations)
- [Cyber Resilience Act reporting obligations take effect on 11 September 2026 — Freshfields](https://www.freshfields.com/en/our-thinking/blogs/technology-quotient/cyber-resilience-act-reporting-obligations-take-effect-on-11-september-2026-102nzmk)
- [Cyber Resilience Act — Reporting obligations (Comisión Europea)](https://digital-strategy.ec.europa.eu/en/policies/cra-reporting)
- [EU CRA SBOM Requirements — Anchore](https://anchore.com/sbom/eu-cra/)
- [PQC Migration in 2026: Building a Roadmap That Survives Contact With Production — Encryption Consulting](https://www.encryptionconsulting.com/pqc-migration-in-2026/)
- [Cryptographic Bill of Materials (CBOM) — QuSecure](https://www.qusecure.com/cryptographic-bill-of-materials-cbom/)
- [AI Pentest False Positive Rate: 2026 Reference Table — Stingrai](https://www.stingrai.io/blog/acceptable-false-positive-rate-autonomous-pentest-2026)
- [AI pentesting needs validation and orchestration, not just models — NHI Mgmt Group](https://nhimg.org/articles/ai-pentesting-needs-validation-and-orchestration-not-just-models/)
- [AI Agents Are Creating an Identity Security Crisis in 2026 — IANS Research](https://www.iansresearch.com/resources/all-blogs/post/security-blog/2026/04/19/ai-agents-are-creating-an-identity-security-crisis-in-2026)
- [The Non-Human Identity Governance Vacuum — Cloud Security Alliance](https://labs.cloudsecurityalliance.org/research/csa-whitepaper-nonhuman-identity-agentic-ai-governance-v1-cs/)
- [Is Penetration Testing Dying? Reddit's 2026 Debate — ACSMI](https://acsmi.org/blogs/is-penetration-testing-dying-reddits-2026-debate-ai-pentesting-cloud-security-where-offensive-careers-are-moving)
