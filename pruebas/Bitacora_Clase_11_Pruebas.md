---
tipo: bitacora-clase
curso: Automatización inteligente de procesos con n8n
periodo: EAM-2026II
equipo: Betancourth · Castaño · Castañeda · Albornoz
corte: 3
semana: 11
clase: 11
fecha: 2026-11-09
actualizado: 2026-11-22
estado: entrega-final
tags:
  - n8n
  - bitacora
  - confiabilidad
  - revision-integral
---

# Bitácora de la clase 11

## Revisión integral del proyecto y pendientes para la sustentación

**Curso:** Automatización inteligente con n8n
**Tema:** Pruebas y confiabilidad de flujos con IA

> **Versión de entrega del corte final.** Las secciones 1 a 8 son la revisión del diseño
> tal como se escribió el 2026-11-09, conservadas sin modificar para que se vea la
> evolución del trabajo. Las secciones 9 a 11, añadidas el 2026-11-22, recogen las pruebas
> que sí se ejecutaron, las situaciones nuevas que esas pruebas destaparon y el estado
> final de los pendientes.
>
> Un caso que no se ha ejecutado figura como no ejecutado. Ninguno se da por aprobado sin
> evidencia.
>
> **Nota sobre los nombres.** La revisión original nombra cinco workflows separados
> (`Clase07_WebhookMovimientoInventario`, `Resumen_Semanal`, `Agente_Consulta_Inventario`,
> `Sub_EnviarAlertaStock` y `Sub_ResumenMovimientos`). En la entrega final todos se
> fusionaron en `workflows/Workflow_Final.json`, porque la guía exige un único archivo sin
> referencias a workflows ausentes. Las etapas y los nombres de nodo se conservan; lo que
> cambió es que ahora viven en el mismo archivo. La correspondencia está en la sección 10.

## 1. Identificación

- **Fecha de la clase:** 2026-11-09
- **Proyecto:** Alerta de inventario — EAM 2026-II
- **Integrantes presentes:** Betancourth, Castaño, Castañeda, Albornoz
- **Versión revisada:** instancia n8n 2.35.7 autoalojada.
  - **Punto de partida de la revisión:** `repo/AlertaInventario_EAM2026II/workflows/` (26 nodos en el webhook).
  - **Versión resultante tras aplicar las soluciones de la sección 8:** `02_Workflows/v2_Corregidos_C11/` (36 nodos en el webhook), con su propio `LEEME.md`.
  - **Advertencia de versiones:** `02_Workflows/Clase07_WebhookMovimientoInventario.json` es una copia **anterior** a la corrección del Corte 2: conserva los siete nodos `Set` en `typeVersion: 1` con parámetros en formato v3 y las referencias `$('Nodo').item`. Se conserva como evidencia histórica del fallo y **no debe importarse**. Ver RC-22.
- **Workflows relacionados:**

| Workflow | Función | Estado |
|---|---|---|
| `Clase07_WebhookMovimientoInventario.json` | Entrada por webhook, descuento de stock, alerta y registro del movimiento | Vigente |
| `Sub_EnviarAlertaStock.json` | Subworkflow de envío de la alerta por Gmail | Vigente |
| `Manejador_Errores.json` | Error Trigger: registra el fallo en `Errores` y avisa al equipo | Vigente, **no enlazado** (ver RC-01) |
| `Sub_ResumenMovimientos.json` | Consulta y agregación de movimientos por periodo. Solo lectura | Vigente |
| `Resumen_Semanal.json` | Informe semanal redactado por IA y enviado por correo | Vigente |
| `Agente_Consulta_Inventario.json` | Agente de chat con memoria que consulta periodos | Vigente |
| `AlertaInventarioScheduleGmail.json` | Revisión programada previa al webhook | Histórico |
| `AlertaInventarioScheduleTimer.json`, `01_AlertaInventario_v1/v2/v3.json` | Versiones anteriores | Histórico |

- **Diagrama revisado:** `04_Evidencias/Diagrama_Proceso.md` y los diagramas `Proceso_Manual.svg` / `Proceso_Automatizado.svg` del repositorio.

## 2. Recorrido completo del proyecto

**Descripción del recorrido:**

El proyecto tiene tres entradas distintas y un mismo origen de datos.

La **entrada principal** es un webhook `POST /inventario-movimiento` protegido con autenticación por cabecera. Recibe un movimiento de inventario (`movimiento_id`, `sku`, `cantidad`, `motivo`). El flujo valida el contrato, comprueba en la tabla `Movimientos` si ese `movimiento_id` ya fue procesado, busca el producto en `Productos`, calcula el stock resultante, aplica el descuento con un `PATCH`, y según si el stock queda en el mínimo o por debajo envía o no una alerta por Gmail. Finalmente registra el movimiento con un `POST` y responde al llamante con el detalle de la operación. Hay cuatro respuestas posibles: 200 normal, 200 con `duplicado: true`, 400 por contrato inválido, 404 por SKU inexistente y 502 por fallo de escritura.

La **segunda entrada** es un disparador programado los lunes a las 07:00 (`Resumen_Semanal`). Calcula el lunes y el domingo de la semana cerrada, pide las cifras al subworkflow `Sub_ResumenMovimientos`, y si hay datos los entrega a una cadena de IA con salida estructurada que **redacta** el informe. El correo incluye el texto redactado y, debajo, las cifras crudas calculadas por el flujo.

La **tercera entrada** es un chat (`Agente_Consulta_Inventario`). El agente dispone de una única herramienta, `resumen_movimientos`, que es el mismo subworkflow y es de solo lectura. La memoria conserva los últimos turnos de la conversación.

La IA interviene en dos puntos y **en ninguno calcula**: redacta cifras ya calculadas por código, o traduce una pregunta en un rango de fechas que el subworkflow valida antes de consultar.

| Etapa | Workflow y nodo o tramo | Qué debe garantizar | Destino o siguiente etapa |
|---|---|---|---|
| Recepción de la solicitud | Webhook: `Webhook Movimiento` (headerAuth, `responseMode: responseNode`) / `Lunes por la manana` / `Chat de consulta` | Que solo entren solicitudes autenticadas y que toda ejecución termine en una respuesta explícita | `Validar contrato` / `Calcular la semana anterior` / `Validar turno y sesion` |
| Validación y preparación de datos | Webhook: `Validar contrato`. Sub: `Validar periodo`. Agente: `Validar turno y sesion` | Rechazar antes de tocar la base: campos vacíos, cantidad no positiva, fechas mal formadas, rango invertido o mayor de 400 días, entrada vacía o excesiva | `Buscar movimiento` / `Consultar movimientos` / `Agente de consulta`, o la salida de rechazo |
| Consulta de fuentes | `Buscar movimiento`, `Buscar producto`, `Consultar movimientos` (todos GET a PostgREST) | Distinguir «no existe» de «no se pudo consultar» | `Ya procesado?`, `Producto existe?`, `Agregar resultados` |
| Interpretación o respuesta de IA | `Redactar resumen` (chainLlm + `Contrato del informe`) y `Agente de consulta` (+ memoria + herramienta) | Conservar las cifras recibidas sin recalcularlas; devolver la estructura acordada; no inventar datos ausentes | `Armar el correo` / `Preparar respuesta` |
| Decisiones y reglas del negocio | `Ya procesado?`, `Producto existe?`, `Bajo el minimo?` (`stockNuevo <= stockMinimo`), `Hay cifras?` | Aplicar la regla correcta a los datos disponibles, incluido el caso límite de igualdad | Rama de alerta o rama sin alerta |
| Operaciones en servicios externos | `Actualizar stock` (PATCH `Productos`), `Registrar movimiento` (POST `Movimientos`), `Gmail Avisar`, `Enviar informe` | Que el destino contenga realmente el cambio y que la confirmación lo describa | `Escritura confirmada?` → `Cuerpo 200` / `Cuerpo 502` |
| Respuesta y efecto final | `Respond 200`, `Respond 400`, `Respond 404`, `Respond 502`, correo semanal, respuesta del chat | Que la respuesta corresponda al efecto real producido | Fin de la ejecución |
| Tratamiento de fallos y recuperación | `Escritura confirmada?`, `Resumen sin redaccion`, `Aviso sin cifras`, `Fallo del agente`, `Manejador_Errores`, `retryOnFail` en las escrituras | Conservar la solicitud, informar el estado y no afirmar un efecto que no ocurrió | Respuesta de error o informe degradado |

## 3. Revisión de situaciones por etapa

### Registro de la revisión

