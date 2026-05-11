# MCP 0 to Hero — Fundamentos Teóricos

> Referencia unificada de toda la teoría del curso, sin código ni tareas prácticas.
> Cubre los conceptos, la arquitectura y los principios empresariales de los 21 módulos.

---

## 1. ¿Qué es MCP?

El **Model Context Protocol (MCP)** es un estándar abierto, propuesto por Anthropic
a finales de 2024, que define *cómo* una aplicación impulsada por un LLM se comunica
con herramientas y datos externos.

Es un protocolo de integración universal: en lugar de que cada modelo y cada producto
inventen su propio sistema de plugins, MCP proporciona un formato de conexión único.
Escribe la integración una vez, conéctala a muchos hosts.

---

## 2. Arquitectura: Los tres roles

| Rol | Descripción |
|-----|-------------|
| **Host** | La aplicación con la que el usuario habla (Claude Desktop, Droid CLI, Cursor, etc.). Es dueña de la interacción con el usuario y del acceso al modelo. |
| **Client** | Componente dentro del host que mantiene una conexión con exactamente un servidor. Un host con cinco servidores tiene cinco clients. |
| **Server** | Proceso que expone capacidades (tools, resources, prompts) y opcionalmente capacidades reversas. Es dueño de la lógica de negocio y los datos. |

**Límite de confianza**: el modelo *puede solicitar* llamadas a herramientas, pero
el servidor valida entradas, aplica políticas y registra evidencia de auditoría.
El host no debe ser tratado como fuente confiable para sanitización de entrada.

---

## 3. Primitivas del servidor

MCP define tres primitivas base:

| Prima | Quién decide usarla | Naturaleza |
|-------|-------------------|------------|
| **Tools** | El modelo (a través del host) | Acciones / verbos |
| **Resources** | El host o el usuario | Datos / sustantivos |
| **Prompts** | El usuario | Plantillas / comandos |

Además, dos capacidades reversas (servidor -> cliente):

| Capacidad | Descripción |
|-----------|-------------|
| **Sampling** | El servidor solicita al modelo del host que genere texto. El servidor no necesita API keys de LLM. |
| **Roots** | El host comunica al servidor qué directorios o URIs están dentro del alcance de la conversación. |

---

## 4. Protocolo wire (JSON-RPC 2.0)

MCP utiliza **JSON-RPC 2.0** como protocolo de mensajería sobre un transporte subyacente.

Estructura de un mensaje:
- `jsonrpc`: siempre `"2.0"`
- `id`: identificador único de solicitud
- `method`: nombre del método a invocar
- `params`: parámetros del método (opcional)

**Lifecycle de inicialización**:
1. El cliente envía `initialize` con la versión del protocolo e información del cliente.
2. El servidor responde con las capacidades que soporta e información del servidor.
3. El cliente envía `notifications/initialized`.
4. A partir de ahí, el cliente puede invocar métodos como `tools/list`, `tools/call`, `resources/list`, etc.

**Capacidades (*capabilities*)**: en la respuesta `initialize`, el servidor anuncia
qué features soporta exactamente (tools, resources, prompts, sampling, roots,
logging, completions). Un servidor solo debe anunciar lo que realmente implementa.

---

## 5. Transportes

MCP define dos transportes:

### 5.1 stdio
- El host lanza el servidor como un subproceso.
- La comunicación ocurre mediante JSON delimitado por líneas sobre stdin/stdout.
- **Cuándo usarlo**: servidores locales, herramientas de usuario, cero configuración.
- **Limitación**: solo funciona en la máquina del usuario.

### 5.2 Streamable HTTP
- El servidor es un servicio HTTP de larga duración.
- Los clients envían POST con JSON-RPC; el servidor puede transmitir respuestas
  mediante Server-Sent Events (SSE).
- **Cuándo usarlo**: servidores remotos, despliegue centralizado, multi-tenant,
  autenticación explícita.
