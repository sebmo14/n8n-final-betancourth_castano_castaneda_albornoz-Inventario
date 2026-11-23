# Alerta automática de inventario

Automatización en **n8n** que recibe los despachos de una bodega en el momento en que
ocurren, descuenta el stock en la base de datos, avisa por correo cuando un producto queda
por debajo de su mínimo, y usa **Google Gemini** para redactar el resumen semanal y para
atender consultas en lenguaje natural sobre los movimientos.

**Curso:** Automatización inteligente de procesos con n8n · EAM · Periodo 2026-II
**Profesor:** Robert Garcia Rey

| Integrante | Aporte |
|---|---|
| Sebastián Betancourth | Entorno, credenciales, versionado y fusión del workflow final |
| Felipe Castaño | Construcción del flujo: HTTP Request, validaciones y mapeo de datos |
| Juan Camilo Castañeda Lopera | Pruebas, casos de la bitácora 11 y evidencias |
| Mateo Albornoz | Documentación, diagramas y datos de prueba |

### El problema

En la bodega los despachos se anotaban en una hoja de cálculo y alguien tenía que revisarla
para darse cuenta de que un producto se estaba agotando. La revisión dependía de que esa
persona se acordara, así que las faltas se descubrían cuando ya no había existencias.

### Objetivo y alcance

Que cada despacho actualice el stock por sí solo y que el aviso salga en el momento en que
el producto cruza su mínimo, sin que nadie tenga que mirar nada.

Queda **dentro** del alcance: recepción de despachos por webhook autenticado, descuento de
stock, control de duplicados, alerta por correo, resumen semanal redactado por IA y un
agente de consulta de solo lectura.

Queda **fuera**: reposición automática de pedidos, interfaz propia de captura (se usa un
simulador para las pruebas) y escritura transaccional real entre las dos tablas, que se
explica en las limitaciones.

---

## Entregables

| Archivo | Qué contiene |
|---|---|
| [`workflows/Workflow_Final.json`](workflows/Workflow_Final.json) | El workflow definitivo, en un solo archivo |
| [`documentacion/Documentacion_Proyecto_Final.md`](documentacion/Documentacion_Proyecto_Final.md) | Explicación completa del proyecto y sus decisiones |
| [`pruebas/Bitacora_Clase_11_Pruebas.md`](pruebas/Bitacora_Clase_11_Pruebas.md) | Revisión integral, pruebas ejecutadas y pendientes |
| [`presentacion/Presentacion_Proyecto_Final.pdf`](presentacion/Presentacion_Proyecto_Final.pdf) | Presentación de la solución y sus resultados |
| [`documentacion/diagramas/`](documentacion/diagramas/) | Proceso manual y proceso automatizado |
| [`pruebas/evidencias/`](pruebas/evidencias/) | Capturas y archivos de cada caso |

---

## Vista general

El workflow tiene **tres puntos de entrada** en un mismo archivo:

**1. Webhook de movimiento** — `POST /webhook/inventario-movimiento`
Recibe un despacho, valida el contrato, comprueba si ya se procesó, busca el producto,
descuenta el stock, registra el movimiento y, si el producto quedó bajo el mínimo, envía la
alerta. Responde con uno de siete códigos distintos según lo que haya pasado.

**2. Resumen semanal** — disparador programado, lunes por la mañana
Calcula el periodo de la semana anterior, consulta los movimientos, **suma las cifras en un
nodo Code** y le pide a Gemini únicamente que redacte el texto. Después compara los números
del texto contra los calculados y marca los que no provienen del cálculo.

**3. Agente de consulta** — disparador de chat
Responde preguntas sobre los movimientos. Tiene una sola herramienta, de solo lectura, que
consulta un periodo y devuelve totales ya calculados.

### Qué hace la IA, y qué no

La IA **redacta**; no calcula. Los totales del resumen los produce un nodo `Code` a partir
de las filas de la base de datos, y el correo lleva las cifras crudas junto al texto para
que cualquiera pueda contrastarlas. El agente no tiene ninguna herramienta de escritura: no
puede crear, modificar ni borrar nada, ni siquiera si se le pide.

### Servicios

| Servicio | Para qué | Nodos |
|---|---|---|
| **Supabase** (PostgREST) | Tablas `Productos`, `Movimientos` y `Errores` | HTTP Request |
| **Gmail** | Alerta de stock y correo del resumen semanal | Gmail |
| **Google Gemini** | Redacción del resumen y agente de consulta | `gemini-3.5-flash-lite` |

---

## Requisitos

- **n8n 2.35.7** o superior, self-hosted. Se probó en `localhost:5678`.
- Una cuenta de **Supabase** con un proyecto propio.
- Una cuenta de **Google** para Gmail (OAuth2) y una **API key de Google AI Studio** para
  Gemini.
