# Informe Semanal de Ciberseguridad
### Semana del 8 al 15 de septiembre de 2026

---

## Resumen ejecutivo

Semana dominada por el **Patch Tuesday más grande de la historia de Microsoft** (~974 CVEs, dos zero-days ya explotados) y por una acumulación inusual de fallos *perfect-10* en infraestructura de borde y desarrollo: SonicWall SMA1000 (CVSS 10.0), Cisco Secure FMC (10.0), GitLab (10.0) y el kernel de SAP (10.0, bautizado *OVERPASS*). El patrón es consistente y explotable: **appliances perimetrales y plataformas DevOps auto-hospedadas** siguen siendo la vía de entrada preferida, con ventanas patch-to-exploit que ya se miden en horas — GitLab pasó de disclosure a explotación activa en **un día**.

En paralelo, la IA ofensiva dejó de ser tendencia para convertirse en operativa: Anthropic y Google GTIG publicaron informes documentando actores estatales que automatizan el ciclo completo de ataque, incluido un bucle de **reconstrucción automática de malware cuando el EDR lo detecta** (Midnight Blizzard). Para red team, la lectura es doble: el arsenal se democratiza y la firma de sofisticación deja de servir como indicador de atribución.

Del lado del ransomware, ShinyHunters ejecutó su extorsión a McKesson (6,4M de afectados, rescate de 55,2M USD no pagado), Qilin apareció desplegándose vía Cisco FMC comprometidos, y The Gentlemen sumó dos objetivos sanitarios más. Cierra la semana un frente poco habitual: **PoCs públicos y funcionales de escalada de privilegios contra CrowdStrike Falcon, Avast y Kaspersky**, publicados sin coordinación por un mismo investigador.

---

## 1. Vulnerabilidades críticas y CVEs

### 🔴 Microsoft Patch Tuesday: récord absoluto y dos zero-days

Microsoft corrigió aproximadamente **974 CVEs** (las cifras varían entre 964 y 974 según el proveedor que las contabilice), de las cuales ~113 son críticas y ~20 wormables. Dos ya estaban bajo explotación activa:

| CVE | Componente | CVSS | Tipo |
|---|---|---|---|
| **CVE-2026-85880** | Windows ALPC | 7.8 | Heap overflow → EoP a SYSTEM |
| **CVE-2026-81963** | Windows Update Stack | 7.8 | Improper link resolution → EoP |

**Nota para red team:** CVE-2026-85880 es especialmente relevante porque permite **escapar de un AppContainer de bajo privilegio** sin interacción del usuario. Es el primer fallo ALPC parcheado desde abril de 2023. CVE-2026-81963 entró en el KEV de CISA el 8 de septiembre con deadline federal el 22.

Más significativo aún: **CVE-2026-85880 no aparece sola**. Proofpoint documentó un exploit kit no catalogado llamado **BlueMoon** que la encadena con dos fallos de Chrome (CVE-2026-85046 y CVE-2026-87491) — ver sección APTs.

### 🔴 SonicWall SMA1000 — cadena SSRF → RCE no autenticado

Divulgados el 1 de septiembre, **ya explotados como zero-day**:

- **CVE-2026-83548** (CVSS **10.0**) — SSRF no autenticado en la interfaz Appliance Work Place
- **CVE-2026-83549** (CVSS 7.8) — Command injection en el Appliance Management Console (AMC)

Encadenados producen **RCE no autenticado**: el SSRF alcanza el AMC interno, y desde ahí el command injection ejecuta comandos OS. Afecta a SMA1000 6210, 7210 y 8200v. Ambos en KEV.

### 🔴 GitLab CVE-2026-85706 — CVSS 10.0, explotado en 24h

Path traversal en la **API de commits del repositorio** que permite a un usuario **no autenticado** leer ficheros arbitrarios del servidor. Origen: confinamiento de rutas incorrecto + ausencia de enforcement de autenticación.

- **Parche:** 10 de septiembre (19.3.2, 19.2.6, 19.1.8). Afectadas todas las CE/EE desde 18.7.
- **Explotación:** sondeos desde las 06:00 UTC del 11 de septiembre; escaló rápidamente de fingerprinting a exfiltración real de ficheros de configuración, secretos y configuraciones SSH del sistema.
- Ya en KEV.

Este es el caso de estudio de la semana sobre compresión de la ventana patch-to-exploit.

### 🔴 SAP OVERPASS — CVE-2026-44756 (CVSS 10.0)

Corrupción de memoria en el procesamiento del **Extended Passport (EPP)** del kernel de SAP, por falta de validación de límites al deserializar campos de longitud suministrados externamente. Explotable **remotamente y sin autenticación**, con ejecución de comandos OS con privilegios administrativos de SAP.

Lo que lo hace grave es la **superficie de alcance**: el código EPP es compartido entre protocolos, de modo que es accesible desde la capa web expuesta a Internet, desde la capa SAP GUI y desde la capa RFC que interconecta sistemas SAP. Onapsis estima **más de 10.000 sistemas SAP expuestos a Internet** con el componente vulnerable. SAP Security Note 3747649.

Acompañado de **CVE-2026-58240** (*S4GET*), pre-auth en el Message Server de NetWeaver.

### 🔴 Cisco Secure Email Gateway — CVE-2026-76461 (CVSS 9.8), zero-day activo