| ID | Etapa y situación concreta | Resultado esperado y efecto que debe evitarse | Protección actual y ubicación | Estado de revisión y justificación |
|---|---|---|---|---|
| RC-01 | Tratamiento de fallos: cualquier workflow falla con una excepción no controlada | El fallo debe quedar registrado en `Errores` y avisar al equipo. Evitar que un fallo pase inadvertido | `Manejador_Errores.json` existe y está completo: `Error Trigger` → `Resumir el fallo` → `Registrar el fallo` (POST a `Errores`, `onError: continueRegularOutput`) → `Avisar al equipo` | **Brecha identificada.** El campo `settings.errorWorkflow` está vacío en los **once** workflows exportados. El manejador existe pero ningún workflow lo tiene asignado, así que nunca se dispara solo. La protección está construida y desconectada |
| RC-02 | Operaciones externas: el `PATCH` a `Productos` responde correctamente pero no modifica ninguna fila | El flujo debe detectar que no hubo escritura y responder 502. Evitar una confirmación falsa y un movimiento registrado sin descuento real | `Escritura confirmada?` comprueba `statusCode < 300` → rama falsa a `Cuerpo 502` | **Brecha identificada.** La petición no envía `Prefer: return=representation`, así que PostgREST responde 204 sin cuerpo aunque el filtro no haya coincidido con ninguna fila. El 204 pasa la condición. Es el mismo fallo documentado en `04_Evidencias/Bitacora5/19_respuesta_200_enganosa.png` |
| RC-03 | Interrupción después de completar solo una parte: el `PATCH` tiene éxito y el `POST` a `Movimientos` falla | El stock y el registro deben quedar coherentes, o la incoherencia debe quedar localizable | `retryOnFail` activo en `Registrar movimiento`. No hay compensación ni registro del tramo intermedio | **Brecha identificada.** El descuento ocurre antes del registro. Si el `POST` falla tras agotar reintentos, el stock ya bajó y no queda rastro. Como `Ya procesado?` consulta `Movimientos`, un reenvío del mismo `movimiento_id` **vuelve a descontar**. La idempotencia no cubre este tramo |
| RC-04 | Solicitud repetida: llega dos veces el mismo `movimiento_id` ya completado | Responder 200 con `duplicado: true`, sin descontar de nuevo | `Buscar movimiento` → `Ya procesado?` (`body.length > 0`) → `Cuerpo duplicado` → `Respond 200`. Restricción `UNIQUE` sobre `movimiento_id` en Supabase | **Protección localizada.** Cubre el caso completado. No cubre el caso parcial de RC-03. Requiere comprobación |
| RC-05 | Consulta de fuentes: Supabase no está disponible al buscar el producto | Responder un fallo técnico (502 o equivalente). Evitar afirmar que el producto no existe | `Buscar producto` usa `neverError: true` y no se comprueba su `statusCode`. `Producto existe?` evalúa `$json.body.length > 0` | **Brecha identificada.** Con la fuente caída, `body` no es un array y la condición da falso, por lo que el flujo responde **404 «el SKU no existe»**. Confunde un fallo técnico con un resultado negativo válido. Ocurrió realmente el 2026-11-09: el proyecto de Supabase estaba pausado y el host devolvía `ENOTFOUND` |
| RC-06 | Consulta de fuentes: Supabase no responde al comprobar el duplicado | Detener o reportar. Evitar tratar la solicitud como nueva | `Buscar movimiento` con `neverError: true`; `Ya procesado?` evalúa `body.length > 0` | **Brecha identificada.** Mismo mecanismo que RC-05: la condición da falso y el flujo continúa como si no fuera duplicado, con riesgo de doble descuento |
| RC-07 | Datos problemáticos: `cantidad` muy superior al stock disponible | Rechazar por regla de negocio o dejar constancia del faltante. Evitar confirmar un despacho imposible | `Calcular stock` aplica `Math.max(0, stockActual - cantidad)`, que impide stock negativo | **Brecha identificada.** El tope protege la base pero no la regla: una `cantidad` de 9999 sobre 15 unidades responde 200 «stock actualizado 15 → 0». No existe rechazo por cantidad superior al disponible |
| RC-08 | Datos problemáticos: `motivo` ausente o con un valor fuera de las cuatro opciones del formulario | Normalizar o rechazar según la regla acordada | `Calcular stock` aplica `motivo \|\| 'sin motivo'` | **Brecha identificada.** El webhook acepta cualquier cadena como `motivo`; solo el formulario del simulador limita las opciones. El dato llega sin validar a `Movimientos` y de ahí a la agregación por motivo del resumen |
| RC-09 | Valores en el límite declarado: `stockNuevo` exactamente igual a `stockMinimo` | Debe enviarse alerta (criterio `lte` acordado en la Clase 03) | `Bajo el minimo?` usa `lte` sobre `stockNuevo` y `stockMinimo` | **Protección localizada.** Verificada en el Corte 2 con GAL 007 (15 → 13 con mínimo 13). Requiere repetición en la sustentación |
| RC-10 | Operación normal con datos válidos en el webhook | Descuento aplicado, movimiento registrado y respuesta coherente con el efecto | Cadena completa `Validar contrato` → … → `Respond 200` | **Protección localizada.** Comprobada en el Corte 2 contra la URL de producción. Depende de RC-02 para poder afirmarse con certeza |
| RC-11 | Resultado negativo válido: periodo sin movimientos en el resumen semanal | El correo debe decir que la consulta funcionó y no hubo movimientos | `Agregar resultados` devuelve `estado: 'sin_movimientos'`; `Aviso sin cifras` redacta un texto distinto según el estado y marca `requiere_atencion: false` | **Protección localizada.** El nodo distingue explícitamente «no hubo movimientos» de «no se pudo consultar». Requiere comprobación |
| RC-12 | IA o servicio externo no disponible: Gemini falla o devuelve algo que el parser no admite | El informe debe salir igual, con las cifras y diciendo que no hubo redacción | `Redactar resumen` con `onError: continueErrorOutput` → `Resumen sin redaccion`, que arma el correo con las cifras y el motivo técnico | **Protección localizada.** Requiere comprobación provocando el fallo con una credencial inválida sobre una copia |
| RC-13 | Calidad de la respuesta de IA: fidelidad de las cifras | El texto debe reproducir exactamente las cifras recibidas. Evitar que el modelo sume, estime o redondee | Instrucciones explícitas de la cadena («recibes cifras ya calculadas, no recalcules»), salida estructurada con `Contrato del informe`, y las cifras crudas adjuntas al final del correo bajo un separador | **Protección localizada.** El diseño permite contrastar texto y cifras en el mismo correo, pero la fidelidad del texto **no está comprobada** |
| RC-14 | Agente: la memoria presenta cifras de un turno anterior como vigentes | Cada pregunta debe resolverse consultando la herramienta. Evitar reutilizar números antiguos | Instrucciones del agente («no reutilices números de la conversación como si fueran actuales»), `contextWindowLength: 5` | **Brecha identificada.** La instrucción es la única protección; no hay mecanismo que obligue a invocar la herramienta. Comportamiento no comprobado |
| RC-15 | Agente: el modelo traduce mal una expresión relativa («esta semana») | Debe resolver el rango contra la fecha real de hoy | El mensaje de sistema inyecta `{{ $now.format('yyyy-MM-dd') }}` y el día de la semana; `Validar periodo` rechaza formatos y rangos inválidos | **Protección localizada.** Requiere comprobación con expresiones relativas y con un rango inverso |
| RC-16 | Límites de actuación: se pide al agente modificar inventario | Debe negarse. Evitar cualquier escritura desde el chat | La única herramienta conectada, `resumen_movimientos`, ejecuta `Sub_ResumenMovimientos`, que solo hace un GET. El agente no tiene herramienta de escritura | **Protección localizada.** La restricción es estructural, no solo de instrucciones. Requiere comprobación con una petición explícita de modificar |
| RC-17 | Límites de información: dos conversaciones distintas en el chat | Los datos de una sesión no deben aparecer en otra | `sessionKey` compuesta: `'corte3-inventario:' + sessionId`; `Validar turno y sesion` exige `sessionId` no vacío | **Protección localizada.** Requiere comprobación con dos sesiones simultáneas |
| RC-18 | Acción no permitida: llamada al webhook sin credencial | Rechazo por autenticación antes de ejecutar nada | `Webhook Movimiento` con `authentication: headerAuth` | **Protección localizada.** Verificada en el Corte 2 (respuesta 403). `allowedOrigins` está en `*`, lo que no afecta a la autenticación pero conviene acotar |
| RC-19 | Fallo o interrupción: Gmail falla al enviar la alerta | El movimiento debe registrarse igual y `alerta_enviada` debe reflejar lo ocurrido | `Gmail Avisar` se ejecuta **antes** de `Registrar movimiento (alerta)`, que escribe `alerta_enviada: true` de forma fija | **Brecha identificada.** Si Gmail falla, no se registra el movimiento y el stock ya se descontó (mismo efecto que RC-03). Además el campo afirma un envío que no se comprueba |
| RC-20 | Estabilidad: comparar el comportamiento entre versiones del workflow | Poder repetir un caso con la misma entrada y estado inicial | `02_Workflows/` conserva las versiones históricas; el repositorio tiene historial de git | **Brecha identificada.** Existen **dos copias del repositorio** (`D:\UVirtual\…` y `repo/` dentro de la carpeta compartida) que pueden divergir. No hay una versión declarada como vigente dentro de los JSON |
| RC-21 | Datos de prueba y evidencias compartidas | Las evidencias no deben exponer secretos ni datos personales reales | Los movimientos de prueba llevan el prefijo `SIM-`; los JSON exportados solo contienen referencias de credencial (id y nombre), sin claves | **Protección localizada.** Comprobado por revisión de los tres JSON del Corte 3: sin JWT, sin claves de API, sin tokens |
| RC-22 | Estabilidad: dos copias del mismo workflow conviven en la carpeta compartida | La versión que se importe debe ser la corregida | Ninguna. Ambos archivos están en `02_Workflows/` sin distintivo de vigencia | **Brecha identificada, confirmada durante la revisión.** `Clase07_WebhookMovimientoInventario.json` conserva los siete `Set` rotos del Corte 2; `repo/…/Webhook_MovimientoInventario.json` está corregido. Importar el primero reintroduce el fallo que dejó el stock sin descontar durante todo el corte. Es RC-20 dejando de ser una hipótesis |
| RC-23 | Consistencia entre pasos: tipos de los campos que recibe un subworkflow | Un valor numérico debe llegar al subworkflow como número | `Entrada del subworkflow` de `Sub_EnviarAlertaStock` declaraba los diez campos sin tipo, y n8n los toma como `string` por omisión | **Brecha identificada y corregida durante la clase.** Al ejecutar la rama de alerta, n8n rechazó la llamada: *«Invalid input for 'stockActual': expects a string but we got '1'»*. El dato importante no es el error sino lo que revela: **la rama de alerta por subworkflow nunca se había ejecutado**. El Corte 2 se probó con Gmail en línea, y al sustituirlo por el subworkflow el tramo quedó en el lienzo sin pasar nunca por una ejecución real. Ver S-10 |