- Permisos de envío en la cuenta de Gmail que se use como remitente.

No hace falta un servidor público: el webhook se prueba contra la URL local.

---

## Configuración

### 1. Tablas en Supabase

En *SQL Editor*, ejecuten lo siguiente. Los nombres de columna llevan mayúsculas, así que
van siempre entre comillas dobles.

```sql
create table if not exists "Productos" (
  id                bigint generated always as identity primary key,
  sku               text    not null unique,
  producto          text    not null,
  bodega            text,
  responsable       text,
  "correoResponsable" text,
  "stockActual"     integer not null default 0,
  "stockMinimo"     integer not null default 0
);

create table if not exists "Movimientos" (
  id             bigint generated always as identity primary key,
  movimiento_id  text    not null unique,   -- impide el doble descuento
  sku            text    not null,
  cantidad       integer not null,
  motivo         text,
  stock_anterior integer,
  stock_nuevo    integer,
  alerta_enviada boolean default false,
  creado_en      timestamptz default now()
);

create table if not exists "Errores" (
  id        bigint generated always as identity primary key,
  workflow  text, nodo text, mensaje text,
  ejecucion text, modo text,
  creado_en timestamptz default now()
);
```

El `unique` sobre `movimiento_id` es lo que de verdad impide que un mismo despacho se
aplique dos veces. La consulta previa del workflow solo sirve para responder rápido.

**Las tres tablas deben tener la misma configuración de RLS.** Si `Productos` no lo tiene
activo, no lo activen en las otras o el HTTP Request devolverá 401.

Carguen algunas filas de prueba en `Productos`, con SKU ficticios:

```sql
insert into "Productos" (sku, producto, bodega, responsable, "correoResponsable",
                         "stockActual", "stockMinimo") values
  ('SIM-001', 'Producto de prueba A', 'Norte', 'Responsable A', 'correo@ejemplo.com', 50, 10),
  ('SIM-002', 'Producto de prueba B', 'Sur',   'Responsable B', 'correo@ejemplo.com', 12, 10);
```

### 2. Credenciales en n8n

Creen las suyas; las del grupo no están en este repositorio.