Validación insuficiente en la **lógica de parseo de correo** de Cisco AsyncOS para Secure Email Gateway. Un atacante **no autenticado y remoto** ejecuta comandos arbitrarios **como root** enviando un mensaje de correo manipulado con sentencias SQL maliciosas. Afecta a appliances virtuales y físicos **con independencia de la configuración del dispositivo**.

El PSIRT de Cisco tuvo conocimiento de la explotación en septiembre de 2026; no se han publicado detalles de las campañas ni atribución. En KEV con deadline federal el **17 de septiembre**.

**Advertencia importante de Cisco:** se han publicado IoCs, pero dado que el atacante obtiene privilegios root, **puede eliminarlos u ocultarlos**. La ausencia de IoCs no es evidencia de ausencia de compromiso — si hay un SEG expuesto, asumir compromiso y validar por otras vías.

### 🟠 Otros de alto impacto

- **Cisco Secure FMC — CVE-2026-20079 (CVSS 10.0):** bypass de autenticación con ejecución de scripts como root. Explotación confirmada desde agosto. En KEV con deadline 12 de septiembre.
- **Citrix NetScaler ADC/Gateway — CVE-2026-19490 (CVSS 9.3):** bypass de autenticación cuando el appliance opera como AAA vserver o Gateway (SSL VPN, ICA Proxy, CVPN, RDP Proxy). Honeypots de Previdian registraron 56 intentos desde el 3 de septiembre, 36 de ellos solo el día 8.
- **Check Point — CVE-2026-85102 y CVE-2026-85103 (ambas CVSS 9.8):** validación incorrecta de confianza de certificado durante la negociación VPN, y heap overflow al decodificar la estructura ASN.1 del certificado VPN. **RCE no autenticado** contra Security Gateway, Spark Firewall y Quantum Security Management. Divulgadas y parcheadas el 9 de septiembre. Sin PoC público ni explotación conocida, pero el **NCSC neerlandés advierte de explotación inminente**.
- **JFrog Artifactory — CVE-2026-42018 + CVE-2026-42016 + CVE-2026-82329:** cadena explotada entre el 15 de agosto y el 8 de septiembre. CVE-2026-42018 expone el token del usuario anónimo interno; CVE-2026-42016 escala ese token a scope admin (la validación comprueba firma/emisor pero no el scope). Resultado: petición no autenticada → token admin en dos pasos. Wiz observó despliegue de un **backdoor custom en Rust con C2**, cuentas admin persistentes y plugins maliciosos para RCE. **El 59 % de las organizaciones seguía vulnerable seis semanas después.**
- **PaperCut NG/MF — CVE-2026-81578 y CVE-2026-82078:** explotación activa documentada por watchTowr, con despliegue de **implantes en memoria** — web shells Godzilla y túneles proxy HTTP *suo5*, ambos como servlet filters que interceptan peticiones HTTP sin escribir nada en disco. Detalle notable: 18 segundos después del despliegue en su honeypot, una IP distinta empezó a interactuar con la web shell usando la clave AES y password correctas.
- **Apple — iOS 27 y macOS Golden Gate 27:** cerca de **200 vulnerabilidades** corregidas, ~100 de ellas afectando a ambos sistemas, sobre más de 90 componentes (AppleKeyStore, Authentication Services, Foundation, Safe Browsing, Sandbox, Security, TCC, WebKit). Destaca **CVE-2026-64752**, corrupción de memoria en CoreMedia explotable mediante una imagen maliciosa mostrada al usuario — Apple optó por **eliminar el código afectado en lugar de parchearlo**. Incluye también correcciones de kernel con impacto en corrupción de memoria, escalada de privilegios y fuga de información.

### Adiciones destacadas al KEV de CISA esta semana

| CVE | Producto | Deadline FCEB |
|---|---|---|
| CVE-2026-76461 (9.8) | Cisco Secure Email Gateway | 17 sep |
| CVE-2026-84869 (9.9) | ConnectWise ScreenConnect | 14 sep |
| CVE-2026-67277 / CVE-2026-86060 | MikroTik RouterOS (*MikroTrick*) | 13 sep |
| CVE-2026-42016/42018/82329 | JFrog Artifactory | 25 sep |
| CVE-2025-25249 | Fortinet | 12 sep |
| CVE-2026-19490 | Citrix NetScaler | 12 sep |
| CVE-2026-20079 | Cisco Secure FMC | 12 sep |

---

## 2. Brechas y hackeos

### McKesson — ShinyHunters ejecuta la extorsión

La cadena de ataque es un manual de ingeniería social moderna y merece atención para diseño de engagements:

1. **Vishing** a empleados de McKesson
2. Credenciales robadas → **takeover de cuentas Okta SSO**
3. Pivote a entornos **Salesforce y Snowflake**
4. Exfiltración de ~1 TB durante cuatro días

ShinyHunters reclamó inicialmente 284 millones de documentos. La cifra confirmada vía Have I Been Pwned: **6,4 millones de personas afectadas**. Exigieron **55,2 millones de dólares**; ante la falta de pago, publicaron los datos. Incluyen nombres, direcciones físicas y de correo, género, fechas de nacimiento, teléfonos, datos del empleador e **información sanitaria sensible**.

### Florida DMV — la cadena de suministro humana

ShinyHunters comprometió la plataforma **DAVID** del DMV de Florida, con más de 200.000 registros de conductores sustraídos. El vector confirmado por FLHSMV: credenciales de **un único usuario del Plant City Police Department**, almacenadas indebidamente en su dispositivo personal.