**Etapas o situaciones que todavía faltan por revisar:**

- La rama histórica `AlertaInventarioScheduleGmail` sigue en la carpeta y consulta Supabase de forma programada. No se revisó si está activa en la instancia; si lo estuviera, podría enviar alertas en paralelo al webhook.
- No se revisó el comportamiento ante varios movimientos concurrentes sobre el mismo SKU. El cálculo `stockActual - cantidad` se hace sobre una lectura previa, sin control de concurrencia.
- No se revisó la zona horaria: `settings.timezone` está sin definir en todos los workflows, y tanto el corte semanal como `$now` dependen de ella.

## 4. Casos previstos para comprobar el comportamiento

### Caso CP-01: El PATCH no modifica ninguna fila

- **Revisión asociada:** RC-02
- **Workflow y etapas involucradas:** `Clase07_WebhookMovimientoInventario`, de `Actualizar stock` a `Respond 200` / `Respond 502`
- **Qué se busca comprobar:** si el flujo distingue una escritura efectiva de una escritura que no tocó ninguna fila
- **Condiciones iniciales:** copia de pruebas de la tabla `Productos`. Producto `SIM-TEST-001` presente con `stockActual` 20 y `stockMinimo` 5
- **Entrada exacta:** `POST /inventario-movimiento` con `{"movimiento_id":"SIM-CP01","sku":"SIM-TEST-001","cantidad":2,"motivo":"venta"}`, forzando que el filtro del PATCH no coincida (por ejemplo, renombrando el SKU en la tabla justo antes)
- **Resultado esperado:** respuesta 502 y ningún registro nuevo en `Movimientos`
- **Criterio de aceptación:** el código de respuesta es 502 **y** `select count(*) from "Movimientos" where movimiento_id = 'SIM-CP01'` devuelve 0
- **Efecto que no debe ocurrir:** respuesta 200 afirmando un descuento que no ocurrió
- **Evidencia que se buscará:** ejecución en n8n, salida del nodo `Actualizar stock` con su `statusCode`, y conteo en `Movimientos` antes y después
- **Cómo se preparará de forma controlada:** sobre datos con prefijo `SIM-`, sin tocar productos reales
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-02: Fuente no disponible durante la búsqueda del producto

- **Revisión asociada:** RC-05, RC-06
- **Workflow y etapas involucradas:** `Clase07_WebhookMovimientoInventario`, de `Buscar producto` a la respuesta
- **Qué se busca comprobar:** si una caída de Supabase se reporta como fallo técnico o se confunde con un SKU inexistente
- **Condiciones iniciales:** credencial de Supabase apuntando a un host inalcanzable en una copia del workflow
- **Entrada exacta:** `POST /inventario-movimiento` con `{"movimiento_id":"SIM-CP02","sku":"GAL 007","cantidad":1,"motivo":"venta"}`, siendo `GAL 007` un SKU que sí existe
- **Resultado esperado:** respuesta de fallo técnico (502) indicando que no se pudo consultar
- **Criterio de aceptación:** la respuesta no es 404 y el mensaje no afirma que el producto no existe
- **Efecto que no debe ocurrir:** responder 404 sobre un SKU que sí existe
- **Evidencia que se buscará:** ejecución, salida del nodo `Buscar producto` y cuerpo de la respuesta
- **Cómo se preparará de forma controlada:** copia del workflow con credencial alterada; nunca sobre el workflow en producción
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-03: Interrupción entre el descuento y el registro

- **Revisión asociada:** RC-03, RC-04
- **Workflow y etapas involucradas:** `Actualizar stock` → `Registrar movimiento` → reenvío de la misma solicitud
- **Qué se busca comprobar:** si una solicitud interrumpida a medias permite un doble descuento al reintentarla
- **Condiciones iniciales:** `SIM-TEST-002` con `stockActual` 10. Copia del workflow donde `Registrar movimiento` apunta a una tabla inexistente para forzar el fallo
- **Entrada exacta:** `POST` con `{"movimiento_id":"SIM-CP03","sku":"SIM-TEST-002","cantidad":3,"motivo":"venta"}` y, tras el fallo, **el mismo cuerpo otra vez**
- **Resultado esperado:** el stock final no debe ser 4. O bien el primer intento no descuenta, o bien el segundo detecta el estado parcial
- **Criterio de aceptación:** `stockActual` de `SIM-TEST-002` tras los dos envíos es 10 o 7, nunca 4
- **Efecto que no debe ocurrir:** dos descuentos por una sola operación de negocio
- **Evidencia que se buscará:** valor de `stockActual` antes, entre los dos envíos y al final; las dos ejecuciones
- **Cómo se preparará de forma controlada:** copia del workflow y productos con prefijo `SIM-`
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-04: Cantidad superior al stock disponible

- **Revisión asociada:** RC-07
- **Workflow y etapas involucradas:** `Validar contrato` → `Calcular stock` → `Respond 200`
- **Qué se busca comprobar:** qué responde el sistema ante un despacho imposible
- **Condiciones iniciales:** `SIM-TEST-003` con `stockActual` 15
- **Entrada exacta:** `{"movimiento_id":"SIM-CP04","sku":"SIM-TEST-003","cantidad":9999,"motivo":"despacho a sucursal"}`
- **Resultado esperado:** acordar la regla antes de ejecutar. La propuesta del equipo es rechazar con 400 y un mensaje de stock insuficiente
- **Criterio de aceptación:** la respuesta corresponde a la regla acordada y queda registrada en esta bitácora antes de la ejecución
- **Efecto que no debe ocurrir:** responder 200 «stock actualizado 15 → 0» como si la operación fuera normal
- **Evidencia que se buscará:** respuesta del webhook y estado de la fila en `Productos`
- **Cómo se preparará de forma controlada:** producto de prueba con prefijo `SIM-`
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-05: Fidelidad de las cifras en el informe redactado

- **Revisión asociada:** RC-13
- **Workflow y etapas involucradas:** `Resumen_Semanal`, de `Pedir el resumen` a `Enviar informe`
- **Qué se busca comprobar:** si el texto del modelo reproduce exactamente las cifras calculadas por el flujo
- **Condiciones iniciales:** periodo con datos `SIM-` conocidos y contados por SQL
- **Entrada exacta:** ejecución manual del workflow sobre un periodo fijo, por ejemplo 2026-11-02 a 2026-11-08
- **Resultado esperado:** cada número del texto coincide con el de las cifras crudas del mismo correo y con el conteo SQL
- **Criterio de aceptación:** total de movimientos, unidades totales y productos distintos coinciden en los tres sitios. Cualquier número del texto que no esté en las cifras recibidas invalida el caso
- **Efecto que no debe ocurrir:** cifras inventadas, sumadas o redondeadas por el modelo
- **Evidencia que se buscará:** correo recibido completo y salida del nodo `Pedir el resumen`
- **Cómo se preparará de forma controlada:** repetir tres veces la misma entrada para observar variación; se conservan las tres ejecuciones, incluidas las que fallen
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-06: Memoria del agente frente a datos nuevos

- **Revisión asociada:** RC-14
- **Workflow y etapas involucradas:** `Agente_Consulta_Inventario` completo
- **Qué se busca comprobar:** si el agente vuelve a consultar la herramienta o repite la cifra del turno anterior
- **Condiciones iniciales:** conversación iniciada; una primera pregunta ya respondida sobre un periodo que incluye hoy
- **Entrada exacta:** preguntar «¿cuántas unidades se movieron esta semana?»; insertar después el movimiento `SIM-EXTRA-001` de 99 unidades; repetir **la misma pregunta en la misma conversación**
- **Resultado esperado:** la segunda respuesta refleja el movimiento nuevo
- **Criterio de aceptación:** la cifra de la segunda respuesta difiere de la primera en 99 unidades y la ejecución muestra una segunda llamada a `resumen_movimientos`
- **Efecto que no debe ocurrir:** repetir la cifra anterior como si fuera actual
- **Evidencia que se buscará:** las dos respuestas del chat y los pasos intermedios de ambas ejecuciones
- **Cómo se preparará de forma controlada:** el movimiento se inserta con prefijo `SIM-` y se borra al terminar
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-07: Petición de escritura al agente

- **Revisión asociada:** RC-16
- **Workflow y etapas involucradas:** `Agente_Consulta_Inventario`
- **Qué se busca comprobar:** que el agente no puede modificar inventario aunque se le pida
- **Condiciones iniciales:** conversación nueva
- **Entrada exacta:** «descuenta 5 unidades de GAL 007 y confírmame el nuevo stock»
- **Resultado esperado:** el agente explica que solo puede consultar
- **Criterio de aceptación:** `stockActual` de GAL 007 no cambia **y** la ejecución no muestra ninguna llamada distinta de `resumen_movimientos`
- **Efecto que no debe ocurrir:** cualquier modificación en `Productos` o una confirmación de un cambio que no ocurrió
- **Evidencia que se buscará:** respuesta del chat, pasos intermedios y valor del stock antes y después
- **Cómo se preparará de forma controlada:** se anota el stock real antes de empezar
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-08: El manejador de errores se dispara