- **Requiere**: pensar en autenticación, CORS, y despliegue.
- El estándar anterior (SSE-only) está obsoleto; Streamable HTTP es el transporte
  actual.

**Principio**: el transporte es una preocupación operativa, no una razón para
duplicar herramientas. Un mismo servidor debe poder ejecutarse sobre cualquiera
de los dos transportes.

---

## 6. Tools (Herramientas)

Una **tool** es una función que el **modelo** puede llamar. Está definida por:
1. Un **nombre** único dentro del servidor.
2. Un **esquema de entrada** (JSON Schema) que el modelo debe satisfacer.
3. Una **función de ejecución** que devuelve contenido al host.

Los type hints de Python se traducen automáticamente a JSON Schema para que el
modelo sepa cómo construir sus llamadas. El docstring de la función se convierte
en la descripción que el modelo ve.

**Manejo de errores**:
- **Errores de validación**: ocurren cuando la entrada no cumple el esquema;
  el cliente recibe un error JSON-RPC automáticamente.
- **Errores de ejecución**: el servidor lanza una excepción; deben ser mensajes
  claros y recuperables para que el modelo pueda reintentar correctamente.
  Mensajes como "failed" son insuficientes.

**Naturaleza**: las tools son **verbos**. El modelo decide cuándo invocarlas.
Deben ser comandos estrechos con validación, tiempo de ejecución acotado y
salida estructurada.

**Principio empresarial**: es preferible tener varias tools pequeñas y auditables
que una sola acción amplia. El modelo puede componer tools; el servidor debe
aplicar políticas y hacer explícitos los fallos.

---

## 7. Resources (Recursos)

Un **resource** es un dato que el **host** o el usuario puede leer. Es un
"sustantivo": representa información, no acciones.

**Direccionamiento**: los resources se identifican mediante **URI**. El servidor
elige el esquema (por ejemplo, `file://`, `notes://`, `db://`, `articles://`).

**Tipos**:
- **Static resource**: una URI fija que siempre devuelve el mismo tipo de dato
  (o se computa en el momento de lectura).
- **Templated resource**: una URI con parámetros variables que el usuario completa.

**Diferencia clave con tools**:
- Tool = verbo (hacer algo). El modelo decide cuándo.
- Resource = sustantivo (leer algo). Normalmente el host o el usuario elige.

**Naturaleza**: los resources son rutas de solo lectura. Deben ser *side-effect
free*, de tamaño acotado y seguros de llamar repetidamente.

**Principio empresarial**: los resources son para contexto durable (guías de
etiquetado, esquemas, manifiestos). Las mutaciones deben permanecer en tools
para que la aprobación y la auditoría sean explícitas.

---

## 8. Prompts (Plantillas)

Un **prompt** es una pieza de conversación predefinida que un usuario puede
insertar en el chat. Es una plantilla reutilizable con argumentos.

**Naturaleza**: los prompts son **user-driven** — el usuario los elige, no el
modelo. Son equivalentes a comandos slash o plantillas de mensaje.
- Tools = model-driven.
- Resources = data-driven.
- Prompts = user-driven.

**Estructura**: un prompt puede ser un solo mensaje de usuario o una secuencia
de múltiples mensajes (por ejemplo, un mensaje de sistema que define un rol y
un mensaje de usuario con la tarea concreta).

**Principio empresarial**: los prompts describen la tarea, el formato de salida
esperado y los recursos relevantes a consultar. No deben tener efectos secundarios
ni ocultar decisiones de política. Úsalos para estandarizar flujos humanos.

---

## 9. Cliente MCP

El **client** es el componente que inicia y gestiona una conexión con un servidor.

**Capas de un cliente**:
1. **Transporte**: proporciona flujos de lectura/escritura (stdio o Streamable HTTP).
2. **Session**: envuelve esos flujos en JSON-RPC, gestiona el handshake y las
   solicitudes/respuestas.