Recordatorio de que el acceso de terceros con privilegios sobre bases de datos gubernamentales suele ser el eslabón más débil de la cadena.

### IDScan — 153 millones de documentos de identidad

La firma de verificación de identidad IDScan confirmó acceso no autorizado a datos de clientes en su plataforma cloud, tras vincularse con una filtración de **escaneos de 153 millones de permisos de conducir** de ciudadanos estadounidenses y canadienses. Los datos se comercializaban a través de un servicio ilícito llamado **Nexus**, ya offline. Expuestos: nombres completos y números de identificación gubernamental.

---

## 3. Ransomware y malware

### Qilin vía Cisco FMC

Cisco Talos identificó **tres clusters de actividad post-compromiso** en instancias FMC (UAT-12197, UAT-11823 y UAT-11988), mezclando actores estatales y crimeware. UAT-11988 es el relevante para ransomware: tras el acceso inicial, empleó **tooling nativo del propio FMC en modo living-off-the-land** para reconocimiento, desplegó herramientas de tunelización para persistencia, recolectó credenciales, construyó la lista de endpoints objetivo, terminó herramientas de seguridad y desplegó **Qilin**. También se observó despliegue de **Cyclops Blink**.

Un firewall management center comprometido es, efectivamente, un punto de control privilegiado sobre toda la red — y sus herramientas legítimas son indistinguibles de la actividad administrativa normal.

### The Gentlemen — el RaaS que está comiéndose el sector sanitario

Operación de doble extorsión activa desde mediados de 2025, con capacidad de cifrado sobre **Windows, Linux, NAS, BSD y ESXi**. Su leak site lista **más de 800 víctimas en 86 países** (manufactura, tecnología, sanidad, transporte, servicios financieros) — una progresión notable: 478 víctimas en junio, 1.570 identificadas vía su C2 SystemBC en abril.

Dos reclamaciones sanitarias recientes:

- **Veradigm** (reclamada el 5 de septiembre): el grupo alega retener **3,5 millones de registros de pacientes** con nombres completos, direcciones, SSN, correos, teléfonos y PII de garantes. Veradigm ha notificado brecha de datos de pacientes.
- **Nutex Health**: notificación a la SEC confirmando acceso a información de pacientes, empleados, proveedores, negocio y financiera. Deadline de publicación fijado en el 11 de septiembre.

### Cl0p / PTC Windchill — campaña en curso

Contexto necesario aunque el grueso de la actividad es de agosto: Cl0p ha nombrado **más de 40 organizaciones** en la campaña que explota **CVE-2026-12569** en las plataformas PLM de PTC (Windchill y FlexPLM) — validación de entrada incorrecta que permite RCE remoto no autenticado, y **la primera vulnerabilidad de Windchill explotada in the wild**. Entre las listadas: **Shell, Philips, Fiserv, Zebra Technologies, Ingersoll Rand, Toast, Mindray y Largan Precision**. GE figuró inicialmente y fue retirada, lo que suele indicar pago o reanudación de negociación.

El detalle técnico relevante lo aportó ReliaQuest: Cl0p emplea un **implante custom** con capacidad de robo de datos completa sin herramientas adicionales. La web shell mapea datos sensibles del vault, **descifra todas las credenciales del keystore de Windchill**, e incluye un class loader Java propio que permite ejecutar código arbitrario dentro del proceso de la aplicación — convirtiendo la shell en backdoor ilimitado para movimiento lateral, ransomware o persistencia.

Mismo patrón que MOVEit, Cleo, GoAnywhere y Oracle EBS. Si hay Windchill en el alcance, CVE-2026-12569 es prioridad.

### FireClient — nueva cadena de instalación

BlueVoyant documentó una evolución en el despliegue del backdoor **FireClient**, dentro de campañas de ingeniería social vía **Microsoft Teams**. El cambio: se abandona el abuso del perfil de Firefox en favor de un mecanismo basado en **MSI**.

```
MSI (Windows Installer)
  └─ versión portable de Kodi
       └─ sideload de zlib.dll troyanizada
            └─ loader FireClient
                 └─ C2 tras endpoints REST de AWS API Gateway
                      └─ backdoor FireClient
```

Variantes posteriores del loader se camuflan como **VMware Tools y NCPA**. La intrusión progresa con robo de credenciales, movimiento lateral y exfiltración.

### Rootkit Linux en F5 BIG-IP APM

Despliegue de un rootkit Linux sobre appliances F5 BIG-IP APM comprometidos, que **intercepta la carga de ficheros PHP e inyecta una web shell fileless directamente en memoria**. Se sospecha segunda etapa tras explotación de **CVE-2025-53521** (RCE crítico parcheado en marzo de 2026). La web shell acepta peticiones con formato específico, descifra el contenido, lo ejecuta vía `eval()` de PHP y devuelve un **HTTP 201 disfrazado de hoja de estilos CSS**. ESET lo rastrea como **PoisonedRefresh**.

---

## 4. APTs y amenazas estatales

### 🎯 BlueMoon — cuatro grupos, el mismo exploit kit, la misma semana

El hallazgo más interesante de la semana desde la perspectiva de inteligencia de amenazas. Proofpoint documentó un exploit kit no catalogado previamente, **BlueMoon**, que encadena:

- **CVE-2026-85046** (Chrome)
- **CVE-2026-87491** (Chrome)
- **CVE-2026-85880** (Windows ALPC — el zero-day del Patch Tuesday)

Es decir: **RCE en navegador → escape de sandbox → SYSTEM**.

Cuatro clusters de espionaje lo emplearon, **tres de ellos evaluados como alineados con China**, contra menos de 20 organizaciones a nivel global. La coincidencia temporal reabre la hipótesis del **"digital quartermaster"**: un proveedor común de tooling ofensivo que abastece simultáneamente a varios grupos, o bien una venta as-a-service a múltiples actores.

### Midnight Blizzard automatiza el bypass de EDR

Anthropic publicó su informe de inteligencia de amenazas (actividad de diciembre 2025 a agosto 2026) y detalló **GTG-20006**, con atribución consistente con **Midnight Blizzard** (Rusia). Uno de los operadores usa el handle `JackPoterz`.

La técnica que importa para red team y para blue team por igual:

> El actor construyó un **bucle de retroalimentación** que monitorizaba si su malware evadía la detección de los productos de seguridad. Cuando una herramienta era marcada, agentes de IA la **modificaban, recompilaban y redesplegaban automáticamente**, repitiendo el ciclo hasta que el malware pasaba desapercibido.

Anthropic lo describe como un desplazamiento de la carga hacia el defensor: el adversario "cierra el bucle" y supera los controles más rápido de lo que estos se desarrollan y despliegan.

Otras operaciones del mismo actor:
- **CaptiveCrunch** — compromiso de al menos tres proveedores de hospitality que operan Wi-Fi de invitados en hoteles, con credenciales admin robadas y **DNS hijacking** para redirigir el tráfico de huéspedes.
- Exportación masiva de buzones en fabricantes de componentes de drones, y robo de un **SDK completo de un sistema de visión para drones**.

Objetivos: inteligencia militar en gobiernos ucranianos y europeos, organizaciones diplomáticas y de defensa, e individuos vinculados a la política exterior estadounidense.

### UNC3569 / GRAYRABBIT vía Sogou Input Method

Actor alineado con China explotando **CVE-2026-51990** en el Sogou Input Method de Tencent para Windows, desplegando el backdoor **GRAYRABBIT**. La cadena de exploit es notable por acumular tres debilidades en un **one-click RCE**:

1. Inyección de argumentos de línea de comandos no validados en el protocol handler `sgbiz:`
2. Navegación URL sin restricciones en un webview basado en CEF
3. Motor Chromium **severamente desactualizado y sin sandbox** — Sogou empaquetaba Chromium 80, lo que permitía aprovechar **CVE-2021-38003** (type confusion en V8)

Tencent lo corrigió en abril de 2026. Caso de libro sobre riesgo de componentes de terceros embebidos y sin mantener.

---

## 5. Tendencias y herramientas

### La IA ofensiva pasa a producción

Semana clave con informes simultáneos de **Google GTIG** y **Anthropic**.

**GTIG** — evaluación más matizada de lo que sugieren los titulares:

- **No** se han observado aún pipelines completamente autónomos desplegados contra objetivos reales.
- Lo que sí se observa: adversarios usando modelos comerciales y open-weight para **convertir disclosures públicos y retrasos en el parcheo en código de exploit N-day funcional**, refinar tooling y avanzar hacia cadenas de exploit multi-etapa.
- Primer caso identificado de un **zero-day desarrollado con IA**: un script Python que bypassea 2FA en una herramienta open-source de administración web.
- Actores chinos y norcoreanos particularmente interesados en usar IA para **descubrimiento de vulnerabilidades**. Un actor vinculado a China desplegó herramientas agénticas **Strix y Hexstrike** contra una tecnológica japonesa y una empresa de ciberseguridad del este asiático. **UNC2814** usó jailbreaks persona-driven para investigación de vulnerabilidades en dispositivos embebidos.

La cita que resume el cambio de paradigma, de David Agranovich (Google):

> La brecha entre un operador individual y un actor estado-nación se ha cerrado en buena medida.

Y de Anthropic, la implicación directa para atribución: la sofisticación **ya no es una señal fiable de quién está detrás de una operación**. El corolario operativo que señalan — "la seguridad por oscuridad ya no es viable" — tiene sentido: configuraciones únicas y oscuras que antes protegían por fricción ahora se vuelven comprensibles y explotables de forma trivial.

### Abuso de Microsoft 365 Direct Send

KnowBe4 identificó **29.785 spoofs confirmados** vía Direct Send durante julio y agosto de 2026. La funcionalidad existe para que impresoras, escáneres y aplicaciones legacy on-premise envíen correo sin cuenta dedicada — **saltándose los gateways de seguridad**. Un atacante que se conecta a ese mismo endpoint abierto puede enviar correo aparentando ser cualquier persona de la organización, y el mensaje llega con apariencia interna porque técnicamente entró por la propia infraestructura.

Patrón temporal observado: actividad concentrada lunes-martes en horario laboral ET, con pico justo antes del mediodía y máximo hacia las 14:00.

**Aplicación directa para red team:** vector de phishing interno de alta credibilidad que evita controles perimetrales. Verificar en cada engagement si Direct Send está habilitado sin restricción de IP.

### Campaña RMM en 46 países

Inicialmente atribuida a targeting canadiense por usar formularios de la Canada Revenue Agency como señuelo, resultó ser una operación global. ANY.RUN conectó **601 casos**. El **45 % de la actividad se asocia a EE. UU.**.