| Tipo de credencial | Dónde se obtiene |
|---|---|
| **Supabase API** | Supabase → *Project Settings → API*: la URL del proyecto y la `service_role key` |
| **Gmail OAuth2** | Google Cloud Console → credencial OAuth2, con la API de Gmail habilitada |
| **Google Gemini (PaLM) API** | [Google AI Studio](https://aistudio.google.com/app/apikey) |
| **Header Auth** | La inventan ustedes: nombre `x-api-key` y un valor propio |

### 3. Parámetros que hay que reemplazar

| Dónde | Qué cambiar |
|---|---|
| Los seis nodos HTTP Request | El host de Supabase que viene en el archivo, por el de su propio proyecto |
| `Webhook Movimiento` → *Options → Allowed Origins* | El origen real desde el que llamen. Ver la nota del simulador |
| *Settings → Error Workflow* | `Manejador_Errores`, una vez importado el anexo |

---

## Importación y ejecución

1. En n8n: *Workflows → Import from File* → `workflows/Workflow_Final.json`.
2. Abran cada nodo marcado con un triángulo rojo y **seleccionen sus credenciales**. Son
   diez nodos: cuatro de Supabase, uno de Header Auth, dos de Gmail, dos de Gemini y el
   nodo HTTP del marcado de alerta.
3. Reemplacen la URL de Supabase en los nodos HTTP Request.
4. Importen también [`documentacion/anexos/Manejador_Errores.json`](documentacion/anexos/Manejador_Errores.json)
   y luego, en el workflow principal, vayan a *Settings → Error Workflow* y selecciónenlo.
   No se lo asignen al propio manejador: cada fallo suyo volvería a dispararlo.
5. **Activen** el workflow. El disparador programado y el webhook de producción solo
   funcionan con el workflow activo.

### Antes de darlo por importado

Abran dos o tres nodos `Set` y confirmen que muestran **Mode: Manual Mapping**. Si aparece
*Keep Only Set*, la importación degradó la versión de los nodos y el stock no se descontará.
Ese fue el fallo que arrastramos durante el Corte 2 y está documentado en la bitácora 11.

### Cómo detener las ejecuciones programadas

El resumen semanal corre cada lunes mientras el workflow esté activo. Para pararlo,
desactiven el workflow desde el interruptor de la esquina superior derecha, o abran el nodo
`Lunes por la manana` y deshabilítenlo con `D`.

---

## Ejemplo reproducible

**Condición inicial.** La fila `SIM-002` tiene `stockActual = 12` y `stockMinimo = 10`.

**Entrada.** Despacho de 5 unidades:

```bash
curl -X POST http://localhost:5678/webhook/inventario-movimiento \
  -H "Content-Type: application/json" \
  -H "x-api-key: EL-VALOR-DE-SU-CREDENCIAL" \
  -d '{
        "movimiento_id": "SIM-MOV-0001",
        "sku": "SIM-002",
        "cantidad": 5,
        "motivo": "venta"
      }'
```

**Resultado esperado.** Respuesta `200`. En `Productos`, `SIM-002` queda con
`stockActual = 7`. En `Movimientos` aparece una fila con `movimiento_id = SIM-MOV-0001` y
`alerta_enviada = true`. Y llega un correo con el asunto
`ALERTA DE STOCK - Producto de prueba B (Sur)`, porque 7 quedó por debajo de 10.

**Dónde comprobarlo.** En el *Table Editor* de Supabase, en la bandeja de entrada del
destinatario, y en *Executions* de n8n.

**Repitan la misma petición.** La segunda vez responde `200` con `duplicado: true`, el
stock se queda en 7 y no se envía un segundo correo.

### El simulador

[`pruebas/simulador.html`](pruebas/simulador.html) es una página que envía estas peticiones
sin necesidad de `curl`, e incluye los casos de cuerpo incompleto, token ausente y SKU
inexistente.

El webhook restringe los orígenes, así que **no funciona abriéndolo con doble clic**: el
origen sería `null` y la petición se rechaza. Sírvanlo y usen esa misma dirección en
*Allowed Origins*:

```bash
cd pruebas && python -m http.server 5500
```

---

## Códigos de respuesta del webhook

| Código | Cuándo |
|---|---|
| `200` | Movimiento procesado, sin alerta |
| `200` | Movimiento procesado y alerta enviada |
| `200` | Movimiento procesado, alerta **no** enviada (`avisoPendiente: true`) |
| `200` | Duplicado: ya estaba procesado, no se descuenta de nuevo |
| `400` | Contrato inválido: campos vacíos, cantidad no entera o fuera de rango, motivo no admitido |
| `404` | El SKU no existe |
| `409` | Stock insuficiente para el despacho |
| `500` | El movimiento no se pudo registrar |
| `502` | No se pudo consultar Supabase, o el descuento no se confirmó. Cubre tanto un código de error del servicio como una conexión que no llega a abrirse |

Los `motivo` admitidos son `venta`, `despacho a sucursal`, `avería` y `ajuste de inventario`.
La cantidad debe ser un entero entre 1 y 10000.

---

## Limitaciones conocidas

**El manejador de errores va aparte.** Un nodo *Error Trigger* no puede convivir con otros
disparadores en el mismo workflow, así que `Manejador_Errores` se entrega como anexo en
`documentacion/anexos/`. El workflow final funciona sin él; lo que se pierde es el registro
automático de fallos en la tabla `Errores`.

**Queda una ventana entre dos escrituras.** El descuento del stock y el registro del
movimiento son dos peticiones distintas. El orden actual reduce el riesgo —se registra
inmediatamente después de descontar— pero una interrupción justo entre ambas dejaría el
stock descontado sin movimiento registrado. La solución completa sería una función de
Postgres invocada por RPC que hiciera las dos escrituras en una transacción, y el equipo la
dejó fuera del alcance del corte.

**La memoria del agente depende del prompt.** No hay un mecanismo que le impida reutilizar
una cifra de un turno anterior; solo la instrucción de volver a consultar.

---

## Dónde seguir leyendo

- [Documentación del proyecto](documentacion/Documentacion_Proyecto_Final.md) — el
  funcionamiento en detalle y las decisiones de diseño.
- [Bitácora de la clase 11](pruebas/Bitacora_Clase_11_Pruebas.md) — la revisión integral,
  los casos de prueba y el estado de los pendientes.
- [Presentación](presentacion/Presentacion_Proyecto_Final.pdf).
- [Diagramas](documentacion/diagramas/) — el proceso manual y el automatizado.

---

## Datos y privacidad

Los datos de inventario de este repositorio son ficticios: los productos de prueba llevan
el prefijo `SIM-`. No hay tokens, claves de API ni archivos `.env` en el repositorio ni en
su historial; las credenciales se crean en cada instalación siguiendo la sección de
configuración.

Dos valores del archivo sí son reales y conviene saberlo: el **host del proyecto de
Supabase**, que sin una clave válida solo devuelve `401`, y el **buzón de pruebas** que
recibe las alertas cuando un producto no tiene responsable asignado. Ese buzón se creó para
el curso y no pertenece a ninguna persona ajena al equipo. Quien monte su propia copia
puede cambiar ambos en los nodos correspondientes.