3. **Métodos de alto nivel**: listar tools, llamar tools, leer resources, listar
   prompts, obtener prompts, etc.

**Lifecycle**: el cliente debe realizar el handshake `initialize` antes de poder
hacer cualquier otra operación. Un cliente = una conexión a un servidor.

Aunque normalmente el host posee el cliente, entender esta capa es importante
para tres razones:
1. **Testing**: los verificadores del curso son clients.
2. **Construcción de hosts**: si escribes una app LLM propia, necesitas un client.
3. **Routers de tools**: un servidor puede ser a su vez cliente de otros servidores.

---

## 10. Context injection, Sampling y Progress

### Context injection
Cualquier tool o resource puede recibir un contexto de ejecución que le da acceso
a funcionalidades del host. El modelo no ve este parámetro; es eliminado del
esquema público automáticamente.

El contexto permite:
- Enviar mensajes de log al host en diferentes niveles (info, warning, error, debug).
- Reportar progreso de operaciones largas.
- Leer recursos del propio servidor.
- Solicitar sampling al modelo del host.

### Sampling
El servidor puede pedirle al **LLM del host** que genere texto. Esto convierte
al servidor en un agente con capacidad de generar lenguaje, sin necesidad de
API keys propias ni vendor lock-in.

**Naturaleza**: la salida de sampling debe tratarse como **texto de borrador no
confiable**, no como datos aprobados para producción.

**Requisito**: el host debe soportar explícitamente sampling. No todos los hosts
lo implementan.

### Progress notifications
Para herramientas de larga duración, el servidor puede notificar al host el
progreso (paso actual / total). Esto mejora la experiencia de usuario y permite
al host gestionar timeouts.

**Principio empresarial**: sampling es potente pero no es un mecanismo de
aprobación. Úsalo para borradores; luego valida y controla cualquier mutación
sensible. Mantén fallbacks deterministas para hosts que no soporten sampling.

---

## 11. Protocol Internals

Por debajo de FastMCP, el protocolo MCP tiene conceptos que es importante
entender para depuración empresarial:

**Initialize**: negocia versión del protocolo, información del servidor y
capacidades. Un servidor debe anunciar exactamente las capacidades que soporta.

**JSON-RPC 2.0**: define campos obligatorios (`jsonrpc`, `id`, `method`) y
opcional (`params`). Las solicitudes sin `params` no deben omitir el campo;
deben enviarlo vacío para evitar ambigüedad en los clients.

**Errores estructurados**: los códigos de error permiten al host distinguir
entre fallos de validación, timeout y errores desconocidos, para decidir si
reintentar o no.

**Principio empresarial**: la mayoría del código debe usar FastMCP. Se baja al
nivel de protocolo cuando se depuran problemas de integración, se construyen
gateways, se revisan logs o se deciden tradeoffs de transporte y compatibilidad.

---

## 12. Modelo de seguridad

### Conceptos fundamentales

**Mínimo privilegio**: una tool solo debe realizar aquello para lo que el
llamante tiene permiso explícito.

**Validación en servidor**: el host y el modelo no son fuentes confiables para
sanitizar entrada. Toda validación debe ocurrir en el servidor.

**Inyección de prompt**: texto controlado por un atacante que intenta anular
las instrucciones del sistema. Patrones comunes incluyen "ignore previous
instructions", "system prompt", "developer message".

**Redacción de secretos**: las API keys, tokens y contraseñas nunca deben
filtrarse a logs, respuestas de tools o campos de elicitation.

**Auditoría**: cada decisión de autorización debe registrarse con actor, acción,
resultado (permitido/denegado) y motivo.

### Separación de responsabilidades
- La capa de políticas está separada de las tools FastMCP para que pueda ser
  testeada de forma unitaria sin levantar un servidor.
- Las reglas de validación, redacción y autorización viven en módulos
  independientes y reutilizables.