- **Método:** documentos falsos que inducen a instalar software RMM **legítimo** — envíos y comunicaciones UPS, PDFs de Adobe, avisos fiscales, temáticas de la Social Security Administration, facturas.
- **Infraestructura:** Vercel desechable y rotada rápidamente.
- **Pivotes de correlación:** `font1.woff2`, recursos de imagen recurrentes y la estructura de entrega `secure.html → project/*.zip`.
- **Sectores principales:** educación, tecnología y gobierno; también banca, finanzas y manufactura.

### Vishing contra ejecutivos y help desk

Continúa la tendencia de **llamadas de IT falsas** dirigidas a ejecutivos para robo de datos y extorsión en Microsoft 365 — la misma familia de TTP que funcionó contra McKesson. El vector help desk / vishing es, a día de hoy, el camino más rentable hacia el SSO corporativo.

### Google Play Early Access como punto ciego

Bitdefender documentó abuso del programa Early Access de Google Play para distribuir apps engañosas (falsos casinos, recompensas, contenido premium). El problema estructural: **Early Access elimina reseñas y valoraciones**, que son precisamente el mecanismo temprano de alerta del usuario. Promoción vía TikTok y Facebook, con anuncios que emplean **deepfakes generados por IA de celebridades**. Permisos anómalos observados: un escáner de códigos QR solicitando reemplazar el launcher oficial de Android.

### Investigación destacada: WeWorm (WeChat)

Los investigadores de Calif divulgaron **WeWorm**, un gusano zero-click sobre WeChat capaz de propagarse por llamadas en Android e iOS **sin que el destinatario conteste**. Toma control de la cuenta en segundos, y desde ahí llama a otro contacto repitiendo el proceso. Detalle relevante: **rechazar la llamada detiene la infección**; contestar o dejarla sonar la propaga. Prerrequisito: el atacante debe estar en la lista de amigos de la víctima. Tencent parcheó el 21 de agosto (Android 8.0.77, iOS 8.0.76). Sin evidencia de explotación in the wild.

---

## 6. Mundo corporativo y regulación

### 🇪🇺 Cyber Resilience Act — arranca el reloj de 24 horas

**El cambio regulatorio más relevante de la semana.** Desde septiembre de 2026 son aplicables las obligaciones de notificación del CRA: los fabricantes que venden productos con elementos digitales en la UE deben reportar **vulnerabilidades explotadas activamente** a las autoridades de ciberseguridad.

- **Aviso temprano: 24 horas** desde el conocimiento de la explotación activa
- **Notificación detallada: 72 horas**

Esto sitúa al CRA junto a NIS2 y DORA en un marco de plazos cada vez más comprimidos. Conviene tener presente la diferencia estructural: **DORA es sectorial** y de aplicación directa a 20 tipos de entidades financieras, mientras **NIS2 es una directiva horizontal** sobre 18 sectores (energía, transporte, infraestructura digital, manufactura, retail). Ambas convergen en asegurar la cadena de suministro y en responsabilizar a las empresas de sus prácticas.

### M&A

El mercado mantiene ritmo alto: 33 operaciones en agosto, 21 en julio, 37 en junio. Movimientos recientes de referencia:

- **Visa → BioCatch** por 2.400 M USD (prevención de fraude)
- **Cyera → Oasis Security** por ~1.000 M USD (gobernanza de identidades no humanas, orientada a proteger agentes de IA y cuentas de servicio)
- **Okta → Permiso Security** por ~200 M USD (detección continua de amenazas de identidad, incluyendo identidades autónomas de IA)

El hilo conductor es evidente: **la identidad no humana y la seguridad de agentes de IA** son donde se concentra el capital. La seguridad de IA fue la mayor categoría de inversión seed del último trimestre, cerca de una cuarta parte de todas las operaciones.

### Aplicación de la ley

- **Xinbi Guarantee desmantelado:** el DoJ estadounidense incautó canales de Telegram y dos wallets de criptomonedas, y desplegó la Scam Center Strike Force en Madagascar para desarticular 13 complejos de estafa operados por sindicatos del crimen organizado chino. El Tesoro sancionó la plataforma y dos empresas de apoyo (Anwen Technology, desarrolladora de XinbiPay; y SafeW Technology). TRM Labs describe un marketplace con escrow que conectaba sindicatos de estafa con vendedores de datos robados, documentos de identidad falsos, herramientas de deepfake y servicios de cash-out, liquidando principalmente en USDT sobre TRON. **Más de 36.000 millones de dólares procesados desde 2022.**
- **Conti:** Oleksii Lytvynenko, ucraniano de 44 años, condenado a **4 años de prisión** por su participación en Conti — como intruso y como desarrollador, con daño directo a al menos 12 empresas. Se declaró culpable en junio de 2026.

### Ruido en el ecosistema de disclosure: el caso Nightmare Eclipse

El investigador conocido como **Nightmare Eclipse** / **Chaotic Eclipse** / **Infinite Nightmare** / **MSNightmare** — identificado como **Abdelhamid Naceri**, exempleado de Microsoft despedido en septiembre de 2024 por compartir información de vulnerabilidades con terceros — ha ampliado su campaña de disclosure no coordinado más allá de Microsoft.