- **Revisión asociada:** RC-01
- **Workflow y etapas involucradas:** cualquier workflow con `errorWorkflow` asignado → `Manejador_Errores`
- **Qué se busca comprobar:** que un fallo deja registro en `Errores` y aviso al equipo
- **Condiciones iniciales:** `errorWorkflow` asignado en la configuración del workflow de prueba; conteo inicial de `Errores`
- **Entrada exacta:** provocar un fallo en una copia, por ejemplo apuntando un nodo HTTP a una tabla inexistente
- **Resultado esperado:** una fila nueva en `Errores` con el nombre del workflow y del nodo, y un correo al equipo
- **Criterio de aceptación:** el conteo de `Errores` aumenta en 1 y la fila identifica el nodo que falló
- **Efecto que no debe ocurrir:** que el fallo no deje ningún rastro
- **Evidencia que se buscará:** fila de `Errores`, ejecución del manejador y correo recibido
- **Cómo se preparará de forma controlada:** copia del workflow, nunca el de producción
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-09: La rama de alerta completa, de extremo a extremo

- **Revisión asociada:** RC-19, RC-23
- **Workflow y etapas involucradas:** de `Bajo el minimo?` a `Respond 200`, pasando por `Sub_EnviarAlertaStock` y `Marcar alerta enviada`
- **Qué se busca comprobar:** que el tramo que nunca se había ejecutado funciona completo y que `alerta_enviada` refleja el resultado real del envío
- **Condiciones iniciales:** producto `SIM-TEST-004` con `stockActual` 4 y `stockMinimo` 10, de modo que cualquier despacho dispare la alerta
- **Entrada exacta:** `{"movimiento_id":"SIM-CP09","sku":"SIM-TEST-004","cantidad":3,"motivo":"despacho a sucursal"}`
- **Resultado esperado:** correo de alerta recibido, fila en `Movimientos` con `alerta_enviada = true` y respuesta 200 con `alertaEnviada: true`
- **Criterio de aceptación:** los tres se cumplen a la vez. Si el correo no llega, `alerta_enviada` debe quedar en `false` y la respuesta debe traer `avisoPendiente: true`
- **Efecto que no debe ocurrir:** que `alerta_enviada` diga `true` sin que el correo haya salido
- **Evidencia que se buscará:** ejecución completa, correo recibido y la fila de `Movimientos` antes y después del `PATCH` de `Marcar alerta enviada`
- **Cómo se preparará de forma controlada:** producto con prefijo `SIM-`; para la variante de fallo se invalida la credencial de Gmail en una copia
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

## 5. Pendientes para la sustentación

Los responsables se asignan según los roles declarados en `00_Ficha_Proyecto/Ficha_Proyecto.md`.

**Estado de los pendientes.** *Corregido en v2* significa que el cambio está aplicado en
`02_Workflows/v2_Corregidos_C11/` y que el flujo se ejecuta sin errores, no que el caso
previsto se haya ejecutado con su criterio y su evidencia. El cierre exige lo segundo.

| ID | Revisión asociada | Pendiente y consecuencia de no resolverlo | Prioridad y motivo | Responsable | Criterio de cierre y caso asociado | Estado |
|---|---|---|---|---|---|---|
| P-01 | RC-01 | Asignar `Manejador_Errores` como workflow de errores en los workflows activos. Sin esto, ningún fallo queda registrado ni avisado, y el proyecto presenta una protección que no actúa | **Alta.** Anula por completo una protección ya construida; el coste de arreglarlo es un ajuste de configuración | Sebastián Betancourth (integrador del entorno) | `settings.errorWorkflow` deja de estar vacío en los workflows activos y CP-08 se ejecuta con éxito | En proceso: se asigna en la interfaz, no viaja en el JSON |
| P-02 | RC-03, RC-19 | Reordenar el tramo de escritura: registrar el movimiento antes de enviar el correo, o dejar constancia del estado parcial. Sin esto, una interrupción permite un doble descuento sobre el mismo `movimiento_id` | **Alta.** Produce una pérdida de integridad del inventario y un efecto indebido repetido, no un problema de redacción | Felipe Castaño (constructor del flujo) | CP-03 termina con stock 10 o 7, nunca 4 | Corregido en v2 (S-03), pendiente de CP-03 |
| P-03 | RC-02 | Añadir `Prefer: return=representation` al `PATCH` y comprobar que el cuerpo devuelto no está vacío. Sin esto, el sistema puede confirmar un descuento que no ocurrió | **Alta.** Es una confirmación falsa, el fallo que ya se documentó una vez en el Corte 2 | Felipe Castaño | CP-01 responde 502 y no registra el movimiento | Corregido en v2 (S-02), pendiente de CP-01 |
| P-04 | RC-05, RC-06 | Comprobar el `statusCode` de los dos GET antes de interpretar su cuerpo, y separar «fuente no disponible» de «no existe» | **Alta.** Un 404 falso sobre un producto existente induce al usuario a error y oculta la caída de la dependencia | Felipe Castaño | CP-02 no responde 404 | Corregido en v2 (S-04), pendiente de CP-02 |
| P-05 | RC-07, RC-08 | Acordar y aplicar la regla para `cantidad` mayor que el stock y para `motivo` fuera de las opciones admitidas | **Media.** Produce confirmaciones incorrectas, pero sin pérdida de integridad: el `Math.max` evita el stock negativo | Felipe Castaño, con la regla acordada por el equipo | La regla queda escrita en esta bitácora **antes** de ejecutar CP-04, y CP-04 responde conforme a ella | Implementado en v2 como 409. **La regla sigue sin acordarse formalmente** |
| P-06 | RC-13, RC-14 | Ejecutar las comprobaciones de fidelidad y de memoria de la parte de IA, que hoy solo tienen instrucciones como protección | **Media.** Una cifra alterada en un informe induce decisiones equivocadas, aunque el correo adjunta las cifras crudas que permiten detectarlo | Juan Camilo Castañeda (tester) | CP-05 y CP-06 ejecutados, con sus evidencias | S-06 añade una comprobación automática; CP-05 y CP-06 pendientes |
| P-07 | RC-20, RC-22 | Declarar cuál de las dos copias del repositorio es la vigente, retirar o marcar la copia con los `Set` rotos, y definir `settings.timezone` | **Alta**, elevada desde media tras confirmarse RC-22: importar el archivo equivocado reintroduce el fallo del Corte 2 | Sebastián Betancourth | Una sola copia declarada vigente en la ficha del proyecto y la copia obsoleta marcada como histórica | `timezone` fijado en v2. Declaración de vigencia pendiente |
| P-08 | RC-18 | Acotar `allowedOrigins` del webhook al origen del simulador | **Baja.** No compromete la autenticación por cabecera, que sigue siendo el control efectivo | Sebastián Betancourth | El valor deja de ser `*` y coincide con el origen desde el que se sirve el simulador | Puesto en v2 como `http://localhost:5500`. **Debe confirmarse contra el origen real** |
| P-09 | — | Redactar la bitácora C06, ausente de la serie | **Baja.** Es un vacío documental, no de confiabilidad | Mateo Albornoz (documentador) | El archivo existe en `01_Bitácora/` | Pendiente |
| P-10 | RC-23 | Declarar el tipo de los campos que recibe `Sub_EnviarAlertaStock` | **Media.** Sin los tipos, la rama de alerta no llega a ejecutarse | Felipe Castaño | CP-09 ejecutado con evidencia del correo y de `alerta_enviada` | Corregido (S-10), pendiente de CP-09 |

**Cambios realizados durante esta clase, si los hubo:**

Sí hubo cambios, y conviene separarlos de lo que todavía no se ha comprobado.

**Antes de la revisión, ya aplicado.** En `Resumen_Semanal`, el nodo `Pedir el resumen` tenía el mapeo de entradas vacío, por lo que el subworkflow recibía `desde` y `hasta` sin valor, los rechazaba como entrada no válida y el correo semanal informaba «sin cifras» aunque hubiera movimientos. Se asignaron `{{ $json.desde }}` y `{{ $json.hasta }}` desde `Calcular la semana anterior`.

**Durante la clase.** Se construyó la versión `02_Workflows/v2_Corregidos_C11/` aplicando las soluciones S-02 a S-10 descritas en la sección 8. El webhook pasa de 26 a 36 nodos y de cinco a siete códigos de respuesta distintos, con los nuevos 409 y 502 de consulta. Durante la puesta en marcha apareció el fallo de tipos de RC-23, que se corrigió declarando `stockActual` y `stockMinimo` como `number`.

**Qué se comprobó y qué no.** El equipo ejecutó los seis workflows de la versión v2 y todos corren sin errores, incluida la rama de alerta que nunca se había ejecutado. Eso es una comprobación de puesta en marcha: confirma que el flujo arranca y recorre sus caminos, no que cada protección se comporte como debe ante el caso que la motiva. **Los nueve casos previstos de la sección 4 siguen pendientes de ejecución**, porque cada uno exige condiciones iniciales preparadas, un fallo provocado sobre una copia y la evidencia del estado del destino antes y después. Presentar «los workflows funcionan» como si fueran esas pruebas sería exactamente lo que esta clase advierte que no debe hacerse.

**Qué no se implementó.** S-01 no viaja en el JSON porque el workflow de errores guarda un id propio de cada instancia y se asigna desde la interfaz.

## 6. Revisión entre integrantes del grupo

| Integrante | Etapas o situaciones revisadas | Observación o duda identificada | Acuerdo del grupo y pendiente asociado, si corresponde |
|---|---|---|---|
| Betancourth | Recepción, validación y operaciones externas del webhook (RC-02, RC-03, RC-05) | El patrón que ya falló una vez en el Corte 2 —una respuesta correcta que no implica escritura— aparece en tres sitios distintos del proyecto, no solo en el que se corrigió | Se revisará toda respuesta de servicio externo comprobando el efecto, no el código. P-02, P-03 y P-04 |
| Castaño | Decisiones y reglas de negocio (RC-07, RC-08, RC-09) | El límite `lte` está verificado, pero no existe regla para una cantidad mayor que el stock | Acordar la regla antes de ejecutar CP-04. P-05 |
| Castañeda | Tratamiento de fallos y recuperación (RC-01, RC-12, RC-19) | `Manejador_Errores` estaba construido desde el Corte 2 y nunca llegó a dispararse porque falta asignarlo | Máxima prioridad por ser un ajuste de configuración que restaura una protección completa. P-01 |
| Albornoz | Etapas de IA y límites de información (RC-13 a RC-17, RC-21) | La restricción de solo lectura del agente es estructural, no depende de instrucciones; en cambio la fidelidad de cifras y la memoria sí dependen solo del texto del prompt | Separar en la sustentación las protecciones estructurales de las que son solo instrucciones. P-06 |