### Tipos de acciones
- Acciones **no sensibles**: pueden ejecutarse sin aprobación explícita.
- Acciones **sensibles** (publicar guías, borrar datasets, cambiar políticas):
  requieren aprobación explícita.

**Principio empresarial**: esta capa vive **dentro** de cada tool. En
producción, considera integrar con un motor de políticas externo.

---

## 13. Roots y alcance del filesystem

### Qué son los roots
Los **roots** son sugerencias del host sobre directorios o URIs relevantes
para la conversación actual. El host los envía al servidor durante la
inicialización.

### Naturaleza
Los roots son **orientativos**, no constitutivos de un límite de seguridad.
El servidor **no debe confiar** en que el host ha validado las rutas.

### Responsabilidad del servidor
1. Almacenar los roots recibidos, resolviendo cada uno a una ruta absoluta.
2. Por cada solicitud de acceso a archivo:
   - Resolver la ruta solicitada a su forma absoluta real.
   - Expandir `~` y resolver enlaces simbólicos.
   - Comparar la ruta resuelta contra los roots almacenados.
   - Rechazar con `PermissionError` si está fuera de alcance.

### Path traversal
El ataque `../../etc/passwd` debe defenderse mediante resolución de rutas
antes de la comparación, no mediante coincidencia de prefijos de string.

**Principio empresarial**: el servidor siempre aplica validación de rutas.
Nunca asumas que el host ha validado por ti. En producción, añade listas
blancas de tipos de archivo, límites de tamaño y auditoría en cada lectura.

---

## 14. Elicitation y humano en el circuito

### Qué es
La **elicitación** es una solicitud estructurada del servidor al host para
obtener entrada del usuario. No es un chat libre ni un mecanismo genérico de
entrada.

### Estructura
El servidor construye una solicitud con:
- `title`: título de la solicitud.
- `description`: descripción del contexto.
- `fields`: campos estructurados que el usuario debe completar (con tipo, rango,
  etc.).

### Outcomes posibles
| Outcome | Significado |
|---------|-------------|
| **Accept** | El usuario aceptó y proporcionó datos válidos. |
| **Decline** | El usuario declinó responder. |
| **Cancel** | El usuario canceló la operación. |

Cada outcome debe ser manejado explícitamente por el servidor.

### Restricciones
Los campos de elicitation **nunca** deben usarse para recolectar contraseñas,
API keys o tokens. Si un flujo requiere secretos, deben obtenerse de un vault
externo.

**Principio empresarial**: la elicitación es para **decisiones de negocio**
(clasificar una página como allow/review/block), no para autenticación. El
servidor valida la respuesta del usuario; no asume que el host la ha validado.

---

## 15. Paginación y Completion

### Paginación
Cuando una lista puede crecer más allá de unos pocos elementos, es necesario
devolver los resultados en páginas.

**Cursor**: token opaco que codifica el estado de la paginación (por ejemplo,
un offset codificado en base64). El cliente no debe poder manipularlo ni
adivinar su significado.

**Contrato**:
- El servidor recibe un cursor opcional y un límite.
- Devuelve un segmento de datos y un `nextCursor`.
- Cuando no hay más elementos, `nextCursor` es `None`.
- El límite debe estar acotado superiormente.

### Completion
El servidor puede sugerir valores para argumentos de prompts o URIs de
resources basándose en un prefijo que el usuario o modelo ha escrito.

**Contrato**:
- Recibe un prefijo y una lista de candidatos.
- Devuelve los candidatos que comienzan con el prefijo.
- El resultado debe estar acotado.

**Principio empresarial**: la paginación es esencial para listas que superan
unas docenas de elementos. Los cursores deben ser opacos; no expongas claves
primarias de base de datos como cursores.

---

## 16. HTTP Auth

### Bearer token
Mecanismo de autenticación donde el cliente presenta un secreto en el header
`Authorization: Bearer <token>`.