**Contra Microsoft:** publicó PoC de **ShieldCrash**, zero-day en Microsoft Defender que es bypass del parche de **CVE-2026-69414** (*ShieldBreak*), que a su vez era bypass de **CVE-2026-50656** (*RoguePlanet*). Tercera iteración de la misma cadena. Naceri tiene en su historial CVE-2021-41379 y CVE-2021-24084.

**Contra otros proveedores (nuevo este mes):** tres zero-days adicionales, todos con PoC público.

| Nombre | Objetivo | Impacto |
|---|---|---|
| **FalconFlank** | CrowdStrike Falcon Sensor | EoP vía la función de remediación de macros maliciosas de Office |
| **PrettyPrague** | Sandbox de Avast | Shell con privilegios de sistema completos; posible impacto en AVG y Norton (GenDigital) |
| **GreenSection** | Nvidia | DoS — crash de cualquier app que use Vulkan u OpenGL |

Precedente en agosto: **HardBreacher**, EoP en producto endpoint de Kaspersky (parcheado el 31 de agosto). **Kevin Beaumont confirmó que los exploits de Avast, CrowdStrike y Kaspersky funcionan.**

**Lectura para red team:** hay PoCs públicos y funcionales de escalada de privilegios contra productos EDR/AV de primera línea. Son material directamente utilizable en engagement — y, por la misma razón, material que los operadores de ransomware ya están incorporando. Conviene verificar el estado de parcheo de Falcon Sensor y productos GenDigital en los entornos cliente.

---

## 🔭 Para estar atento esta semana

**1. Check Point VPN — la explotación es cuestión de tiempo**
El NCSC neerlandés evalúa como alta la probabilidad de explotación de CVE-2026-85102/85103. No hay PoC público todavía, pero un RCE no autenticado 9.8 contra gateways VPN es exactamente el perfil que las bandas de ransomware priorizan. Los antecedentes de Check Point con Qilin este mismo año refuerzan la expectativa. **Si hay Security Gateway o Spark Firewall en el alcance, parchear ya.**

**2. GitLab CVE-2026-85706 — escalada del payload**
La explotación ya pasó de lectura de ficheros a exfiltración de secretos y configuraciones SSH. El siguiente paso lógico es el uso de esas credenciales para **compromiso de la cadena de suministro de software**: runners CI/CD, tokens de registry, claves de despliegue. Esperar incidentes derivados en las próximas dos semanas.

**3. JFrog Artifactory — el 59 % sin parchear**
Un artifact repository comprometido es envenenamiento de la cadena de suministro por diseño. Con backdoors en Rust ya desplegados y persistencia establecida vía cuentas admin y plugins, **parchear no basta**: hay que auditar cuentas administrativas, plugins instalados y tokens emitidos desde el 15 de agosto.

**4. BlueMoon — ¿proliferación más allá de los cuatro clusters?**
Si la hipótesis del quartermaster digital o del modelo as-a-service es correcta, el kit debería aparecer en manos de más actores. Vigilar la telemetría de Chrome y el parcheo de CVE-2026-85880 — la cadena completa deja de funcionar si se corta cualquiera de los tres eslabones.

**5. SAP OVERPASS — 10.000 sistemas expuestos y sin explotación pública aún**
CVSS 10.0, pre-auth, alcanzable desde tres capas distintas, y un inventario expuesto grande y notoriamente lento de parchear. Es el candidato más probable a **próxima campaña masiva tipo Cl0p**. El precedente de Windchill está fresco.

**6. Cisco Secure Email Gateway — la explotación precede a la visibilidad**
CVE-2026-76461 se explota enviando un correo. No hace falta acceso previo, ni configuración concreta, ni interacción del usuario. Cisco reconoce que los IoCs publicados pueden ser borrados por un atacante con root. Esperar más detalles de atribución y, probablemente, revisión al alza del alcance en los próximos días.

**7. PoCs anti-EDR en circulación — ventana de adopción por criminales**
FalconFlank, PrettyPrague y HardBreacher son funcionales y públicos. El patrón histórico dice que las bandas de ransomware incorporan este tipo de PoC en cuestión de semanas, porque resuelve su problema más caro: la terminación de herramientas de seguridad antes del cifrado. Vigilar advisories de CrowdStrike y GenDigital.

**8. Cyber Resilience Act — primeras notificaciones bajo el reloj de 24h**
Los primeros casos reales de aplicación revelarán qué tan operativo es el plazo. Merece seguimiento por su impacto en los tiempos de disclosure coordinado, que afectan directamente a la ventana de trabajo de la investigación ofensiva.

---

## Fuentes

**Vulnerabilidades y parches**