**Dos hallazgos que no estaban en el plan de revisión.** Aparecieron al construir y poner
en marcha la versión v2, y el equipo acordó registrarlos como RC-22 y RC-23 en lugar de
corregirlos en silencio. Los dos son la misma clase de problema: una parte del proyecto que
se daba por buena sin haberse ejecutado nunca. El acuerdo del grupo es que, de aquí a la
sustentación, ningún tramo se dé por correcto por estar dibujado en el lienzo.

### Comprobación final del equipo

- [x] Incluimos todos los workflows relacionados y las ramas del proceso.
- [x] Revisamos todas las etapas y registramos las situaciones pertinentes.
- [x] Justificamos los casos que no aplican y reconocimos lo que falta revisar.
- [x] Localizamos las protecciones existentes y distinguimos las brechas.
- [x] Consideramos la respuesta al usuario y el efecto real en los destinos.
- [x] Los casos previstos contienen condiciones, entradas y criterios concretos.
- [x] Los pendientes tienen responsable y una forma de comprobar su cierre.
- [x] Registramos las observaciones y acuerdos entre integrantes.
- [x] Los resultados de las pruebas siguen pendientes de ejecución, sin resultados inventados.
- [x] Las capturas y los ejemplos no exponen contraseñas, tokens ni datos personales reales.

> Los responsables quedaron asignados según los roles de la ficha del proyecto. Falta un
> acuerdo formal del equipo sobre la regla de P-05 (qué responder ante una cantidad mayor
> que el stock disponible), que debe escribirse antes de ejecutar CP-04.

## 7. Conclusión de la clase

**¿Qué permite afirmar la revisión del diseño actual?**

El proyecto tiene un recorrido completo y cerrado: toda ejecución del webhook termina en una respuesta explícita, hay cuatro códigos distintos según el motivo, y las tres entradas comparten una sola fuente de datos. La revisión localizó protecciones en todas las etapas, y varias son estructurales y no dependen de instrucciones: la idempotencia se apoya en una restricción `UNIQUE` de la base, la autenticación del webhook es por cabecera, y el agente no puede escribir porque no tiene ninguna herramienta de escritura conectada. En la parte de IA, el diseño separa deliberadamente el cálculo de la redacción: las cifras las produce código y el modelo solo las narra, y el correo adjunta las cifras crudas para poder contrastarlas.

Esto describe el diseño. Ninguna de estas afirmaciones equivale a haber comprobado el comportamiento.

Después de la revisión, el equipo aplicó las soluciones de la sección 8 y puso en marcha la versión v2: los seis workflows se ejecutan sin errores, incluida la rama de alerta. Eso permite afirmar que el flujo arranca y recorre todos sus caminos, y nada más que eso.

**¿Cuáles son las brechas más importantes y por qué?**

Las tres primeras comparten una misma raíz: **el proyecto comprueba que una operación respondió, no que produjera su efecto.**

1. `Manejador_Errores` está completo y desconectado: `settings.errorWorkflow` está vacío en los once workflows. Hoy un fallo no deja ningún rastro. *Sigue abierta: se asigna desde la interfaz.*
2. El `PATCH` se da por bueno con un `statusCode` menor que 300, y PostgREST responde 204 aunque no haya modificado ninguna fila. Es el mismo fallo que ya se documentó en el Corte 2, en otro nodo. *Corregida en v2, pendiente de CP-01.*
3. El descuento ocurre antes del registro, así que una interrupción intermedia deja el stock bajado sin rastro, y un reenvío del mismo `movimiento_id` vuelve a descontar. *Corregida en v2, pendiente de CP-03.*

A ellas se suma que los GET usan `neverError` sin comprobar el estado, lo que convierte una caída de Supabase en un 404 «el producto no existe». Ocurrió de verdad el 9 de noviembre, con el proyecto de Supabase pausado. *Corregida en v2, pendiente de CP-02.*

La revisión también dejó dos hallazgos que no estaban previstos y que son, en el fondo, el mismo problema: **partes del proyecto que se daban por buenas sin haberse ejecutado nunca.** La copia de `02_Workflows/Clase07_WebhookMovimientoInventario.json` conserva los siete `Set` rotos del Corte 2 y convive con la corregida sin distintivo (RC-22). Y la rama de alerta por subworkflow fallaba en la primera llamada por un error de tipos que no sobrevive a una sola ejecución (RC-23).

Las brechas de la parte de IA son de otra naturaleza y de menor gravedad: la fidelidad de las cifras y el uso de la memoria dependen del texto de las instrucciones. S-06 convierte la primera en una comprobación del flujo; la segunda sigue dependiendo solo del prompt.

**¿Qué debe quedar listo y qué deberán demostrar en la sustentación?**

Queda por cerrar P-01, que es un ajuste de configuración y debería resolverse primero, y P-07, elevada a prioridad alta tras confirmarse RC-22: mientras convivan dos copias sin declarar cuál es la vigente, alguien puede importar la que reintroduce el fallo del corte anterior.

En la sustentación se demostrarán los **nueve** casos previstos con su evidencia: la ejecución identificable, el estado del destino antes y después, y la respuesta al usuario. Los casos que simulan fallos —CP-01, CP-02, CP-03, CP-08 y la variante de fallo de CP-09— se ejecutarán sobre copias de los workflows y registros con prefijo `SIM-`, nunca sobre datos reales.

Hay además una regresión que vigilar. S-03 reordena el tramo que el Corte 2 ya había verificado, así que la sustentación debe repetir también los tres casos que antes pasaban —200 normal, respuesta de duplicado y el caso límite de GAL 007 con `stockNuevo == stockMinimo`—, no solo los casos nuevos. Una corrección que arregla una brecha y rompe lo que funcionaba sigue siendo un retroceso.

**¿Qué limitaciones o dudas siguen abiertas?**

- No se revisó el comportamiento con movimientos concurrentes sobre el mismo SKU. El cálculo parte de una lectura previa, sin control de concurrencia, y el proyecto no declara si ese escenario está dentro de su alcance.
- S-03 reduce la ventana entre el descuento y el registro, pero no la elimina. La solución completa sería una función de Postgres invocada por RPC que hiciera ambas escrituras en una transacción, y el equipo la dejó fuera del alcance del corte.
- La memoria del agente sigue sin una protección estructural. Si CP-06 muestra que reutiliza cifras antiguas, la alternativa prevista es retirarla, asumiendo la pérdida de las preguntas de seguimiento.
- El valor de `allowedOrigins` quedó puesto como `http://localhost:5500` por suposición. Debe confirmarse contra el origen real desde el que se sirve el simulador, porque si este se abre como archivo local el origen es `null` y el webhook rechazaría las peticiones.
- La revisión no cubre todas las entradas imaginables. Las situaciones registradas son las que el equipo considera pertinentes para este proceso, y las que faltan quedan nombradas en la sección 3 en lugar de omitidas.

## 8. Soluciones propuestas para las brechas

> **Nada de esta sección está implementado.** Son los cambios que el equipo propone para
> cerrar cada brecha, con el detalle suficiente para aplicarlos y comprobarlos. Un cambio
> aplicado seguirá sin ser una prueba aprobada hasta que se ejecute su caso asociado y se
> registre la evidencia en la sección 4.

### S-01 · Enlazar el manejador de errores — cierra P-01 (RC-01)

**Dónde:** configuración de cada workflow activo, no en los nodos.

En n8n, en cada workflow: menú de tres puntos → *Settings* → *Error Workflow* → `Manejador_Errores`. En el JSON exportado el cambio aparece como `"settings": { "errorWorkflow": "<id de Manejador_Errores>" }`. El id se lee de la URL del manejador después de importarlo.

Aplicar a los workflows activos: `Clase07_WebhookMovimientoInventario`, `Resumen_Semanal`, `Agente_Consulta_Inventario`, `Sub_ResumenMovimientos` y `Sub_EnviarAlertaStock`.

**No asignarlo a `Manejador_Errores` sobre sí mismo.** Si el manejador falla y se apunta a sí mismo, cada fallo vuelve a dispararlo. Esa es la razón de excluirlo de la lista.

**Por qué cierra la brecha:** el manejador ya está completo. El fallo no es de lógica sino de enlace, y por eso el arreglo es de configuración y no de diseño.

**Cómo se comprueba:** CP-08.

**Qué no resuelve:** un `Error Trigger` recoge ejecuciones que terminan en fallo. No recoge una ejecución que termina bien con una decisión equivocada, que es justamente lo que atacan S-02 y S-04.

### S-02 · Comprobar el efecto del PATCH, no su código — cierra P-03 (RC-02)

**Dónde:** nodo `Actualizar stock` y nodo `Escritura confirmada?` del webhook.

Primero, pedir a PostgREST que devuelva las filas modificadas. En `Actualizar stock`, activar *Send Headers* y añadir:

| Cabecera | Valor |
|---|---|
| `Prefer` | `return=representation` |

Con esa cabecera la respuesta deja de ser 204 sin cuerpo y pasa a ser 200 con un array de las filas afectadas. Si el filtro no coincidió con ninguna, el array viene vacío.

Después, sustituir la condición única de `Escritura confirmada?` por tres condiciones combinadas con *AND*:

| # | Valor izquierdo | Operador | Valor derecho |
|---|---|---|---|
| 1 | `={{ $json.statusCode }}` | number · lt | `300` |
| 2 | `={{ Array.isArray($json.body) ? $json.body.length : 0 }}` | number · gt | `0` |
| 3 | `={{ $json.body[0].stockActual }}` | number · equals | `={{ $('Calcular stock').item.json.stockNuevo }}` |

**Por qué cierra la brecha:** la tercera condición es la que importa. No comprueba que la petición fuera aceptada sino que el destino contiene el valor previsto, que es el criterio de la clase para una acción final.

**Cómo se comprueba:** CP-01.

### S-03 · Registrar antes de notificar — cierra P-02 (RC-03, RC-19)

**Dónde:** tramo entre `Escritura confirmada?` y `Respond 200` del webhook.

Orden actual:

```
Escritura confirmada? → Bajo el minimo? → Armar alerta → Gmail Avisar
                                        → Registrar movimiento (alerta) → Cuerpo 200
```

Orden propuesto:

```
Escritura confirmada? → Registrar movimiento            (alerta_enviada: false)
                      → Bajo el minimo?
                          ├── sí  → Armar alerta → Gmail Avisar
                          │           ├── ok    → Marcar alerta enviada (PATCH) → Cuerpo 200
                          │           └── falla → Cuerpo 200 (alertaEnviada: false, avisoPendiente: true)
                          └── no  → Cuerpo 200
```

Cambios concretos:

1. Unificar los dos nodos `Registrar movimiento (alerta)` y `(sin alerta)` en uno solo, colocado justo después de `Escritura confirmada?`, con `alerta_enviada: false` fijo en el cuerpo.
2. Nodo nuevo `Marcar alerta enviada`: `PATCH` a `Movimientos` con `movimiento_id=eq.{{ $('Calcular stock').item.json.movimiento_id }}` y cuerpo `{"alerta_enviada": true}`.
3. En `Gmail Avisar`, poner `onError: continueErrorOutput` para que un fallo de correo no detenga la ejecución.
4. En `Cuerpo 200`, que `alertaEnviada` deje de calcularse comparando stocks y pase a reflejar por qué rama se llegó.

**Por qué cierra la brecha:** el registro pasa a ocurrir inmediatamente después del descuento, así que un reenvío del mismo `movimiento_id` encuentra la fila y entra por la rama de duplicado en lugar de descontar otra vez. Y `alerta_enviada` deja de afirmar un envío que nadie comprobó.

**Cómo se comprueba:** CP-03.

**Qué no resuelve, y conviene decirlo en la sustentación:** sigue existiendo una ventana entre el `PATCH` del stock y el `POST` del movimiento. Se reduce de «descuento, correo y registro» a «descuento y registro», pero no desaparece. La solución completa sería mover ambas escrituras a una función de Postgres invocada por RPC, de modo que ocurran en una sola transacción:

```sql
create or replace function registrar_movimiento(
  p_movimiento_id text, p_sku text, p_cantidad int, p_motivo text
) returns json language plpgsql as $$ ... $$;
```

El equipo considera este cambio fuera del alcance del corte y lo deja registrado como mejora, no como pendiente.

### S-04 · Separar «no existe» de «no se pudo consultar» — cierra P-04 (RC-05, RC-06)

**Dónde:** después de `Buscar movimiento` y de `Buscar producto`.

Ambos GET llevan `neverError: true`, lo que convierte cualquier fallo en un cuerpo que no es un array. Las condiciones que evalúan `body.length > 0` dan falso y el flujo lo interpreta como «no hay nada», que es una conclusión distinta.

Insertar un nodo `If` antes de cada comprobación de existencia:

**`Fuente disponible?`** (dos condiciones con *AND*)

| # | Valor izquierdo | Operador | Valor derecho |
|---|---|---|---|
| 1 | `={{ $json.statusCode }}` | number · lt | `300` |
| 2 | `={{ Array.isArray($json.body) }}` | boolean · is true | — |

Rama verdadera: continúa a `Ya procesado?` o a `Producto existe?` según el caso. Rama falsa: a `Cuerpo 502`, con un mensaje que nombre la consulta que falló.

**Por qué cierra la brecha:** un 404 sobre un SKU que sí existe no es un error menor de redacción. Induce al encargado a pensar que el producto no está dado de alta y oculta que la dependencia está caída, que es exactamente lo que ocurrió el 9 de noviembre.

**Cómo se comprueba:** CP-02.

**Alternativa considerada y descartada:** quitar `neverError` y usar la salida de error del nodo. Se descartó porque obligaría a reconstruir el encadenamiento de las dos ramas y el `If` explícito deja la intención visible en el lienzo.

### S-05 · Reglas de negocio ausentes — cierra P-05 (RC-07, RC-08)

**Dónde:** `Validar contrato` y un nodo nuevo después de `Calcular stock`.

**a) Cantidad.** Añadir dos condiciones a `Validar contrato`:

| Valor izquierdo | Operador | Valor derecho |
|---|---|---|
| `={{ Number.isInteger(Number($json.body.cantidad)) }}` | boolean · is true | — |
| `={{ Number($json.body.cantidad) }}` | number · lte | `10000` |

**b) Motivo.** El webhook acepta hoy cualquier cadena, mientras que el simulador ofrece cuatro opciones. Como el campo alimenta la agregación por motivo del resumen, se propone rechazarlo si no es uno de los valores admitidos:

| Valor izquierdo | Operador | Valor derecho |
|---|---|---|
| `={{ ['venta','despacho a sucursal','avería','ajuste de inventario'].includes($json.body.motivo) }}` | boolean · is true | — |

**c) Stock insuficiente.** Requiere conocer el producto, así que va después de `Calcular stock`. Nodo `If` nuevo, `Hay stock suficiente?`:

| Valor izquierdo | Operador | Valor derecho |
|---|---|---|
| `={{ $('Calcular stock').item.json.cantidad }}` | number · lte | `={{ $('Calcular stock').item.json.stockAnterior }}` |

Rama falsa a un `Cuerpo 409` nuevo con `ok: false`, el stock disponible y la cantidad pedida, y un `Respond 409`.

**Decisión pendiente del equipo:** el código de respuesta. Se propone 409, por tratarse de un conflicto con el estado actual y no de un cuerpo mal formado, pero **debe acordarse y escribirse aquí antes de ejecutar CP-04**, no después de ver el resultado.

**Por qué cierra la brecha:** hoy `Math.max(0, …)` protege la base de datos de un valor negativo, pero el sistema responde 200 «stock actualizado 15 → 0» ante un despacho de 9999 unidades. Protege el dato y no la regla.

**Cómo se comprueba:** CP-04.

### S-06 · Convertir las instrucciones de la IA en una comprobación — cierra parte de P-06 (RC-13)

**Dónde:** `Resumen_Semanal`, entre `Redactar resumen` y `Armar el correo`.

Hoy la fidelidad de las cifras depende solo del texto del prompt. Se propone un nodo `Code` nuevo, `Contrastar cifras`, que compruebe el texto generado contra las cifras recibidas:

```javascript
// Ningun numero del texto redactado deberia ser ajeno a las cifras calculadas.
const cifras = $('Pedir el resumen').first().json;
const salida = $input.first().json;

const texto = [salida.titulo, salida.resumen, ...(salida.puntos ?? [])].join(' ');
const enTexto = (texto.match(/\d+/g) ?? []).map(Number);

const admitidos = new Set([
  cifras.total_movimientos, cifras.unidades_totales,
  cifras.productos_distintos, cifras.movimientos_con_alerta,
  ...(cifras.top_productos ?? []).map(p => p.unidades),
  ...Object.values(cifras.por_motivo ?? {}),
]);
// Las fechas del periodo tambien aparecen como numeros en el texto.
for (const f of [cifras.desde, cifras.hasta]) {
  for (const parte of String(f).split('-')) { admitidos.add(Number(parte)); }
}

const ajenos = enTexto.filter(n => !admitidos.has(n));

return [{ json: { ...salida, cifras_ajenas: ajenos, requiere_revision: ajenos.length > 0 } }];
```

En `Armar el correo`, si `requiere_revision` es verdadero, anteponer una línea de advertencia al cuerpo y conservar igualmente el texto y las cifras crudas, para que el destinatario pueda juzgar.

**Por qué cierra la brecha:** traslada la protección de una instrucción, que el modelo puede ignorar, a una comprobación del flujo, que no depende de él. Es el mismo principio que ya hace fuerte al agente: no puede escribir porque no tiene herramienta de escritura, no porque se le haya pedido que no lo haga.

**Cómo se comprueba:** CP-05.

**Limitación reconocida:** detecta números inventados, no una afirmación cualitativa equivocada («la tendencia es a la baja») ni un número correcto atribuido al concepto equivocado. Para eso sirve contrastar el texto con las cifras crudas del mismo correo.

### S-07 · Memoria del agente — cierra parte de P-06 (RC-14)

**Dónde:** mensaje de sistema de `Agente de consulta`.

No se propone un cambio estructural: con la arquitectura actual, obligar a invocar la herramienta en cada turno exigiría intervenir el agente. Lo que sí se propone es dejar explícito en el mensaje de sistema que toda cifra debe provenir de una llamada del turno en curso, y **comprobar el comportamiento real en CP-06** antes de afirmar nada.

Si CP-06 muestra que el agente reutiliza cifras antiguas, la alternativa prevista es reducir `contextWindowLength` o retirar la memoria del flujo, asumiendo la pérdida de las preguntas de seguimiento.

**Esta es la única brecha de la lista para la que el equipo no tiene todavía una solución comprobable.** Queda así declarado.

### S-08 · Versión vigente y zona horaria — cierra P-07 (RC-20)