### Scopes
Permisos finos asociados a un token (por ejemplo, `labels:read`, `labels:write`).
No todos los tokens tienen todos los scopes; el servidor verifica por acción,
no solo por sesión.

### Protected resource metadata
Concepto de OAuth 2.0: el servidor de recursos publica metadatos que describen
cómo los clients deben obtener autorización. Estos metadatos incluyen:
- La URL del servidor de autorización.
- Los scopes soportados.

### Alcance
El servidor MCP valida tokens y verifica scopes, pero **no implementa** un
servidor de autorización OAuth completo (no hay flujo de login, no emite
tokens).

**Principio empresarial**: en producción, delega la validación de tokens al
reverse proxy o API gateway, y pasa los claims validados al servidor MCP.
Nunca almacenes secretos de producción en código fuente.

---

## 17. Flujos agentivos (Agentic Workflows)

### Estructura
Un flujo agentivo se modela como una secuencia de pasos explícitos y nombrados:

1. **Planificación**: definir los pasos necesarios.
2. **Ejecución**: sampling para generar borradores, tools para acciones estructuradas.
3. **Aprobación**: gate humano antes de mutar estado sensible.
4. **Progreso**: fracción de pasos completados.

### Componentes
- **Step**: unidad atómica del flujo, con nombre y estado (pending, done, blocked).
- **Approval gate**: punto de control donde se requiere decisión humana antes
  de continuar.
- **Progress**: métrica de pasos completados sobre total.

### Principios
- Los borradores se generan con sampling, pero las decisiones finales requieren
  herramientas explícitas y aprobación.
- El modelo no debe mutar guías sin un checkpoint de aprobación.
- Cada paso debe ser observable (nombre, estado, progreso) para el host.

**Principio empresarial**: ningún bucle autónomo debe mutar datos de producción
sin un gate humano. Sampling para borradores, tools para acciones estructuradas,
elicitación para aprobaciones. Cada paso se registra con un correlation ID.

---

## 18. Testing y CI

### Tests hermeticos
Son tests sin dependencias externas: sin red, sin secretos reales, sin
llamadas a LLMs reales, sin servicios alojados. Son el estándar para
servidores MCP de producción.

### Categorías de tests
| Tipo | Qué prueba |
|------|------------|
| **Unit tests** | Funciones individuales (políticas, validación, lógica de negocio). |
| **verify.py** | Criterios de aceptación del ejercicio/herramienta. |
| **Fakes** | Simulaciones ligeras que reemplazan al host (sampling, filesystem, HTTP). |
| **Integration tests** | Servidor completo sobre stdio con componentes falsos. |

### CI Pipeline
Un pipeline de integración continua debe reflejar la verificación local:
instalar dependencias, ejecutar linter, ejecutar tests, verificar soluciones,
compilar.

**Principio empresarial**: un test inestable que depende de latencia de red o
no-determinismo del modelo destruye la confianza del equipo. Invierte en fakes
temprano.

---

## 19. Observabilidad y operaciones

### Correlation ID
Identificador único que se propaga a través de todas las capas de una solicitud
(desde la entrada hasta la salida) para poder unir logs, métricas y eventos de
auditoría en sistemas distribuidos.

### Logs estructurados
Registros en formato máquina-parseable (JSON o clave-valor), no texto libre.
Cada entrada incluye nivel, evento, correlation ID, timestamp y campos
adicionales.

### Métricas
Mediciones de rendimiento y errores:
- **Timers**: duración de operaciones, con estado (ok/error) incluso cuando
  ocurre una excepción.
- **Contadores**: número de invocaciones por tool, por actor.
- **Indicadores**: tasas de error, latencia.

### Runbook
Guía legible por humanos para incidentes comunes: qué comprobar, cómo
recuperarse, a quién escalar.

**Principio empresarial**: la observabilidad se diseña desde la primera tool,
no se añade después. En producción, los logs se envían a un agregador y se
alerta sobre umbrales de error.

