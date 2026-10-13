# HTTPX: Guía PRO

> Guía práctica de **httpx**, de la instalación a producción: implementación, patrones, rendimiento, seguridad y testing.
> Versión de referencia: **httpx 0.28.1** (última estable). Requiere Python 3.8+.

---

## Índice

1. [¿Qué es httpx y por qué usarlo?](#1-qué-es-httpx-y-por-qué-usarlo)
2. [Instalación](#2-instalación)
3. [Primeros pasos](#3-primeros-pasos)
4. [El objeto `Client` / `AsyncClient`](#4-el-objeto-client--asyncclient)
5. [Peticiones: params, headers, body, archivos](#5-peticiones-params-headers-body-archivos)
6. [Respuestas](#6-respuestas)
7. [Timeouts](#7-timeouts)
8. [Pool de conexiones y límites](#8-pool-de-conexiones-y-límites)
9. [Manejo de errores](#9-manejo-de-errores)
10. [Reintentos y resiliencia](#10-reintentos-y-resiliencia)
11. [Autenticación](#11-autenticación)
12. [Event hooks (interceptores)](#12-event-hooks-interceptores)
13. [Streaming (descargas y subidas grandes)](#13-streaming-descargas-y-subidas-grandes)
14. [Async y concurrencia](#14-async-y-concurrencia)
15. [HTTP/2](#15-http2)
16. [Proxies y transports](#16-proxies-y-transports)
17. [Seguridad](#17-seguridad)
18. [Integración con FastAPI](#18-integración-con-fastapi)
19. [Patrón: cliente tipado tipo Feign](#19-patrón-cliente-tipado-tipo-feign)
20. [Testing](#20-testing)
21. [Logging y observabilidad](#21-logging-y-observabilidad)
22. [Migrar desde `requests`](#22-migrar-desde-requests)
23. [Cambios importantes en 0.28 y httpx 1.0](#23-cambios-importantes-en-028-y-httpx-10)
24. [Errores comunes (anti-patrones)](#24-errores-comunes-anti-patrones)
25. [Checklist de producción](#25-checklist-de-producción)

---

## 1. ¿Qué es httpx y por qué usarlo?

httpx es un cliente HTTP moderno para Python con API **sync y async**, soporte **HTTP/1.1 y HTTP/2**, tipado completo y timeouts estrictos por defecto. Su API es casi compatible con `requests`.

| Característica | `requests` | `httpx` | `aiohttp` |
|---|---|---|---|
| API síncrona | ✅ | ✅ | ❌ |
| API asíncrona | ❌ | ✅ | ✅ |
| HTTP/2 | ❌ | ✅ (extra) | ❌ |
| Timeout por defecto | ❌ (espera infinita) | ✅ 5 s | ✅ 5 min |
| Type hints | Parcial | ✅ Completo | ✅ |
| Llamar apps ASGI/WSGI en memoria | ❌ | ✅ | ❌ |
| Mock nativo para tests | ❌ | ✅ `MockTransport` | ❌ |
| Servidor HTTP incluido | ❌ | ❌ | ✅ |

### Equivalencias con Spring

| Spring | httpx |
|---|---|
| `RestTemplate` / `RestClient` | `httpx.Client` |
| `WebClient` (reactivo) | `httpx.AsyncClient` |
| `@FeignClient` | Clase propia que envuelve un `AsyncClient` ([sección 19](#19-patrón-cliente-tipado-tipo-feign)) |
| `ClientHttpRequestInterceptor` | `event_hooks` o un `Transport` personalizado |
| `ConnectionPool` de Apache/OkHttp | `httpx.Limits` |
| Resilience4j `@Retry` | `tenacity` |
| `MockRestServiceServer` / WireMock | `httpx.MockTransport` o `respx` |

---

## 2. Instalación

```bash
pip install httpx
```

### Extras opcionales

| Extra | Instala | Para qué |
|---|---|---|
| `httpx[http2]` | `h2` | Soporte HTTP/2 |
| `httpx[brotli]` | `brotli`/`brotlicffi` | Descomprimir respuestas `br` |
| `httpx[zstd]` | `zstandard` | Descomprimir respuestas `zstd` |
| `httpx[socks]` | `socksio` | Proxies SOCKS |
| `httpx[cli]` | `click`, `rich` | Cliente de línea de comandos |

```bash
pip install "httpx[http2,brotli]"
```

### Fija la versión (recomendado)

En PyPI existen pre-releases `1.0.devN` con cambios de API. Para producción, fija la rama estable:

```toml
# pyproject.toml
[project]
dependencies = [
  "httpx[http2]>=0.28,<0.29",
]
```

```text
# requirements.txt
httpx[http2]>=0.28,<0.29
```

### CLI (bonus)

```bash
pip install "httpx[cli]"
httpx https://httpbin.org/json
httpx https://httpbin.org/post -m POST -j '{"nombre": "Gil"}'
```

---

## 3. Primeros pasos

### Síncrono

```python
import httpx

r = httpx.get("https://httpbin.org/get", params={"q": "httpx"})
print(r.status_code)        # 200
print(r.json()["args"])     # {'q': 'httpx'}
```

### Asíncrono

```python
import asyncio
import httpx

async def main():
    async with httpx.AsyncClient() as client:
        r = await client.get("https://httpbin.org/get")
        print(r.json())

asyncio.run(main())
```

> ⚠️ Las funciones de módulo (`httpx.get`, `httpx.post`…) crean **una conexión nueva por llamada**. Sirven para scripts y pruebas rápidas. En una aplicación real, usa siempre un `Client` reutilizable (siguiente sección).

---

## 4. El objeto `Client` / `AsyncClient`

El `Client` es el equivalente a un `RestClient` configurado como bean: mantiene el **pool de conexiones** (keep-alive), cookies y configuración compartida.

```python
import httpx

client = httpx.Client(
    base_url="https://api.ejemplo.com/v1",
    headers={"User-Agent": "mi-app/1.0", "Accept": "application/json"},
    timeout=httpx.Timeout(10.0, connect=5.0),
    limits=httpx.Limits(max_connections=100, max_keepalive_connections=20),
    follow_redirects=False,   # por defecto ya es False
    http2=True,               # requiere httpx[http2]
)

try:
    r = client.get("/usuarios/1")       # -> https://api.ejemplo.com/v1/usuarios/1
    r.raise_for_status()
finally:
    client.close()
```

### Mejor con context manager

```python
with httpx.Client(base_url="https://api.ejemplo.com") as client:
    r = client.get("/salud")

async with httpx.AsyncClient(base_url="https://api.ejemplo.com") as client:
    r = await client.get("/salud")
```

### Parámetros principales

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `base_url` | `str` | `""` | Prefijo para rutas relativas |
| `headers` | `dict` | — | Headers enviados en cada petición |
| `params` | `dict` | — | Query params por defecto |
| `cookies` | `dict` | — | Cookies iniciales |
| `auth` | `Auth` / tupla | — | Autenticación por defecto |
| `timeout` | `float` / `Timeout` | 5 s | Timeouts |
| `limits` | `Limits` | 100 / 20 | Tamaño del pool |
| `verify` | `bool` / `SSLContext` | `True` | Verificación TLS |
| `http2` | `bool` | `False` | Habilitar HTTP/2 |
| `follow_redirects` | `bool` | `False` | Seguir redirecciones |
| `proxy` | `str` / `Proxy` | — | Proxy para todas las peticiones |
| `transport` | `BaseTransport` | — | Transport personalizado (reintentos, mocks…) |
| `event_hooks` | `dict` | — | Hooks `request` / `response` |
| `trust_env` | `bool` | `True` | Leer `HTTP_PROXY`, `SSL_CERT_FILE`, `.netrc`… |

### Sobrescribir por petición

Todo se puede sobrescribir en cada llamada:

```python
client.get("/reporte", timeout=60.0, headers={"X-Trace-Id": "abc"})
```

---

## 5. Peticiones: params, headers, body, archivos

```python
# Query params (listas -> ?tag=a&tag=b)
client.get("/buscar", params={"q": "laptop", "tag": ["a", "b"], "page": 2})

# JSON
client.post("/usuarios", json={"nombre": "Gil", "email": "gil@ejemplo.com"})

# Form URL-encoded
client.post("/login", data={"user": "gil", "pass": "secreto"})

# Bytes / texto crudo
client.post("/raw", content=b"\x00\x01", headers={"Content-Type": "application/octet-stream"})

# Multipart: archivo + campos
with open("foto.jpg", "rb") as f:
    client.post(
        "/upload",
        files={"archivo": ("foto.jpg", f, "image/jpeg")},
        data={"descripcion": "Perfil"},
    )

# Varios archivos con el mismo campo
files = [
    ("docs", ("a.pdf", open("a.pdf", "rb"), "application/pdf")),
    ("docs", ("b.pdf", open("b.pdf", "rb"), "application/pdf")),
]
client.post("/upload-multiple", files=files)

# Métodos
client.put("/usuarios/1", json={...})
client.patch("/usuarios/1", json={"email": "nuevo@ejemplo.com"})
client.delete("/usuarios/1")
client.head("/archivo")
client.options("/recurso")

# Método genérico
client.request("PATCH", "/usuarios/1", json={...})
```

### Construir la petición sin enviarla

Útil para firmar peticiones o inspeccionarlas:

```python
request = client.build_request("POST", "/pagos", json={"monto": 100})
request.headers["X-Firma"] = firmar(request.content)
response = client.send(request)
```

---

## 6. Respuestas

```python
r = client.get("/usuarios/1")

r.status_code              # 200
r.reason_phrase            # "OK"
r.is_success               # True para 2xx
r.is_error                 # True para 4xx/5xx
r.headers["content-type"]  # headers case-insensitive
r.json()                   # dict / list
r.text                     # str (decodificado según charset)
r.content                  # bytes
r.url                      # URL final (tras redirects)
r.elapsed                  # timedelta de la petición
r.http_version             # "HTTP/1.1" o "HTTP/2"
r.history                  # lista de respuestas de redirección
r.request                  # el Request original

r.raise_for_status()       # lanza HTTPStatusError en 4xx/5xx; devuelve la respuesta si OK
```

Encadenado (desde 0.25, `raise_for_status()` devuelve la propia respuesta):

```python
data = client.get("/usuarios/1").raise_for_status().json()
```

### Mapear a Pydantic

```python
from pydantic import BaseModel, TypeAdapter

class Usuario(BaseModel):
    id: int
    nombre: str
    email: str

usuario = Usuario.model_validate(r.json())
usuarios = TypeAdapter(list[Usuario]).validate_python(client.get("/usuarios").json())

# Más rápido: valida directamente los bytes JSON
usuario = Usuario.model_validate_json(r.content)
```

---

## 7. Timeouts

httpx aplica **5 segundos por defecto** a todo. Nunca uses `timeout=None` en producción.

| Tipo | Qué mide |
|---|---|
| `connect` | Establecer la conexión TCP/TLS |
| `read` | Tiempo máximo entre bloques de datos recibidos |
| `write` | Tiempo máximo entre bloques de datos enviados |
| `pool` | Esperar una conexión libre del pool |

```python
# 10 s para todo, pero 3 s para conectar
timeout = httpx.Timeout(10.0, connect=3.0)

# Todo explícito
timeout = httpx.Timeout(connect=3.0, read=30.0, write=10.0, pool=5.0)

client = httpx.AsyncClient(timeout=timeout)

# Por petición (p. ej. un reporte lento)
await client.get("/reporte-pesado", timeout=httpx.Timeout(120.0, connect=3.0))
```

> 💡 `read` **no** es el tiempo total de la respuesta: es el tiempo entre bloques. Si necesitas un límite de tiempo total, envuelve la llamada:
>
> ```python
> import asyncio
> r = await asyncio.wait_for(client.get("/lento"), timeout=15)
> ```

---

## 8. Pool de conexiones y límites

```python
limits = httpx.Limits(
    max_connections=100,           # conexiones simultáneas totales
    max_keepalive_connections=20,  # conexiones ociosas que se mantienen abiertas
    keepalive_expiry=5.0,          # segundos que vive una conexión ociosa
)
client = httpx.AsyncClient(limits=limits)
```

**Cómo dimensionarlo:**

- `max_connections` ≈ número de peticiones concurrentes que esperas hacia ese servicio.
- Si ves `PoolTimeout`, el pool está saturado: sube `max_connections` o limita la concurrencia con un semáforo ([sección 14](#14-async-y-concurrencia)).
- Usa **un cliente por servicio destino** si cada uno necesita una configuración distinta (timeouts, auth, límites).

---

## 9. Manejo de errores

### Jerarquía de excepciones

```text
HTTPError
├── RequestError                 # algo falló al enviar/recibir
│   ├── TransportError
│   │   ├── TimeoutException
│   │   │   ├── ConnectTimeout
│   │   │   ├── ReadTimeout
│   │   │   ├── WriteTimeout
│   │   │   └── PoolTimeout
│   │   ├── NetworkError
│   │   │   ├── ConnectError
│   │   │   ├── ReadError
│   │   │   ├── WriteError
│   │   │   └── CloseError
│   │   ├── ProtocolError
│   │   │   ├── LocalProtocolError
│   │   │   └── RemoteProtocolError
│   │   ├── ProxyError
│   │   └── UnsupportedProtocol
│   ├── DecodingError
│   └── TooManyRedirects
└── HTTPStatusError              # raise_for_status() con 4xx/5xx

InvalidURL, CookieConflict, StreamError (y subclases)  # errores de uso
```

### Patrón recomendado

```python
import logging
import httpx

log = logging.getLogger(__name__)

class ServicioNoDisponible(Exception): ...
class RecursoNoEncontrado(Exception): ...

async def obtener_usuario(client: httpx.AsyncClient, user_id: int) -> dict:
    try:
        r = await client.get(f"/usuarios/{user_id}")
        r.raise_for_status()
        return r.json()

    except httpx.HTTPStatusError as e:
        status = e.response.status_code
        if status == 404:
            raise RecursoNoEncontrado(f"Usuario {user_id} no existe") from e
        if status >= 500:
            raise ServicioNoDisponible(f"Error {status} en servicio de usuarios") from e
        raise  # otros 4xx: error del llamador

    except httpx.TimeoutException as e:
        log.warning("Timeout consultando usuario %s: %s", user_id, type(e).__name__)
        raise ServicioNoDisponible("Timeout") from e

    except httpx.RequestError as e:
        # ConnectError, ReadError, etc. e.request siempre está disponible
        log.error("Error de red hacia %s: %s", e.request.url, e)
        raise ServicioNoDisponible("Error de red") from e
```

---

## 10. Reintentos y resiliencia

### Opción A: reintentos nativos (solo errores de conexión)

```python
transport = httpx.AsyncHTTPTransport(retries=3)
client = httpx.AsyncClient(transport=transport)
```

> ⚠️ `retries` **solo** reintenta `ConnectError` y `ConnectTimeout`. **No** reintenta 5xx ni `ReadTimeout`.

### Opción B: `tenacity` (control total, recomendado)

```bash
pip install tenacity
```

```python
import httpx
from tenacity import (
    retry, stop_after_attempt, wait_exponential_jitter,
    retry_if_exception, before_sleep_log,
)
import logging

log = logging.getLogger(__name__)

RETRYABLE_STATUS = {429, 502, 503, 504}

def es_reintentable(exc: BaseException) -> bool:
    if isinstance(exc, (httpx.ConnectError, httpx.ConnectTimeout,
                        httpx.ReadTimeout, httpx.RemoteProtocolError)):
        return True
    if isinstance(exc, httpx.HTTPStatusError):
        return exc.response.status_code in RETRYABLE_STATUS
    return False

reintentar = retry(
    retry=retry_if_exception(es_reintentable),
    stop=stop_after_attempt(4),
    wait=wait_exponential_jitter(initial=0.5, max=8),
    before_sleep=before_sleep_log(log, logging.WARNING),
    reraise=True,
)

@reintentar
async def get_json(client: httpx.AsyncClient, url: str) -> dict:
    r = await client.get(url)
    r.raise_for_status()
    return r.json()
```

### Respetar `Retry-After` (429/503)

```python
from tenacity import wait_exponential_jitter
from tenacity.wait import wait_base

class WaitRetryAfter(wait_base):
    def __init__(self, fallback: wait_base):
        self.fallback = fallback

    def __call__(self, retry_state):
        exc = retry_state.outcome.exception()
        if isinstance(exc, httpx.HTTPStatusError):
            ra = exc.response.headers.get("Retry-After")
            if ra and ra.isdigit():
                return min(float(ra), 30.0)   # tope de seguridad
        return self.fallback(retry_state)

# uso: wait=WaitRetryAfter(wait_exponential_jitter(initial=0.5, max=8))
```

> 🔐 **Idempotencia**: solo reintenta automáticamente métodos idempotentes (`GET`, `PUT`, `DELETE`, `HEAD`). Para `POST`, usa una `Idempotency-Key` si la API la soporta:
>
> ```python
> import uuid
> await client.post("/pagos", json=pago, headers={"Idempotency-Key": str(uuid.uuid4())})
> ```

### Circuit breaker

Para un patrón tipo Resilience4j `@CircuitBreaker`, puedes usar `purgatory` o `aiobreaker`, o implementar un contador simple de fallos con ventana de tiempo.

---

## 11. Autenticación

### Básica, Digest y Bearer

```python
# Basic
client = httpx.Client(auth=("usuario", "password"))
client = httpx.Client(auth=httpx.BasicAuth("usuario", "password"))

# Digest
client = httpx.Client(auth=httpx.DigestAuth("usuario", "password"))

# Bearer estático (header fijo)
client = httpx.Client(headers={"Authorization": f"Bearer {token}"})
```

### Auth personalizada con refresh automático de token

`httpx.Auth` funciona como un generador: entrega la petición, recibe la respuesta y puede reintentar.

```python
import threading
import httpx

class BearerRefreshAuth(httpx.Auth):
    """Añade el token y, si recibe 401, lo renueva y reintenta UNA vez."""

    def __init__(self, token_url: str, client_id: str, client_secret: str):
        self.token_url = token_url
        self.client_id = client_id
        self.client_secret = client_secret
        self._token: str | None = None
        self._lock = threading.Lock()

    def _fetch_token(self) -> str:
        r = httpx.post(
            self.token_url,
            data={"grant_type": "client_credentials"},
            auth=(self.client_id, self.client_secret),
            timeout=10,
        )
        r.raise_for_status()
        return r.json()["access_token"]

    def auth_flow(self, request: httpx.Request):
        with self._lock:
            if self._token is None:
                self._token = self._fetch_token()
        request.headers["Authorization"] = f"Bearer {self._token}"

        response = yield request

        if response.status_code == 401:
            with self._lock:
                self._token = self._fetch_token()
            request.headers["Authorization"] = f"Bearer {self._token}"
            yield request
```

### Versión async (con `async_auth_flow`)

```python
import asyncio
import httpx

class AsyncBearerRefreshAuth(httpx.Auth):
    def __init__(self, token_url: str, client_id: str, client_secret: str):
        self.token_url = token_url
        self.creds = (client_id, client_secret)
        self._token: str | None = None
        self._lock = asyncio.Lock()

    async def _fetch_token(self) -> str:
        async with httpx.AsyncClient(timeout=10) as c:
            r = await c.post(self.token_url, data={"grant_type": "client_credentials"}, auth=self.creds)
            r.raise_for_status()
            return r.json()["access_token"]

    async def async_auth_flow(self, request: httpx.Request):
        async with self._lock:
            if self._token is None:
                self._token = await self._fetch_token()
        request.headers["Authorization"] = f"Bearer {self._token}"

        response = yield request

        if response.status_code == 401:
            async with self._lock:
                self._token = await self._fetch_token()
            request.headers["Authorization"] = f"Bearer {self._token}"
            yield request

client = httpx.AsyncClient(auth=AsyncBearerRefreshAuth(
    "https://auth.ejemplo.com/oauth/token", "mi-id", "mi-secreto"))
```

> Para OAuth2 completo (authorization code, PKCE…), usa **Authlib**, que se integra con httpx: `from authlib.integrations.httpx_client import AsyncOAuth2Client`.

### mTLS (certificado de cliente)

```python
import ssl
import httpx

ctx = ssl.create_default_context()
ctx.load_cert_chain(certfile="cliente.crt", keyfile="cliente.key")
client = httpx.Client(verify=ctx)
```

---

## 12. Event hooks (interceptores)

Equivalen a los interceptores de Spring. Las funciones reciben el `Request` o el `Response`.

```python
import logging
import time
import httpx

log = logging.getLogger("http.out")

async def log_request(request: httpx.Request):
    request.extensions["inicio"] = time.perf_counter()
    log.info("→ %s %s", request.method, request.url)

async def log_response(response: httpx.Response):
    inicio = response.request.extensions.get("inicio", time.perf_counter())
    ms = (time.perf_counter() - inicio) * 1000
    log.info("← %s %s %d (%.0f ms)",
             response.request.method, response.request.url, response.status_code, ms)

async def raise_on_4xx_5xx(response: httpx.Response):
    if response.is_error:
        await response.aread()      # obligatorio para leer el body dentro del hook
        response.raise_for_status()

client = httpx.AsyncClient(
    event_hooks={
        "request": [log_request],
        "response": [log_response, raise_on_4xx_5xx],
    }
)
```

> - En `AsyncClient`, los hooks **deben ser `async def`**.
> - Dentro de un hook de respuesta, el body aún no se ha leído: usa `response.read()` (sync) o `await response.aread()` (async) antes de acceder a `.json()` o `.text`.

### Propagar un trace-id (correlación)

```python
import contextvars
import uuid

trace_id_var: contextvars.ContextVar[str] = contextvars.ContextVar("trace_id", default="")

async def add_trace_id(request: httpx.Request):
    request.headers["X-Trace-Id"] = trace_id_var.get() or str(uuid.uuid4())

client = httpx.AsyncClient(event_hooks={"request": [add_trace_id]})
```

---

## 13. Streaming (descargas y subidas grandes)

### Descargar sin cargar todo en memoria

```python
import httpx

async def descargar(client: httpx.AsyncClient, url: str, destino: str):
    async with client.stream("GET", url) as r:
        r.raise_for_status()
        total = int(r.headers.get("Content-Length", 0))
        descargado = 0
        with open(destino, "wb") as f:
            async for chunk in r.aiter_bytes(chunk_size=64 * 1024):
                f.write(chunk)
                descargado += len(chunk)
                if total:
                    print(f"\r{descargado * 100 // total}%", end="")
```

| Método (sync / async) | Devuelve |
|---|---|
| `iter_bytes()` / `aiter_bytes()` | Bytes **descomprimidos** |
| `iter_raw()` / `aiter_raw()` | Bytes **crudos** (sin descomprimir) |
| `iter_text()` / `aiter_text()` | Texto decodificado |
| `iter_lines()` / `aiter_lines()` | Línea por línea (ideal para SSE / NDJSON) |

### Consumir un stream de LLM (por ejemplo Ollama, NDJSON)

```python
import json
import httpx

async def chat_stream(prompt: str):
    async with httpx.AsyncClient(base_url="http://localhost:11434", timeout=None) as c:
        async with c.stream("POST", "/api/chat", json={
            "model": "llama3.2:3b",
            "messages": [{"role": "user", "content": prompt}],
            "stream": True,
        }) as r:
            r.raise_for_status()
            async for line in r.aiter_lines():
                if line:
                    data = json.loads(line)
                    print(data["message"]["content"], end="", flush=True)
```

> Aquí `timeout=None` es aceptable solo porque es un servicio local de confianza con respuestas largas. En otros casos, usa `Timeout(connect=5, read=60, write=10, pool=5)`.

### Subir un archivo grande en streaming

```python
async def leer_en_chunks(path: str, size: int = 64 * 1024):
    with open(path, "rb") as f:
        while chunk := f.read(size):
            yield chunk

await client.post("/upload", content=leer_en_chunks("video.mp4"),
                  headers={"Content-Type": "application/octet-stream"})
```

---

## 14. Async y concurrencia

### Muchas peticiones en paralelo, con límite

```python
import asyncio
import httpx

async def obtener(client: httpx.AsyncClient, sem: asyncio.Semaphore, uid: int):
    async with sem:
        r = await client.get(f"/usuarios/{uid}")
        r.raise_for_status()
        return r.json()

async def main():
    sem = asyncio.Semaphore(10)  # máx. 10 peticiones simultáneas
    async with httpx.AsyncClient(base_url="https://api.ejemplo.com") as client:
        resultados = await asyncio.gather(
            *(obtener(client, sem, i) for i in range(1, 201)),
            return_exceptions=True,     # no aborta todo si una falla
        )
    ok = [r for r in resultados if not isinstance(r, Exception)]
    errores = [r for r in resultados if isinstance(r, Exception)]
    print(f"OK: {len(ok)}  Errores: {len(errores)}")

asyncio.run(main())
```

### Con `TaskGroup` (Python 3.11+)

```python
async with asyncio.TaskGroup() as tg:
    tareas = [tg.create_task(obtener(client, sem, i)) for i in range(1, 51)]
resultados = [t.result() for t in tareas]
```

> `TaskGroup` cancela las demás tareas si una falla. Usa `gather(return_exceptions=True)` si prefieres resultados parciales.

### Rate limiting simple

```python
import asyncio
import time

class RateLimiter:
    """Máximo `rate` peticiones por segundo."""
    def __init__(self, rate: float):
        self.intervalo = 1.0 / rate
        self._ultimo = 0.0
        self._lock = asyncio.Lock()

    async def __aenter__(self):
        async with self._lock:
            espera = self._ultimo + self.intervalo - time.monotonic()
            if espera > 0:
                await asyncio.sleep(espera)
            self._ultimo = time.monotonic()

    async def __aexit__(self, *exc):
        return False

limiter = RateLimiter(rate=5)
async with limiter:
    await client.get("/endpoint-limitado")
```

### ⚠️ Nunca uses `httpx.Client` (sync) dentro de código async

Bloquea el event loop. En FastAPI con `async def`, usa siempre `AsyncClient`.

---

## 15. HTTP/2

```bash
pip install "httpx[http2]"
```

```python
client = httpx.AsyncClient(http2=True)
r = await client.get("https://www.google.com")
print(r.http_version)  # "HTTP/2"
```

- Multiplexa muchas peticiones sobre **una sola conexión TCP**, lo que conviene cuando haces muchas peticiones concurrentes al mismo host.
- Si el servidor no lo soporta, httpx negocia HTTP/1.1 automáticamente (vía ALPN).
- No siempre es más rápido: haz benchmarks con tu caso real.

---

## 16. Proxies y transports

### Proxy

```python
client = httpx.Client(proxy="http://usuario:pass@proxy.empresa.com:8080")

# SOCKS (requiere httpx[socks])
client = httpx.Client(proxy="socks5://localhost:1080")
```

### Proxy por esquema o dominio con `mounts`

```python
client = httpx.Client(mounts={
    "http://": httpx.HTTPTransport(proxy="http://proxy:8080"),
    "https://": httpx.HTTPTransport(proxy="http://proxy:8080"),
    "all://*.interno.empresa.com": None,   # sin proxy para dominios internos
})
```

### Variables de entorno

Con `trust_env=True` (el default), httpx lee `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY`, `SSL_CERT_FILE`, `SSL_CERT_DIR` y `.netrc`.

### Transport personalizado (middleware de bajo nivel)

```python
import httpx

class HeaderInjectorTransport(httpx.AsyncBaseTransport):
    def __init__(self, inner: httpx.AsyncBaseTransport, headers: dict[str, str]):
        self.inner = inner
        self.headers = headers

    async def handle_async_request(self, request: httpx.Request) -> httpx.Response:
        request.headers.update(self.headers)
        return await self.inner.handle_async_request(request)

    async def aclose(self):
        await self.inner.aclose()

transport = HeaderInjectorTransport(
    httpx.AsyncHTTPTransport(retries=2, http2=True),
    {"X-App": "mi-app"},
)
client = httpx.AsyncClient(transport=transport)
```

### Llamar una app ASGI/WSGI en memoria (sin red)

```python
from fastapi import FastAPI
app = FastAPI()

client = httpx.AsyncClient(transport=httpx.ASGITransport(app=app), base_url="http://test")
```

### Unix Domain Socket (por ejemplo, Docker)

```python
transport = httpx.HTTPTransport(uds="/var/run/docker.sock")
with httpx.Client(transport=transport) as c:
    print(c.get("http://docker/containers/json").json())
```

---

## 17. Seguridad

### 17.1 TLS: nunca desactives la verificación

```python
# ❌ NUNCA en producción
httpx.Client(verify=False)

# ✅ CA corporativa o autofirmada: usa un SSLContext
import ssl
ctx = ssl.create_default_context(cafile="/ruta/ca-empresa.pem")
client = httpx.Client(verify=ctx)

# ✅ Usar el almacén de certificados del sistema operativo
# pip install truststore
import truststore
ctx = truststore.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
client = httpx.Client(verify=ctx)
```

Por defecto, httpx usa los certificados de `certifi`. Mantenlo actualizado (`pip install -U certifi`).

### 17.2 Timeouts siempre

Sin timeout, un servidor lento o malicioso puede dejar tus workers colgados indefinidamente (agotamiento de recursos). Ver [sección 7](#7-timeouts).

### 17.3 SSRF: cuando la URL viene del usuario

Si tu API recibe una URL del usuario (webhooks, "importar desde URL", previews), un atacante puede apuntar a `http://169.254.169.254` (metadata de la nube), `localhost` o tu red interna.

```python
import ipaddress
import socket
from urllib.parse import urlsplit
import httpx

class URLNoPermitida(Exception): ...

PUERTOS_PERMITIDOS = {80, 443}

def validar_url_publica(url: str) -> str:
    partes = urlsplit(url)
    if partes.scheme not in {"http", "https"}:
        raise URLNoPermitida("Esquema no permitido")
    if not partes.hostname:
        raise URLNoPermitida("Host inválido")
    puerto = partes.port or (443 if partes.scheme == "https" else 80)
    if puerto not in PUERTOS_PERMITIDOS:
        raise URLNoPermitida("Puerto no permitido")

    for info in socket.getaddrinfo(partes.hostname, puerto, proto=socket.IPPROTO_TCP):
        ip = ipaddress.ip_address(info[4][0])
        if (ip.is_private or ip.is_loopback or ip.is_link_local
                or ip.is_reserved or ip.is_multicast or ip.is_unspecified):
            raise URLNoPermitida(f"IP no permitida: {ip}")
    return url

async def fetch_externo(client: httpx.AsyncClient, url: str) -> bytes:
    validar_url_publica(url)
    # follow_redirects=False: una redirección podría apuntar a una IP interna
    r = await client.get(url, follow_redirects=False, timeout=10)
    r.raise_for_status()
    return r.content
```

> ⚠️ Esta validación no cubre el **DNS rebinding** (el DNS puede resolver distinto entre la validación y la conexión). Para máxima seguridad, sal por un **proxy de egreso** con allowlist o aplica reglas de firewall de red.

### 17.4 Limitar el tamaño de la respuesta

Evita que una respuesta gigante agote tu memoria:

```python
MAX_BYTES = 10 * 1024 * 1024  # 10 MB

class RespuestaDemasiadoGrande(Exception): ...

async def get_limitado(client: httpx.AsyncClient, url: str) -> bytes:
    async with client.stream("GET", url) as r:
        r.raise_for_status()
        cl = r.headers.get("Content-Length")
        if cl and int(cl) > MAX_BYTES:
            raise RespuestaDemasiadoGrande(cl)
        buffer = bytearray()
        async for chunk in r.aiter_bytes():
            buffer.extend(chunk)
            if len(buffer) > MAX_BYTES:
                raise RespuestaDemasiadoGrande(len(buffer))
        return bytes(buffer)
```

### 17.5 No filtres secretos

```python
# ❌ El token queda en logs de proxies, servidores e historial
client.get(f"/datos?api_key={API_KEY}")

# ✅ En headers
client.get("/datos", headers={"X-API-Key": API_KEY})
```

Redacta headers sensibles en tus logs:

```python
SENSIBLES = {"authorization", "x-api-key", "cookie", "set-cookie", "proxy-authorization"}

def headers_seguros(headers: httpx.Headers) -> dict:
    return {k: ("***" if k.lower() in SENSIBLES else v) for k, v in headers.items()}

async def log_request(request: httpx.Request):
    log.debug("→ %s %s %s", request.method, request.url.copy_with(query=None),
              headers_seguros(request.headers))
```

- Carga los secretos desde variables de entorno o un gestor de secretos (Vault, AWS Secrets Manager), **nunca** hardcodeados.
- Las URLs con credenciales (`https://user:pass@host`) también terminan en logs: evítalas.

### 17.6 Redirecciones

- `follow_redirects=False` es el default. Actívalo solo si lo necesitas.
- Al seguir una redirección a **otro origen**, httpx elimina el header `Authorization`. Aun así, no confíes en redirecciones hacia dominios externos.
- Limita los saltos con `max_redirects` (default 20).

### 17.7 `trust_env` en entornos controlados

En contenedores o servicios donde las variables de entorno podrían manipularse, desactívalo para que nadie pueda inyectar un proxy:

```python
client = httpx.AsyncClient(trust_env=False)
```

### 17.8 Valida lo que recibes

Nunca confíes en el JSON externo: valídalo con Pydantic antes de usarlo ([sección 6](#6-respuestas)).

### 17.9 Dependencias

```bash
pip install pip-audit
pip-audit                 # busca CVEs en httpx, httpcore, h11, certifi…
```

Fija versiones y actualiza `httpcore`, `h11` y `certifi` con regularidad.

---

## 18. Integración con FastAPI

### Un solo `AsyncClient` para toda la app (ciclo de vida con `lifespan`)

```python
# app/main.py
from contextlib import asynccontextmanager
from typing import Annotated

import httpx
from fastapi import FastAPI, Depends, Request, HTTPException

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = httpx.AsyncClient(
        base_url="http://localhost:8080/api",
        timeout=httpx.Timeout(10.0, connect=3.0),
        limits=httpx.Limits(max_connections=100, max_keepalive_connections=20),
        headers={"User-Agent": "fastapi-gateway/1.0"},
    )
    yield
    await app.state.http.aclose()

app = FastAPI(lifespan=lifespan)

def get_http(request: Request) -> httpx.AsyncClient:
    return request.app.state.http

HttpDep = Annotated[httpx.AsyncClient, Depends(get_http)]

@app.get("/usuarios/{uid}")
async def usuario(uid: int, http: HttpDep):
    try:
        r = await http.get(f"/usuarios/{uid}")
        r.raise_for_status()
    except httpx.HTTPStatusError as e:
        raise HTTPException(status_code=e.response.status_code, detail="Error en servicio de usuarios")
    except httpx.RequestError:
        raise HTTPException(status_code=503, detail="Servicio de usuarios no disponible")
    return r.json()
```

### Proxy en streaming (sin cargar la respuesta en memoria)

```python
from fastapi.responses import StreamingResponse
from starlette.background import BackgroundTask

@app.get("/archivos/{nombre}")
async def proxy_archivo(nombre: str, http: HttpDep):
    req = http.build_request("GET", f"/archivos/{nombre}")
    r = await http.send(req, stream=True)
    if r.is_error:
        await r.aclose()
        raise HTTPException(status_code=r.status_code)
    return StreamingResponse(
        r.aiter_raw(),
        status_code=r.status_code,
        media_type=r.headers.get("content-type"),
        background=BackgroundTask(r.aclose),   # cierra la respuesta al terminar
    )
```

---

## 19. Patrón: cliente tipado tipo Feign

Una clase por servicio externo, con modelos Pydantic, errores de dominio y reintentos. Es lo más parecido a un `@FeignClient` y no depende de librerías abandonadas.

```python
# app/clients/usuarios.py
from __future__ import annotations

import httpx
from pydantic import BaseModel, TypeAdapter
from tenacity import retry, stop_after_attempt, wait_exponential_jitter, retry_if_exception

# ---------- DTOs ----------
class Usuario(BaseModel):
    id: int
    nombre: str
    email: str

class CrearUsuario(BaseModel):
    nombre: str
    email: str

_ListaUsuarios = TypeAdapter(list[Usuario])

# ---------- Errores de dominio ----------
class UsuariosClientError(Exception): ...
class UsuarioNoEncontrado(UsuariosClientError): ...
class UsuariosNoDisponible(UsuariosClientError): ...

def _reintentable(exc: BaseException) -> bool:
    if isinstance(exc, (httpx.ConnectError, httpx.ConnectTimeout, httpx.ReadTimeout)):
        return True
    return isinstance(exc, httpx.HTTPStatusError) and exc.response.status_code in {502, 503, 504}

_retry = retry(
    retry=retry_if_exception(_reintentable),
    stop=stop_after_attempt(3),
    wait=wait_exponential_jitter(initial=0.3, max=3),
    reraise=True,
)

# ---------- Cliente ----------
class UsuariosClient:
    def __init__(self, http: httpx.AsyncClient):
        self._http = http

    @classmethod
    def create(cls, base_url: str, token: str) -> "UsuariosClient":
        return cls(httpx.AsyncClient(
            base_url=base_url,
            headers={"Authorization": f"Bearer {token}"},
            timeout=httpx.Timeout(10.0, connect=3.0),
        ))

    async def aclose(self) -> None:
        await self._http.aclose()

    async def __aenter__(self) -> "UsuariosClient":
        return self

    async def __aexit__(self, *exc) -> None:
        await self.aclose()

    # --- Endpoints ---
    @_retry
    async def obtener(self, user_id: int) -> Usuario:
        r = await self._http.get(f"/usuarios/{user_id}")
        self._check(r, recurso=f"usuario {user_id}")
        return Usuario.model_validate_json(r.content)

    @_retry
    async def buscar(self, nombre: str | None = None, page: int = 0, size: int = 20) -> list[Usuario]:
        params = {"page": page, "size": size}
        if nombre:
            params["nombre"] = nombre
        r = await self._http.get("/usuarios", params=params)
        self._check(r)
        return _ListaUsuarios.validate_json(r.content)

    async def crear(self, data: CrearUsuario) -> Usuario:    # POST: sin reintento automático
        r = await self._http.post("/usuarios", json=data.model_dump())
        self._check(r)
        return Usuario.model_validate_json(r.content)

    async def eliminar(self, user_id: int) -> None:
        r = await self._http.delete(f"/usuarios/{user_id}")
        self._check(r, recurso=f"usuario {user_id}")

    # --- Interno ---
    @staticmethod
    def _check(r: httpx.Response, recurso: str = "recurso") -> None:
        if r.status_code == 404:
            raise UsuarioNoEncontrado(f"No existe {recurso}")
        if r.status_code >= 500:
            # se relanza como HTTPStatusError para que tenacity decida si reintenta
            r.raise_for_status()
        r.raise_for_status()
```

Uso:

```python
async with UsuariosClient.create("http://localhost:8080/api", token="...") as usuarios:
    u = await usuarios.obtener(1)
    nuevos = await usuarios.buscar(nombre="Gil")
```

### Alternativa: generar el cliente desde OpenAPI

Si el servicio destino publica OpenAPI (FastAPI, o Spring Boot con springdoc):

```bash
pip install openapi-python-client
openapi-python-client generate --url http://localhost:8080/v3/api-docs
```

Genera un paquete con modelos y funciones tipadas que usa httpx por debajo.

---

## 20. Testing

### 20.1 `MockTransport` (nativo, sin dependencias)

```python
# tests/test_usuarios_client.py
import httpx
import pytest
from app.clients.usuarios import UsuariosClient, UsuarioNoEncontrado

def handler(request: httpx.Request) -> httpx.Response:
    if request.url.path == "/usuarios/1":
        return httpx.Response(200, json={"id": 1, "nombre": "Gil", "email": "gil@ejemplo.com"})
    if request.url.path == "/usuarios/99":
        return httpx.Response(404, json={"detail": "not found"})
    return httpx.Response(500)

@pytest.fixture
async def client():
    http = httpx.AsyncClient(transport=httpx.MockTransport(handler), base_url="http://test")
    yield UsuariosClient(http)
    await http.aclose()

async def test_obtener_ok(client):
    u = await client.obtener(1)
    assert u.nombre == "Gil"

async def test_obtener_404(client):
    with pytest.raises(UsuarioNoEncontrado):
        await client.obtener(99)
```

```bash
pip install pytest pytest-asyncio
# pytest.ini / pyproject: asyncio_mode = "auto"
```

### 20.2 `respx` (estilo WireMock, más expresivo)

```bash
pip install respx
```

```python
import httpx
import respx
import pytest

@respx.mock
async def test_reintenta_en_503():
    ruta = respx.get("http://test/usuarios/1").mock(side_effect=[
        httpx.Response(503),
        httpx.Response(200, json={"id": 1, "nombre": "Gil", "email": "g@e.com"}),
    ])
    async with httpx.AsyncClient(base_url="http://test") as http:
        from app.clients.usuarios import UsuariosClient
        u = await UsuariosClient(http).obtener(1)

    assert u.id == 1
    assert ruta.call_count == 2          # verifica que reintentó

@respx.mock
async def test_timeout():
    respx.get("http://test/lento").mock(side_effect=httpx.ReadTimeout("timeout"))
    async with httpx.AsyncClient(base_url="http://test") as http:
        with pytest.raises(httpx.ReadTimeout):
            await http.get("/lento")
```

### 20.3 Probar tu propia app FastAPI

```python
import httpx
import pytest
from asgi_lifespan import LifespanManager   # pip install asgi-lifespan
from app.main import app

@pytest.fixture
async def api():
    # ASGITransport NO ejecuta el lifespan; LifespanManager sí
    async with LifespanManager(app):
        transport = httpx.ASGITransport(app=app)
        async with httpx.AsyncClient(transport=transport, base_url="http://test") as c:
            yield c

async def test_salud(api):
    r = await api.get("/salud")
    assert r.status_code == 200
```

---

## 21. Logging y observabilidad

### Logs internos de httpx

```python
import logging

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(name)s %(levelname)s %(message)s")
logging.getLogger("httpx").setLevel(logging.INFO)      # una línea por petición
logging.getLogger("httpcore").setLevel(logging.DEBUG)  # detalle de conexiones/TLS (solo para depurar)
```

> En producción, `httpcore` en DEBUG es muy ruidoso y puede exponer información sensible.

### OpenTelemetry (trazas distribuidas)

```bash
pip install opentelemetry-instrumentation-httpx
```

```python
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

HTTPXClientInstrumentor().instrument()   # instrumenta todos los clientes
```

Propaga automáticamente `traceparent`, de modo que tus servicios Spring Boot (con Micrometer Tracing) verán la misma traza.

---

## 22. Migrar desde `requests`

| `requests` | `httpx` | Nota |
|---|---|---|
| `requests.Session()` | `httpx.Client()` | |
| `allow_redirects=True` (default) | `follow_redirects=False` (default) | ⚠️ cambia el default |
| sin timeout por defecto | 5 s por defecto | |
| `r.ok` | `r.is_success` | |
| `r.iter_content()` | `r.iter_bytes()` | |
| `stream=True` en la petición | `with client.stream(...)` | |
| `session.mount(prefix, adapter)` | `mounts={...}` | |
| `proxies={"https": ...}` | `proxy=...` o `mounts` | |
| `HTTPAdapter(max_retries=...)` | `HTTPTransport(retries=...)` + tenacity | |
| `requests.exceptions.RequestException` | `httpx.HTTPError` | |
| `files=` / `data=` / `json=` | igual | `data=` con bytes → usa `content=` |

---

## 23. Cambios importantes en 0.28 y httpx 1.0

### 0.28 (versión actual estable)

- Se eliminó el argumento `proxies=`. Usa `proxy=` o `mounts=`.
- Se eliminó el atajo `app=`. Usa `transport=httpx.ASGITransport(app=app)` o `WSGITransport`.
- `verify` como ruta en string y el argumento `cert` quedaron **deprecados**. Usa un `ssl.SSLContext`.
- El JSON del body ahora se envía compacto (sin espacios).

### httpx 1.0

En PyPI ya existen pre-releases `1.0.devN`. Pueden traer cambios incompatibles: **no las uses en producción** hasta que salga la versión estable, y revisa el changelog oficial antes de migrar.

---

## 24. Errores comunes (anti-patrones)

| ❌ Anti-patrón | Problema | ✅ Solución |
|---|---|---|
| `httpx.get()` en un bucle | Abre una conexión nueva por llamada | Reutiliza un `Client` |
| Crear un `AsyncClient` por request en FastAPI | Sin pool y sin keep-alive | Un cliente en `lifespan` |
| No cerrar el cliente | Fuga de sockets | `async with` o `aclose()` |
| `httpx.Client` dentro de `async def` | Bloquea el event loop | `AsyncClient` |
| `verify=False` | Vulnerable a MITM | `SSLContext` con tu CA |
| `timeout=None` | Workers colgados | `httpx.Timeout(...)` |
| Olvidar `raise_for_status()` | Errores 4xx/5xx silenciosos | Llámalo siempre o usa un hook |
| Reintentar un `POST` sin idempotencia | Cobros o registros duplicados | `Idempotency-Key` |
| `gather()` sin límite sobre 10 000 URLs | Satura el pool y al servidor | `Semaphore` |
| Leer `.json()` en un hook sin `aread()` | `ResponseNotRead` | `await response.aread()` |
| Usar una URL del usuario sin validar | SSRF | [sección 17.3](#173-ssrf-cuando-la-url-viene-del-usuario) |
| Tokens en el query string | Filtración en logs | Headers |

---

## 25. Checklist de producción

- [ ] Versión fijada: `httpx>=0.28,<0.29`
- [ ] Un `AsyncClient` reutilizable por servicio, creado en `lifespan` y cerrado al apagar
- [ ] `base_url` y headers comunes configurados en el cliente
- [ ] `Timeout` explícito (connect, read, write, pool)
- [ ] `Limits` dimensionados para la concurrencia esperada
- [ ] `raise_for_status()` y mapeo a errores de dominio
- [ ] Reintentos con backoff y jitter, solo en errores transitorios y métodos idempotentes
- [ ] Respeto de `Retry-After` en 429/503
- [ ] TLS verificado (`verify=True` o `SSLContext`), nunca `verify=False`
- [ ] Secretos en headers, cargados desde env o un vault, redactados en logs
- [ ] Protección SSRF si alguna URL viene del usuario
- [ ] Límite de tamaño de respuesta para contenido externo
- [ ] Respuestas validadas con Pydantic
- [ ] Tests con `MockTransport` o `respx` (éxito, 404, 5xx, timeout, reintento)
- [ ] Logging con correlación (trace-id) u OpenTelemetry
- [ ] `pip-audit` en el CI

---

## Referencias

- Documentación oficial: <https://www.python-httpx.org/>
- Changelog: <https://github.com/encode/httpx/blob/master/CHANGELOG.md>
- PyPI: <https://pypi.org/project/httpx/>
- respx: <https://lundberg.github.io/respx/>
- tenacity: <https://tenacity.readthedocs.io/>
- OWASP, prevención de SSRF: <https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html>