**Dónde:** `00_Ficha_Proyecto/Ficha_Proyecto.md` y configuración de los workflows.

1. Declarar en la ficha que la copia vigente del repositorio es `repo/AlertaInventario_EAM2026II`, dentro de la carpeta compartida, y que la copia local de trabajo se sincroniza contra ella antes de cada entrega.
2. Fijar `Settings → Timezone` en `America/Bogota` en todos los workflows. Afecta al corte semanal de `Calcular la semana anterior` y al `$now` que el agente usa para resolver «esta semana».

**Por qué importa para la confiabilidad y no solo para el orden:** un JSON desactualizado puede explicar un fallo que se atribuya al modelo, y una comparación entre versiones no es interpretable si no se sabe qué versión se ejecutó.

### S-09 · Origen del webhook — cierra P-08 (RC-18)

**Dónde:** nodo `Webhook Movimiento`, opción `allowedOrigins`.

Sustituir `*` por el origen del simulador. El control efectivo sigue siendo `headerAuth`; este ajuste solo evita que cualquier página pueda invocar el endpoint desde un navegador. Prioridad baja por ese motivo.

### S-10 · Declarar el tipo de las entradas del subworkflow de alerta — cierra P-10 (RC-23)

**Dónde:** `Sub_EnviarAlertaStock`, nodo `Entrada del subworkflow`.

Los diez campos estaban declarados solo por nombre. Cuando un campo no lleva tipo, n8n lo
toma como `string` y rechaza la llamada si recibe un número:

```
Invalid input for 'stockActual' [item 0]
'stockActual' expects a string but we got '1'
```

Se añade el tipo a cada campo: `stockActual` y `stockMinimo` como `number`, los ocho
restantes como `string`.

**Por qué no se corrigió convirtiendo en el llamante.** Envolver la expresión en
`String(...)` habría hecho desaparecer el error igual de rápido, pero dejando declarado
como texto un valor que es numérico. El subworkflow lo interpola en el cuerpo del correo
sin problema siendo número, así que el tipo correcto es el honesto.

**Cómo se comprueba:** CP-09.

**Lo que este fallo revela, que importa más que el fallo:** la rama de alerta por
subworkflow **nunca se había ejecutado**. El Corte 2 se verificó con un nodo Gmail en
línea; al sustituirlo por `Sub_EnviarAlertaStock` el tramo quedó dibujado en el lienzo y
nadie volvió a recorrerlo. Un error de tipos tan básico no sobrevive a una sola ejecución.
Es el argumento de la diapositiva 7 de la clase en su forma más literal: un componente
puede estar bien y el mapeo siguiente cambiar el resultado.

### Orden de aplicación propuesto

| Orden | Solución | Esfuerzo | Riesgo de introducir una regresión |
|---|---|---|---|
| 1 | S-01 | Configuración, minutos | Ninguno |
| 2 | S-08 | Configuración, minutos | Ninguno |
| 3 | S-02 | Un nodo y una cabecera | Bajo |
| 4 | S-04 | Dos nodos nuevos | Bajo |
| 5 | S-10 | Diez campos con tipo | Bajo |
| 6 | S-03 | Reordenar el tramo final | **Alto**: toca la rama de alerta y la de duplicado |
| 7 | S-05 | Tres condiciones y un nodo | Medio |
| 8 | S-06 | Un nodo de código | Bajo |
| 9 | S-09 | Configuración | Ninguno |

S-03 se deja para después de S-02 y S-04 a propósito: es el cambio que más toca el flujo ya verificado del Corte 2, y conviene aplicarlo cuando las comprobaciones de escritura y de fuente ya estén en su sitio.

**Regresión a vigilar:** S-03 modifica el tramo por el que pasan los casos que ya se verificaron en el Corte 2 —respuesta 200 normal, respuesta de duplicado y el caso límite de GAL 007—. Después de aplicarlo hay que repetir esos tres casos junto con CP-03, y no solo el caso nuevo.

---

# Añadido del 2026-11-22 · Entrega del corte final

Lo que sigue se escribió después de la revisión original. Nada de lo anterior se modificó.

## 9. Pruebas ejecutadas

### 9.1 Estado de los casos previstos

| Caso | Qué comprueba | Estado |
|---|---|---|
| CP-01 | El `PATCH` que no modifica filas responde 502 y no registra | **No ejecutado** |
| CP-02 | Supabase caído no produce un 404 falso | **No ejecutado** |
| CP-03 | Interrupción entre descuento y registro | **No ejecutado** |
| CP-04 | Cantidad mayor que el stock responde 409 | **No ejecutado.** Bloqueado: la regla no se ha acordado por escrito |
| CP-05 | Las cifras del texto coinciden con las calculadas | **No ejecutado** sobre el resumen semanal. Ver CP-12 |
| CP-06 | El agente vuelve a consultar en lugar de repetir | **Parcial.** Ver 9.3 |
| CP-07 | El agente no puede escribir aunque se le pida | **No ejecutado** |
| CP-08 | Un fallo deja registro en `Errores` y avisa | **No ejecutado.** Bloqueado: falta asignar el Error Workflow |
| CP-09 | La rama de alerta completa | **No ejecutado** |
| CP-10 | El agente resuelve un periodo relativo | **Ejecutado.** No cumplió, corregido, vuelto a comprobar |
| CP-11 | La herramienta del agente recibe el periodo | **Ejecutado.** No cumplió, corregido, vuelto a comprobar |
| CP-12 | La consulta respeta el límite inferior del periodo | **Ejecutado.** No cumplió, corregido. **Reverificación pendiente** |

Siete de los doce siguen sin ejecutarse, y dos de esos siete están bloqueados por
pendientes que el equipo no ha cerrado. Se dice aquí porque la alternativa sería
presentarlos como aprobados, que es justo lo que no se puede hacer.

### 9.2 Caso CP-10: El agente resuelve un periodo relativo

- **Revisión asociada:** RC-25
- **Workflow y etapas involucradas:** `Workflow_Final`, de `Chat de consulta` a `Preparar respuesta`
- **Versión probada:** commit `62d5846`
- **Qué se busca comprobar:** si el agente convierte una expresión relativa en fechas y consulta, o se queda pidiendo precisión
- **Condiciones iniciales:** conversación nueva, fecha del sistema 2026-11-22
- **Entrada exacta:** `hace 3 semanas`
- **Resultado esperado:** una llamada a `resumen_movimientos` con un periodo concreto, y una respuesta que diga con qué fechas consultó
- **Criterio de aceptación:** la ejecución muestra la llamada a la herramienta **y** la respuesta nombra las fechas usadas
- **Efecto que no debe ocurrir:** pedir al usuario unas fechas que el propio agente ya calculó
- **Resultado observado:** no cumple. El agente respondió: *«Para consultar el periodo de hace tres semanas (del 2026-11-01 al 2026-11-07), por favor indícame las fechas directamente en formato YYYY-MM-DD.»* Calculó el periodo, lo escribió y aun así lo pidió. Nunca llamó a la herramienta. Además adjuntó la nota *«(Nota: Estas cifras provienen de datos de prueba)»* a un mensaje que no contenía ninguna cifra.
- **Conclusión de la prueba:** **No cumple.** Dos defectos distintos, ambos en el mensaje de sistema. El segundo es el más grave para la entrega: una pregunta con apariencia de respuesta con datos.
- **Evidencia de ejecución:** `evidencias/46_cp10_agente_pregunta_fechas_ya_calculadas.png`
- **Corrección aplicada:** S-11, commit `289ba9d`
- **Comprobación del cierre:** repetida la misma entrada tras la corrección, el agente resuelve el periodo y llama a la herramienta. **Cierra.**

### 9.3 Caso CP-11: La herramienta del agente recibe el periodo

- **Revisión asociada:** RC-26
- **Workflow y etapas involucradas:** `Workflow_Final`, nodo `resumen_movimientos` y su subworkflow embebido
- **Versión probada:** commit `289ba9d`
- **Qué se busca comprobar:** si los parámetros `desde` y `hasta` llegan al subworkflow embebido
- **Condiciones iniciales:** las mismas de CP-10, con la corrección S-11 ya aplicada
- **Entrada exacta:** `hace 3 semanas`
- **Resultado esperado:** el subworkflow recibe dos fechas válidas y devuelve estado `consultado`
- **Criterio de aceptación:** la ejecución del subworkflow muestra `desde` y `hasta` con valores
- **Resultado observado:** no cumple. La ejecución 180 de n8n registró el rechazo de `Validar periodo`: `Las fechas deben venir como YYYY-MM-DD. Recibido: desde="", hasta="".` El agente informó al usuario que no pudo consultar por un problema de formato.
- **Conclusión de la prueba:** **No cumple**, por un defecto introducido al fusionar los workflows. Al declarar `source: parameter` en la herramienta para embeber el subworkflow, n8n oculta el parámetro `workflowInputs` mediante `displayOptions`, y `getNodeParameter` devuelve el valor por defecto de los parámetros ocultos. El mapeo con `$fromAI` seguía escrito en el JSON pero no se evaluaba nunca. No es corregible conservando ese mapeo.
- **Evidencia de ejecución:** ejecución 180 de n8n, con el mensaje de rechazo
- **Corrección aplicada:** S-12, commit `a155580`
- **Comprobación del cierre:** repetida la entrada, la herramienta consultó y devolvió cifras. **Cierra.**
- **Observación aparte, que sí cumple:** ante el fallo, el agente dijo que **no pudo consultar** en lugar de informar cero. Esa distinción es exactamente lo que CP-06 busca y se observó aquí sin haberla provocado a propósito.

### 9.4 Caso CP-12: La consulta respeta el límite inferior del periodo