---

## 20. Plantilla de producción

### Settings centralizados
Toda la configuración del servidor reside en un objeto único, tipado, con
valores por defecto. Se carga desde variables de entorno (o desde un dict para
tests). Los módulos individuales no leen `os.environ` directamente.

### Validación al arranque (deployment checks)
Antes de que el servidor acepte conexiones, se ejecutan validaciones que
verifican:
- El transporte es soportado.
- Si se usa HTTP, hay un token de autenticación configurado.
- Cualquier otra dependencia o requisito de configuración.

Si alguna validación falla, el servidor falla rápido con mensajes claros y
código de salida no cero.

### Inmutabilidad
Después de la carga, la configuración no debe modificarse en tiempo de
ejecución.

### Docker
El servidor se empaqueta en una imagen Docker mínima y reproducible.

**Principio empresarial**: un servidor que arranca con mala configuración es un
riesgo operacional. Validar temprano, fallar rápido y con mensajes claros.

---

## 21. Capstone Enterprise (plantilla final)

La plantilla `enterprise-labeling-mcp` combina todos los conceptos del curso en
un servidor de producción que demuestra:

- **Recursos**: guías de etiquetado como resources.
- **Prompts**: evaluación de páginas con instrucciones reutilizables.
- **Tools**: clasificación, carga de datasets, cambios de guías.
- **Gates humanos**: aprobación explícita para cambios sensibles.
- **Elicitación**: solicitudes estructuradas con outcomes accept/decline/cancel.
- **Sampling**: borradores de recomendación generados por el host.
- **Roots**: acceso seguro a datasets locales con validación de rutas.
- **Auditoría**: logs estructurados con correlation IDs.
- **Autenticación**: bearer tokens, scopes, metadata OAuth.
- **Transportes**: soporte para stdio y Streamable HTTP.
- **Despliegue**: Docker, tests, runbook, threat model.

---

## 22. Principios empresariales transversales

| Principio | Descripción |
|-----------|-------------|
| **Mínimo privilegio** | Cada tool solo debe hacer lo que el llamante tiene permitido explícitamente. |
| **Validación en servidor** | El host y el modelo no son confiables para sanitizar entrada. Toda validación ocurre en el servidor. |
| **Auditoría** | Cada decisión de autorización se registra con actor, acción, resultado y motivo. |
| **No secretos en código** | Las credenciales van en variables de entorno o vaults externos, nunca en el código fuente. |
| **Tests hermeticos** | Sin red, sin LLMs reales, sin secretos. Tests deterministas y repetibles. |
| **Fail fast** | La configuración inválida se detecta al arrancar, no en la primera solicitud. |
| **Observabilidad desde el diseño** | Correlation IDs, logs estructurados y métricas desde la primera tool. |
| **Gates humanos** | Las mutaciones de estado sensible requieren aprobación humana explícita. |
| **Roots no es seguridad** | Los roots son sugerencias del host; el servidor siempre valida rutas por su cuenta. |
| **Sampling no es aprobación** | El sampling genera borradores no confiables. Las decisiones finales requieren tools y validación. |
| **Transporte según contexto** | stdio para herramientas locales; Streamable HTTP para despliegue centralizado con auth. |
| **Configuración única e inmutable** | Un objeto central tipado; los módulos no leen variables de entorno directamente. |
| **Elección de scope** | Un servidor pequeño con tests y límites claros es mejor que uno grande que no se puede auditar. |

---

## Referencias teóricas

- [Especificación MCP](https://modelcontextprotocol.io/specification/latest)
- [SDK Python MCP](https://py.sdk.modelcontextprotocol.io/)
- [Streamable HTTP transport](https://modelcontextprotocol.io/specification/latest/basic/transports)
- [Autorización y protected resource metadata](https://modelcontextprotocol.io/specification/latest/basic/authorization)