- [Microsoft's September 2026 Patch Tuesday Addresses 964 CVEs — Tenable](https://www.tenable.com/blog/microsofts-september-2026-patch-tuesday-addresses-964-cves-cve-2026-81963-cve-2026-85880)
- [Microsoft Patches Record 974 Flaws, Including Two Exploited Windows Zero-Days — The Hacker News](https://thehackernews.com/2026/09/microsoft-patches-record-974-flaws.html)
- [Microsoft Patches Record 974 Vulnerabilities — SecurityWeek](https://www.securityweek.com/microsoft-patches-record-974-vulnerabilities-including-two-exploited-zero-days/)
- [Microsoft September 2026 Patch Tuesday fixes 966 flaws — BleepingComputer](https://www.bleepingcomputer.com/news/microsoft/microsoft-september-2026-patch-tuesday-fixes-966-flaws-2-zero-days/)
- [SonicWall SMA1000 vulnerabilities in active exploitation — Sophos](https://www.sophos.com/en-us/blog/sonicwall-83548-83549)
- [Critical SonicWall SMA1000 Vulnerabilities Exploited in the Wild — Rapid7](https://www.rapid7.com/blog/post/etr-critical-sonicwall-sma1000-vulnerabilities-cve-2026-83548-cve-2026-83549-exploited-in-the-wild/)
- [SonicWall SMA 1000 appliances under attack via zero-day flaws — Help Net Security](https://www.helpnetsecurity.com/2026/09/02/sonicwall-sma-1000-cve-2026-83548-cve-2026-83549-zero-day-attacks/)
- [GitLab CVSS 10 File-Read Flaw Draws In-the-Wild Probes — The Hacker News](https://thehackernews.com/2026/09/gitlab-cvss-10-file-read-flaw-draws-in.html)
- [Perfect-10 GitLab bug under attack days after patch lands — The Register](https://www.theregister.com/security/2026/09/14/perfect-10-gitlab-bug-under-attack-days-after-patch-lands/5296176)
- [CISA: Hackers now exploit max severity GitLab flaw — BleepingComputer](https://www.bleepingcomputer.com/news/security/cisa-hackers-now-exploit-max-severity-gitlab-flaw-in-attacks/)
- [SAP Patches CVSS 10.0 Kernel Flaw — The Hacker News](https://thehackernews.com/2026/09/sap-patches-cvss-100-kernel-flaw.html)
- [Mitigating OVERPASS (CVE-2026-44756) — Onapsis](https://onapsis.com/blog/sap-overpass-remediation/)
- [SAP warns of maximum severity 'OVERPASS' kernel vulnerability — BleepingComputer](https://www.bleepingcomputer.com/news/security/sap-warns-of-maximum-severity-overpass-kernel-vulnerability/)
- [Check Point Discloses Two 9.8-Rated VPN Certificate Flaws — The Hacker News](https://thehackernews.com/2026/09/check-point-discloses-two-98-rated-vpn.html)
- [Dutch NCSC: Critical Check Point VPN flaws exploitation is imminent — BleepingComputer](https://www.bleepingcomputer.com/news/security/dutch-ncsc-critical-check-point-vpn-flaws-exploitation-is-imminent/)
- [Artifactory Under Attack: In-the-Wild Exploitation — Wiz](https://www.wiz.io/blog/artifactory-under-attack-in-the-wild-exploitation-of-cve-2026-42016-cve-2026-4201)
- [Attackers Chain JFrog Artifactory Flaws — The Hacker News](https://thehackernews.com/2026/09/attackers-chain-jfrog-artifactory-flaws.html)
- [CISA Flags Exploited Cisco, Citrix, Fortinet Flaws — The Hacker News](https://thehackernews.com/2026/09/cisa-flags-exploited-cisco-citrix.html)
- [CISA Adds 5 Actively Exploited Artifactory, ScreenConnect, and RouterOS Flaws to KEV — The Hacker News](https://thehackernews.com/2026/09/cisa-adds-5-actively-exploited.html)
- [CISA Adds Four Known Exploited Vulnerabilities to Catalog — CISA](https://www.cisa.gov/news-events/alerts/2026/09/09/cisa-adds-four-known-exploited-vulnerabilities-catalog)
- [Root RCE Zero-Day in Cisco Secure Email Gateway Under Active Exploitation — SecurityWeek](https://www.securityweek.com/root-rce-zero-day-in-cisco-secure-email-gateway-under-active-exploitation/)
- [Cisco Secure Email Gateway Flaw Exploited in the Wild — The Hacker News](https://thehackernews.com/2026/09/cisco-secure-email-gateway-flaw.html)
- [Apple Patches 200 Vulnerabilities With New iOS 27, macOS Golden Gate 27 Releases — SecurityWeek](https://www.securityweek.com/apple-patches-200-vulnerabilities-with-new-ios-27-macos-golden-gate-27-releases/)
- [Nightmare Eclipse Drops CrowdStrike, Nvidia, Avast Zero-Day Exploits — SecurityWeek](https://www.securityweek.com/nightmare-eclipse-drops-crowdstrike-nvidia-avast-zero-day-exploits/)
- [Researcher Releases FalconFlank PoC Showing Privilege Escalation in CrowdStrike Falcon — The Hacker News](https://thehackernews.com/2026/09/researcher-releases-falconflank-poc.html)
- [New CrowdStrike 'FalconFlank' zero-day grants SYSTEM privileges — BleepingComputer](https://www.bleepingcomputer.com/news/security/new-crowdstrike-falconflank-zero-day-grants-system-privileges/)

**Brechas e incidentes**

- [ShinyHunters expose 6.4M in attack on medical supplier McKesson — The Register](https://www.theregister.com/security/2026/09/10/shinyhunters-expose-64m-in-attack-on-medical-supplier-mckesson/5295550)
- [McKesson Confirms Data Breach as Attacker Deadline Looms — SecurityWeek](https://www.securityweek.com/mckesson-confirms-data-breach-as-attacker-deadline-looms/)
- [Florida confirms DMV database breached via stolen police account — BleepingComputer](https://www.bleepingcomputer.com/news/security/florida-confirms-dmv-database-breached-via-stolen-police-account/)

**Ransomware y malware**

- [Cl0p Ransomware Group Names Over 40 Victims of PTC Windchill Campaign — SecurityWeek](https://www.securityweek.com/cl0p-ransomware-group-names-over-40-victims-of-ptc-windchill-campaign/)
- [Cisco FMC flaws exploited by ransomware gang, state-sponsored hackers — BleepingComputer](https://www.bleepingcomputer.com/news/security/cisco-fmc-flaws-exploited-by-ransomware-gang-state-sponsored-hackers/)
- [Cisco FMC Flaws Exploited to Steal Credentials and Deploy Qilin Ransomware — The Hacker News](https://thehackernews.com/2026/09/cisco-fmc-flaws-exploited-to-steal.html)
- [The Gentlemen come calling as Nutex confirms sensitive data theft — The Register](https://www.theregister.com/cyber-crime/2026/09/01/the-gentlemen-come-calling-as-nutex-confirms-sensitive-data-theft/5293642)
- [Veradigm warns of patient data breach after ransomware gang claims attack — BleepingComputer](https://www.bleepingcomputer.com/news/security/veradigm-discloses-patient-data-breach-after-gentlemen-gang-claims-attack/)
- [Clop returns with custom implant in mass extortion campaign — ReliaQuest](https://reliaquest.com/blog/clop-returns-with-custom-implant-in-mass-extortion-campaign/)
- [F5 BIG-IP APM Malware Injects a PHP Web Shell Into Memory — The Hacker News](https://thehackernews.com/2026/09/f5-big-ip-apm-malware-injects-php-web.html)
- [PaperCut NG/MF Zero-Day Active Exploitation Underway — watchTowr](https://watchtowr.com/resources/papercut-ng-mf-zero-day-cve-2026-81578-cve-2026-82078-active-exploitation-underway/)

**APTs e IA ofensiva**

- [Four Spy Groups Used the Same Chrome and Windows Exploit Kit Within a Week — The Hacker News](https://thehackernews.com/2026/09/four-spy-groups-used-same-chrome-and.html)
- [Countering misuse of AI: September 2026 — Anthropic](https://www.anthropic.com/threat-intelligence-report-september-2026)
- [Russian State-Sponsored Hackers Use Claude to Rebuild Malware After Detection — The Hacker News](https://thehackernews.com/2026/09/russian-state-sponsored-hackers-use.html)
- [Anthropic caught Russia-linked spies using Claude in hacking operations — The Record](https://therecord.media/anthropic-russia-hackers-claude)
- [Adversaries Leverage AI for Vulnerability Exploitation, Augmented Operations, and Initial Access — Google Cloud / GTIG](https://cloud.google.com/blog/topics/threat-intelligence/ai-vulnerability-exploitation-initial-access)
- [China-Linked UNC3569 Exploited Sogou Input Method — The Hacker News](https://thehackernews.com/2026/09/china-linked-unc3569-exploited-sogou.html)

**Tendencias, TTPs y regulación**

- [Weekly Recap: Rogue AI Agents, WeChat Worm, PaperCut Attacks, AI Espionage, and Rootkits — The Hacker News](https://thehackernews.com/2026/09/weekly-recap-rogue-ai-agents-wechat.html)
- [US Becomes Top Target in RMM Phishing Campaign Spanning 46 Countries — The Hacker News](https://thehackernews.com/2026/09/us-becomes-top-target-in-rmm-phishing.html)
- [Fake IT Calls Target Executives in Microsoft 365 Data Theft and Extortion Attacks — The Hacker News](https://thehackernews.com/2026/09/microsoft-365-attackers-use-help-desk.html)
- [WeChat Zero-Click Worm Took Over Accounts on iPhone and Android — The Hacker News](https://thehackernews.com/2026/09/wechat-zero-click-worm-took-over.html)
- [EU's Cyber Resilience Act starts the 24-hour vulnerability clock — The Register](https://www.theregister.com/security/2026/09/11/eus-cyber-resilience-act-starts-the-24-hour-vulnerability-clock/5295821)
- [Cybersecurity M&A Roundup: 33 Deals Announced in August 2026 — SecurityWeek](https://www.securityweek.com/cybersecurity-ma-roundup-33-deals-announced-in-august-2026/)
- [US Disrupts Xinbi Guarantee Scam Marketplace — The Hacker News](https://thehackernews.com/2026/09/us-disrupts-xinbi-guarantee-scam.html)
- [ShieldBreak Zero-Day PoC Claims — The Hacker News](https://thehackernews.com/2026/08/shieldbreak-zero-day-poc-claims.html)

---

*Informe generado el 15 de septiembre de 2026. Cobertura: 8–15 de septiembre de 2026.*

*Notas metodológicas:*

- *Los recuentos de CVEs del Patch Tuesday de septiembre varían entre 964 y 974 según la fuente, por diferencias en el criterio de contabilización (CVEs de Chromium/Edge, Mariner y componentes de terceros). Se han conservado las cifras tal como las reporta cada fuente.*
- *La campaña Cl0p/PTC Windchill se incluye como contexto de actividad en curso, no como novedad de esta semana: el grueso de las publicaciones de víctimas corresponde a agosto de 2026.*
- *En el caso de la brecha de McKesson, la cifra reclamada por el atacante (284 millones de documentos) y la confirmada por Have I Been Pwned (6,4 millones de personas) miden cosas distintas y no son directamente comparables. Se reportan ambas.*