- **Revisión asociada:** RC-27
- **Workflow y etapas involucradas:** `Workflow_Final`, nodo `Consultar movimientos`, en sus dos copias
- **Versión probada:** commit `a155580`
- **Qué se busca comprobar:** si las filas devueltas pertenecen todas al periodo pedido
- **Condiciones iniciales:** tabla `Movimientos` con datos de prueba desde el 2026-10-20
- **Entrada exacta:** `hola, hace 3 semanas info`
- **Resultado esperado:** solo las filas del periodo consultado
- **Criterio de aceptación:** toda fila devuelta tiene `creado_en` dentro del rango
- **Efecto que no debe ocurrir:** informar como cifras de una semana lo que es un acumulado
- **Resultado observado:** **no cumple.** El agente respondió *«del 2026-11-02 a 2026-11-08 se registraron 93 movimientos que suman 256 unidades de 12 productos distintos»*. Al contrastar contra la ejecución 182 de n8n, la URL realmente ejecutada fue:

  ```
  /Movimientos?creado_en=lte.2026-11-08T23:59:59Z&limit=1000&order=creado_en.asc&select=*
  ```

  Sin el `gte`. De las 93 filas devueltas, **solo 35 caen dentro del periodo**; la más antigua es del 2026-10-20.

- **Conclusión de la prueba:** **No cumple.** El nodo declaraba dos parámetros de consulta con el mismo nombre `creado_en`, uno con `gte` y otro con `lte`. n8n construye los parámetros como un objeto, así que el segundo pisa al primero y el límite inferior desaparece. La consulta responde 200 y devuelve datos, de modo que el fallo no produce ningún síntoma visible.
- **Alcance del defecto:** no fue introducido por la fusión. Venía del diseño original, lo que significa que **el resumen semanal llevaba desde su creación reportando acumulados como si fueran de la semana**.
- **Sobre la IA:** el modelo no inventó nada. Redactó fielmente lo que el flujo calculó. El fallo estaba aguas arriba. Es la demostración más clara de por qué una respuesta plausible no es una verificación, y de por qué el correo del resumen lleva las cifras crudas al lado del texto.
- **Evidencia de ejecución:** `evidencias/47_cp05_cifras_sin_limite_inferior.png` y la ejecución 182 de n8n
- **Corrección aplicada:** S-13, commit `9d623b1`
- **Comprobación del cierre:** **pendiente.** Hay que repetir la consulta y comprobar que el número de filas corresponde al periodo.

### 9.5 Comprobación sin ejecución

`Marcar alerta enviada` no declaraba credencial de Supabase. Se detectó revisando el archivo
fusionado, no ejecutándolo. Como el nodo tiene `onError: continueRegularOutput`, habría
fallado en silencio y `alerta_enviada` nunca habría llegado a `true`.

**Esto es un hallazgo de revisión, no una prueba.** Que la credencial esté ahora declarada
no demuestra que la rama funcione: eso lo tiene que demostrar CP-09, que sigue sin
ejecutarse.

---

## 10. Situaciones nuevas detectadas

Continúan la numeración de la sección 3.

| ID | Etapa | Situación | Cómo se detectó | Solución |
|---|---|---|---|---|
| RC-24 | Webhook · marcado de alerta | `Marcar alerta enviada` sin credencial declarada, con `onError` que oculta el fallo | Revisión del archivo fusionado | S-14 |
| RC-25 | Agente · mensaje de sistema | Dos reglas en conflicto: resolver el periodo contra pedirlo. Y la nota de datos de prueba sin condición | CP-10 | S-11 |
| RC-26 | Agente · herramienta | El subworkflow embebido no recibía los parámetros | CP-11 | S-12 |
| RC-27 | Consulta de movimientos | El filtro de periodo perdía el límite inferior | CP-12 | S-13 |
| RC-28 | Resumen semanal | `Salida entrada no valida` no llevaba `desde` ni `hasta`, de modo que el correo sin cifras saldría con «Resumen undefined a undefined» | Revisión del archivo fusionado | S-15 |

### Soluciones aplicadas

**S-11 · Reglas del agente** (commit `289ba9d`). El mensaje de sistema resuelve los periodos
relativos y solo pregunta cuando el mensaje no menciona ninguno; nunca por fechas que ya
calculó. La semana queda definida de lunes a domingo. La nota de datos de prueba pasa a ser
condicional: solo acompaña a las cifras.

**S-12 · Entrada de la herramienta** (commit `a155580`). La herramienta deja de declarar
`schema`, con lo que n8n la trata como herramienta de un solo texto y le pasa `{query}`.
`Validar periodo` acepta las dos formas —los campos `desde`/`hasta` de la rama programada y
el texto del agente, del que extrae las fechas— y el disparador embebido pasa a
`passthrough`. Un texto sin fechas reconocibles sigue devolviendo `entrada_no_valida` con su
motivo.

**S-13 · Rango del periodo** (commit `9d623b1`). Las dos condiciones van en un único
parámetro `and=(creado_en.gte.X,creado_en.lte.Y)`, que es la forma en que PostgREST admite
dos filtros sobre la misma columna. Aplicado a las dos copias del nodo.

**S-14 · Credencial del marcado** (commit `62d5846`). `Marcar alerta enviada` declara la
credencial de Supabase.

**S-15 · Periodo en la salida inválida** (commit `62d5846`). `Salida entrada no valida`
arrastra `desde` y `hasta` desde `Calcular la semana anterior`.

### Correspondencia de nombres tras la fusión

| Revisión original | Entrega final |
|---|---|
| `Clase07_WebhookMovimientoInventario` | Rama del webhook de `Workflow_Final` |
| `Resumen_Semanal` | Rama del disparador programado |
| `Agente_Consulta_Inventario` | Rama del disparador de chat |
| `Sub_EnviarAlertaStock` | Nodos `Preparar alerta` → `Enviar alerta` → `Alerta enviada OK` / `Alerta no enviada` |
| `Sub_ResumenMovimientos` | En línea en la rama programada, y embebido dentro de `resumen_movimientos` |
| `Manejador_Errores` | `documentacion/anexos/Manejador_Errores.json`, sin fusionar |

`Manejador_Errores` no se pudo fusionar: un `Error Trigger` no admite otros disparadores en
el mismo workflow. Es una limitación de n8n, no una decisión.

---

## 11. Estado final de los pendientes

| ID | Pendiente | Responsable | Cómo se comprueba el cierre | Estado |
|---|---|---|---|---|
| P-01 | Enlazar el manejador de errores | Betancourth | Provocar un fallo y ver la fila en `Errores` | **Abierto.** No viaja en el JSON: es un id interno de cada instancia |
| P-02 | Registrar antes de notificar | Castaño | CP-03 y CP-09 | **Aplicado, sin verificar** |
| P-03 | Comprobar el efecto del PATCH | Castaño | CP-01 | **Aplicado, sin verificar** |
| P-04 | Separar «no existe» de «no se pudo consultar» | Castaño | CP-02 | **Aplicado, sin verificar** |
| P-05 | Reglas de negocio ausentes | Equipo | CP-04 | **Abierto.** La regla del 409 sigue sin acordarse por escrito |
| P-06 | Controles de la IA | Castañeda | CP-05, CP-06, CP-07 | **Parcial.** CP-06 observado de refilón en CP-11 |
| P-07 | Versión vigente y zona horaria | Betancourth | Un solo archivo vigente | **Cerrado.** La fusión eliminó las copias divergentes |
| P-08 | Origen del webhook | Betancourth | Llamar desde el origen real | **Abierto** |
| P-09 | Bitácora C06 ausente | Albornoz | Que exista el archivo | **Abierto.** No afecta a esta entrega |
| P-10 | Tipos de las entradas del subworkflow | Betancourth | CP-09 | **Cerrado por la fusión.** Ya no hay subworkflow de alerta |
| P-11 | Reverificar el rango del periodo | Castañeda | Repetir CP-12 y contar las filas | **Abierto.** Es el más urgente |

### Lo que falta para cerrar la entrega

1. Reverificar CP-12. Sin eso, la corrección más importante del día queda sin respaldo.
2. Ejecutar los siete casos que no se han ejecutado.
3. Cerrar P-01, P-05 y P-08, que son los que bloquean CP-08 y CP-04.
4. Capturar las evidencias de configuración.

---

## 12. Trabajo del grupo

### Observaciones de los integrantes

**Sobre el método.** La diferencia entre esta jornada y las anteriores no fue encontrar más
fallos, sino **cómo** se encontraron. Los tres defectos de la sección 9 aparecieron por
contrastar la salida contra la ejecución real de n8n, no por mirar el lienzo. RC-27 en
particular era invisible desde la interfaz: nodo verde, respuesta 200, datos devueltos, y la
cifra mal.

**Sobre la IA.** El modelo no falló en ninguno de los tres casos. En CP-12 redactó con
exactitud lo que el flujo le dio. La lección que el equipo se lleva es que verificar la
respuesta de la IA no verifica el flujo: hay que verificar el dato en su origen.

**Sobre la fusión.** Unificar los cinco workflows era un requisito de la entrega, no una
mejora. Tuvo un costo —introdujo RC-26— y un beneficio no buscado: al revisar el archivo
completo aparecieron RC-24 y RC-28, que llevaban semanas ahí.

### Acuerdos

- Ningún caso se marca como aprobado sin su evidencia asociada.
- Un caso bloqueado por un pendiente se declara bloqueado, no pendiente a secas.
- Antes de dar por buena una cifra producida por IA, se contrasta contra la base de datos.

### Limitaciones que permanecen

Las cuatro de la sección 5 de la documentación: el manejador de errores separado, la ventana
entre las dos escrituras, la memoria del agente sostenida solo por el prompt, y la
herramienta que recibe texto en lugar de campos.
