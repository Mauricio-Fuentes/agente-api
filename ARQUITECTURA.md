# Arquitectura: agente de tipo API con Python, Pydantic AI y uv

Este documento describe la arquitectura propuesta para un agente HTTP en Python. El servicio recibe un mensaje, lo ejecuta con un agente de Pydantic AI y devuelve una respuesta validada. [uv](https://docs.astral.sh/uv/) gestiona el entorno y las dependencias. Todavía no hay código: este archivo fija el diseño antes de implementarlo.

## 1. Objetivo

Construir un proceso único que exponga un agente por HTTP.

- Una tarea de negocio concreta por versión. El agente no es un chat genérico sin límites.
- Una ruta de conversación, `POST /chat`, y una ruta de estado, `GET /health`.
- La salida del modelo siempre cumple un tipo Pydantic. La API no devuelve texto libre sin validar.
- El mismo proceso arranca con un modelo real o, si no hay clave, con `TestModel`.

## 2. Decisiones de diseño

| Decisión | Elección | Motivo |
| --- | --- | --- |
| Lenguaje | Python 3.11 o superior | Mínimo que exige Pydantic AI. |
| Gestor de proyecto | uv | Entorno reproducible, lockfile y `uv run`. |
| Agente | Pydantic AI (`pydantic-ai`) | Bucle del modelo, herramientas tipadas y salida validada. |
| Transporte | FastAPI + Uvicorn | Validación de entrada, códigos HTTP y documentación en `/docs`. |
| Configuración | Pydantic Settings | El modelo y la clave se leen del entorno al arrancar. |
| Sesión | Diccionario en memoria | Alcanza para la primera versión. La interfaz permite cambiar el almacén después. |
| Pruebas del modelo | `TestModel` y `FunctionModel` | Ejercitan herramientas y el contrato HTTP sin llamar a un proveedor. |

Quedan fuera de esta versión: autenticación de la API, base de datos, observabilidad (Logfire), evaluaciones, interfaz de chat y más de un agente.

## 3. Vista general

```text
Cliente
  │  POST /chat
  │  GET  /health
  ▼
FastAPI                         transporte y validación HTTP
  │
  ▼
Servicio de chat                sesión, dependencias, llamada a agent.run
  │
  ├── Almacén de sesiones       historial por session_id
  ▼
Agent (Pydantic AI)             instrucciones, herramientas, output_type
  │
  ├── Modelo                    proveedor real, o TestModel si no hay clave
  └── Herramientas              funciones tipadas del dominio
        │
        ▼
      Dependencias              cliente HTTP, catálogo, reloj, lo que el dominio necesite
```

El agente se crea una vez al arrancar el proceso, igual que la aplicación de FastAPI. Cada petición le entrega el mensaje, el historial de esa sesión y las dependencias. Las herramientas no leen variables globales: reciben lo que necesitan por `RunContext.deps`.

## 4. Flujo de un mensaje

1. El cliente envía `POST /chat` con el texto y, si continúa una conversación, un `session_id`.
2. FastAPI valida el cuerpo con `ChatRequest`. Un cuerpo inválido responde `422` y no llega al agente.
3. El servicio busca el historial de esa sesión. Si no viene `session_id`, abre una sesión nueva y genera el identificador.
4. Construye las dependencias de esa petición y llama a `agent.run` con el mensaje y el historial.
5. El modelo puede llamar una herramienta. Pydantic AI valida los argumentos contra la firma de la función, la ejecuta y devuelve el resultado al modelo.
6. El turno termina cuando el modelo produce un valor del `output_type`. Si esa salida no valida, la petición falla y el servicio no guarda un turno a medias.
7. El servicio persiste el historial actualizado y responde `ChatResponse`: la salida del agente, el `session_id` y el modo (`live` o `test`).

```text
Cliente          API            Servicio         Sesiones        Agent           Modelo / herramientas
  │  POST /chat    │                │                │               │                    │
  │───────────────►│ valida         │                │               │                    │
  │                │───────────────►│ get(session)   │               │                    │
  │                │                │───────────────►│               │                    │
  │                │                │ run(mensaje, historial, deps)  │                    │
  │                │                │───────────────────────────────►│ solicitud          │
  │                │                │                                │───────────────────►│
  │                │                │                                │◄──── tool result ──│
  │                │                │                                │◄──── output_type ──│
  │                │                │ put(historial) │               │                    │
  │                │                │───────────────►│               │                    │
  │◄───────────────│ 200 ChatResponse                │               │                    │
```

`GET /health` no ejecuta el agente. Comprueba que el proceso está vivo y reporta si la configuración tiene clave de modelo.

## 5. Capas

### 5.1 API

Responsabilidad: rutas, códigos HTTP y modelos de entrada y salida.

- Traduce el cuerpo HTTP a la llamada del servicio.
- Traduce el resultado del servicio a JSON.
- No contiene instrucciones del agente ni reglas de negocio.
- No elige el modelo.

Rutas de esta versión:

| Método | Ruta | Uso |
| --- | --- | --- |
| `GET` | `/health` | Proceso vivo y modo del modelo (`live` o `test`). |
| `POST` | `/chat` | Un turno de conversación. |
| `GET` | `/docs` | Documentación interactiva que genera FastAPI. |

### 5.2 Servicio de chat

Responsabilidad: unir la petición HTTP con el agente.

- Abre o recupera la sesión.
- Construye las dependencias de la petición.
- Llama a `agent.run`.
- Guarda el historial solo después de una salida válida.
- Convierte fallos del agente en errores de aplicación que la API ya sabe expresar como HTTP.

El servicio es el único llamador de `agent.run` en el proceso. Los tests de HTTP pasan por él. Los tests de herramientas pueden llamarlas directo.

### 5.3 Agente

Responsabilidad: un solo `Agent` de Pydantic AI.

- Identificador del modelo, leído de la configuración.
- Instrucciones fijas: identidad, cuándo usar cada herramienta y la forma de la respuesta.
- `output_type`: un modelo Pydantic del dominio.
- Herramientas registradas con decorador.
- `deps_type`: el tipo de las dependencias que el servicio inyecta.

Las instrucciones viven junto al agente. No se arman dentro de las rutas.

### 5.4 Dominio y herramientas

Responsabilidad: las acciones que el modelo puede pedir.

- Cada herramienta es una función con tipos y un docstring. El docstring le dice al modelo cuándo usarla.
- `@agent.tool_plain` cuando la función no necesita estado de la petición.
- `@agent.tool` cuando necesita dependencias. El primer argumento es `RunContext` y lee `ctx.deps`.
- Los argumentos llegan validados. La función no reinterpreta JSON crudo.
- Lo que no esté registrado como herramienta no es una capacidad del agente.

La primera versión incluye una o dos herramientas. Añadir otra es un archivo o una función nueva, más una línea en las instrucciones que diga cuándo usarla.

### 5.5 Configuración

Responsabilidad: leer el entorno una vez, al arrancar.

Valores:

| Variable | Papel |
| --- | --- |
| Identificador del modelo | Por ejemplo `openai:gpt-5.2`. Lo consume el `Agent`. |
| Clave del proveedor | `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` u otra, según el proveedor elegido. No se escribe en el repositorio. |
| Modo | Derivado: hay clave y modelo configurado, entonces `live`. Si falta la clave, `test`. |

En modo `test` el agente usa `TestModel`. El servicio sigue respondiendo y `/health` informa el modo. Así se puede recorrer el contrato HTTP sin gastar tokens.

La clave no aparece en logs, en el cuerpo de error ni en `/health`.

### 5.6 Sesiones

Responsabilidad: guardar el historial de mensajes de Pydantic AI por `session_id`.

Interfaz mínima:

- `get(session_id) -> historial | vacío`
- `put(session_id, historial) -> None`
- `create() -> session_id`

La implementación inicial es un diccionario en el proceso. Se pierde al reiniciar y no se comparte entre procesos. Esa es la única pieza que se sustituye cuando haga falta un almacén durable. Las rutas y la definición del agente no cambian.

El historial que se guarda es el que devuelve el agente tras un turno válido. Un turno que falla no reemplaza el historial anterior.

## 6. Módulos

```text
pyproject.toml                 uv. Dependencias: pydantic-ai, fastapi, uvicorn
                               Desarrollo: pytest
src/agent_api/
  main.py                      Crea settings, agente, sesiones y la app una sola vez
  config.py                    Settings desde el entorno y elección del modelo
  api/
    routes.py                  GET /health y POST /chat
    schemas.py                 ChatRequest, ChatResponse, HealthResponse
  agent/
    assistant.py               Agent, instrucciones y output_type
    deps.py                    Tipo de las dependencias de una petición
    tools.py                   Herramientas del dominio
  sessions.py                  Interfaz y almacén en memoria
tests/
  test_chat.py                 POST /chat con TestModel, sin red
  test_tools.py                Herramientas llamadas con argumentos fijos
```

`main.py` es el punto de composición. El resto de los módulos recibe sus colaboradores por argumento o por el estado de la app, no creando el agente otra vez en cada petición.

## 7. Contratos

La API y el agente no comparten el mismo modelo. El contrato HTTP lleva metadatos que el modelo no rellena.

### 7.1 Entrada HTTP: `ChatRequest`

| Campo | Tipo | Obligatorio | Descripción |
| --- | --- | --- | --- |
| `message` | texto | sí | Turno del usuario. Vacío se rechaza. |
| `session_id` | texto | no | Si viene, el turno continúa esa sesión. Si no, se crea una. |

### 7.2 Salida del agente: `AgentReply`

La define el dominio. La forma concreta depende de la tarea. La estructura base de esta arquitectura es:

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `answer` | texto | Respuesta al usuario. |
| `sources` | lista de texto | Herramientas o datos que sostienen la respuesta. Vacía si no usó ninguna. |

El `output_type` del `Agent` es `AgentReply`. Pydantic AI rechaza una salida que no cumpla ese modelo.

### 7.3 Salida HTTP: `ChatResponse`

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `session_id` | texto | Sesión creada o continuada. El cliente la reenvía en el siguiente turno. |
| `reply` | `AgentReply` | Salida validada del agente. |
| `mode` | `live` o `test` | Si el turno usó el proveedor real o `TestModel`. |

### 7.4 Salud: `HealthResponse`

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `status` | texto | `ok` cuando el proceso responde. |
| `mode` | `live` o `test` | Igual que en el chat. No incluye la clave ni el nombre secreto. |

### 7.5 Errores

| Situación | HTTP | Cuerpo |
| --- | --- | --- |
| JSON o campos inválidos | 422 | Detalle de validación de FastAPI. |
| `session_id` desconocido | 404 | `session_id` no existe. |
| El modelo no produjo un `AgentReply` válido | 502 | Error genérico de salida. Sin texto interno del proveedor. |
| El proveedor rechazó la llamada (clave, cupo, red) | 502 | Error genérico de modelo. |
| Fallo inesperado | 500 | Error genérico. |

Los mensajes de error no incluyen la clave, el historial completo ni el prompt del sistema.

## 8. Modelo

`config.py` elige la implementación al arrancar y la entrega ya construida.

- Modo `live`: el `Agent` recibe el identificador configurado, por ejemplo `openai:gpt-5.2`. La clave la lee el proveedor desde el entorno.
- Modo `test`: el `Agent` recibe `TestModel`. No hay llamada de red.

Cambiar de proveedor es cambiar el identificador y la variable de la clave. Las rutas, el servicio y las herramientas siguen iguales.

En los tests se fuerza además:

- `ALLOW_MODEL_REQUESTS = False`, para que un descuido no llame a un proveedor.
- `Agent.override(model=TestModel())` cuando el caso solo necesita una salida válida.
- `Agent.override(model=FunctionModel(...))` cuando el caso debe comprobar qué herramienta se llamó y con qué argumentos.

## 9. Dependencias de una petición

`deps.py` define un tipo pequeño, construido por el servicio en cada turno. Contiene solo lo que las herramientas leen.

Ejemplos de campos, según el dominio que se elija al implementar:

- un cliente HTTP ya configurado;
- un catálogo o repositorio en memoria;
- un reloj, para que los tests no dependan de la hora real.

El modelo no ve este objeto. Lo ve la herramienta a través de `RunContext`.

## 10. Arranque y ejecución

Comandos previstos, cuando exista el código:

```bash
uv sync
uv run uvicorn agent_api.main:app --host 127.0.0.1 --port 4317
```

Comprobación manual:

1. `GET /health` responde `ok` y el modo.
2. `POST /chat` con un mensaje y sin `session_id` devuelve `reply` y un `session_id` nuevo.
3. Un segundo `POST /chat` con ese `session_id` continúa la misma conversación.
4. Con la clave en el entorno, se reinicia el proceso y el mismo `POST` corre en modo `live`.

El puerto `4317` evita chocar con los puertos habituales de desarrollo. Se puede cambiar por variable de entorno si hace falta.

## 11. Pruebas

Dos niveles, ambos sin red.

**Herramientas.** Llaman a la función con argumentos fijos y un `RunContext` de prueba. Comprueban el valor de retorno y los casos de borde del dominio (dato ausente, argumento vacío).

**HTTP.** El cliente de pruebas de FastAPI envía `POST /chat`. El agente está sobreescrito con `TestModel` o `FunctionModel`.

Casos mínimos de HTTP:

- mensaje válido sin sesión: `200`, `reply` con la forma de `AgentReply` y un `session_id`;
- segundo mensaje con ese `session_id`: `200` y el mismo identificador;
- `message` vacío o ausente: `422`;
- `session_id` que no existe: `404`;
- sin clave de modelo: `/health` reporta `test` y `/chat` sigue respondiendo.

## 12. Fronteras de evolución

Estas piezas se pueden sustituir sin redibujar el servicio:

| Pieza actual | Sustitución posterior | Qué no se toca |
| --- | --- | --- |
| Diccionario de sesiones | Almacén durable con la misma interfaz `get` / `put` / `create` | Rutas y `Agent`. |
| `TestModel` como respaldo | Siempre exigir clave y fallar el arranque si falta | Contrato de `/chat`. |
| Una o dos herramientas | Nuevas funciones registradas en el mismo agente | `ChatRequest`. |
| Sin autenticación | Dependencia de FastAPI delante del servicio | Instrucciones y herramientas. |

Observabilidad y evaluaciones se añaden cuando `POST /chat` ya devuelve `AgentReply` de forma estable. No forman parte del camino de la petición en esta versión.

## 13. Resumen

Un proceso. FastAPI en el borde. Un servicio que administra la sesión. Un `Agent` de Pydantic AI con instrucciones, herramientas tipadas y `AgentReply` como salida. uv instala y arranca ese proceso. La ausencia de clave degrada el modelo a `TestModel` y lo declara en `/health`, sin cambiar el contrato.
