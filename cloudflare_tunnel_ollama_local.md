# Conectar una API en la nube con Ollama local usando Cloudflare Tunnel

## Objetivo

Permitir que una API alojada en la nube (por ejemplo, FastAPI) envíe
solicitudes a un Ollama instalado en tu PC local, sin abrir puertos
entrantes en el router.

**Arquitectura recomendada:**

``` text
API en la nube
    |
    | HTTPS + Cloudflare Access Service Token
    v
https://ollama-api.tudominio.com
    |
Cloudflare Tunnel (conexión iniciada desde tu PC)
    |
    v
Gateway FastAPI local (127.0.0.1:8001)
    |
    | solo tráfico local
    v
Ollama (127.0.0.1:11434)
```

> **Importante:** no recomiendo publicar directamente el puerto `11434`
> de Ollama sin una capa de autenticación. Ollama normalmente no
> proporciona autenticación de usuario para su API local. Este
> procedimiento publica un pequeño gateway FastAPI, protegido por
> Cloudflare Access y por una API key adicional, que solo reenvía
> solicitudes a Ollama.

## 1. Requisitos

-   Una cuenta de Cloudflare.
-   Un dominio administrado por Cloudflare, por ejemplo `tudominio.com`.
-   Ollama funcionando en tu PC local.
-   Python 3.10 o posterior en el PC local.
-   Una API en la nube capaz de realizar peticiones HTTPS salientes.
-   Conexión a Internet activa en el PC local.

Cloudflare Tunnel inicia conexiones salientes desde tu PC; no necesitas
abrir puertos en el router. Consulta la [documentación oficial de
Cloudflare Tunnel](https://developers.cloudflare.com/tunnel/).

En los ejemplos se utiliza `ollama-api.tudominio.com`. Sustituye ese
dominio por uno que controles.

## 2. Comprobar Ollama en el PC local

En PowerShell, ejecuta:

``` powershell
ollama --version
ollama list
```

Comprueba que la API local responde:

``` powershell
Invoke-RestMethod -Method Get -Uri "http://127.0.0.1:11434/api/tags"
```

Deberías obtener un JSON con los modelos instalados. Si no responde,
inicia Ollama y vuelve a probar.

**No configures el router para reenviar el puerto 11434 ni cambies
Ollama para escuchar públicamente.** Mantén su acceso en `127.0.0.1`.

## 3. Crear un gateway local con FastAPI

El gateway recibirá la petición de la API en la nube, validará una API
key y reenviará una solicitud controlada a Ollama.

Crea una carpeta, por ejemplo `C:\ollama-gateway`, y dentro crea un
entorno virtual:

``` powershell
mkdir C:\ollama-gateway
cd C:\ollama-gateway
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install fastapi uvicorn httpx
```

Genera una clave aleatoria para el gateway. Guarda el resultado en un
gestor de secretos; no la publiques en el código ni en Git:

``` powershell
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

Crea `app.py`:

``` python
import os
import secrets
from typing import Literal

import httpx
from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel, Field

app = FastAPI(title="Ollama Gateway")

# Configurar OLLAMA_GATEWAY_API_KEY como variable de entorno.
GATEWAY_API_KEY = os.environ["OLLAMA_GATEWAY_API_KEY"]

# El modelo se limita a una lista permitida para evitar que el cliente
# solicite modelos arbitrarios.
ALLOWED_MODELS = {
    "qwen3:4b",
    "llama3.2:3b",
}

OLLAMA_URL = "http://127.0.0.1:11434/api/chat"


class ChatMessage(BaseModel):
    role: Literal["system", "user", "assistant"]
    content: str = Field(min_length=1, max_length=12000)


class ChatRequest(BaseModel):
    model: str
    messages: list[ChatMessage] = Field(min_length=1, max_length=40)
    stream: bool = False


@app.get("/health")
async def health():
    # No devuelve información de configuración ni claves.
    return {"status": "ok"}


@app.post("/api/chat")
async def chat(
    request: ChatRequest,
    x_api_key: str = Header(default=""),
):
    if not secrets.compare_digest(x_api_key, GATEWAY_API_KEY):
        raise HTTPException(status_code=401, detail="No autorizado")

    if request.model not in ALLOWED_MODELS:
        raise HTTPException(status_code=400, detail="Modelo no permitido")

    # Deshabilitamos streaming en este ejemplo para simplificar el manejo
    # de la respuesta. Se pueden añadir límites de concurrencia y uso.
    if request.stream:
        raise HTTPException(
            status_code=400,
            detail="Este endpoint no admite stream=true",
        )

    payload = {
        "model": request.model,
        "messages": [m.model_dump() for m in request.messages],
        "stream": False,
    }

    try:
        async with httpx.AsyncClient(timeout=httpx.Timeout(300.0)) as client:
            response = await client.post(OLLAMA_URL, json=payload)
            response.raise_for_status()
            return response.json()
    except httpx.TimeoutException:
        raise HTTPException(status_code=504, detail="Ollama agotó el tiempo de espera")
    except httpx.HTTPStatusError:
        raise HTTPException(status_code=502, detail="Ollama devolvió un error")
    except httpx.RequestError:
        raise HTTPException(status_code=502, detail="No se pudo conectar con Ollama")
```

### 3.1. Configurar variables de entorno

En PowerShell, define la misma clave que generaste en el paso anterior:

``` powershell
$env:OLLAMA_GATEWAY_API_KEY = "PEGA_AQUI_TU_CLAVE_ALEATORIA"
```

Ajusta `ALLOWED_MODELS` en `app.py` para que coincida exactamente con
los nombres mostrados por `ollama list`. No uses una clave real en
ejemplos, capturas de pantalla ni repositorios.

### 3.2. Iniciar el gateway

``` powershell
uvicorn app:app --host 127.0.0.1 --port 8001
```

Deja esta terminal abierta. El gateway solo escucha en loopback; no
queda expuesto directamente a la red local.

En otra ventana de PowerShell, prueba el estado:

``` powershell
Invoke-RestMethod -Uri "http://127.0.0.1:8001/health"
```

Prueba una generación (reemplaza la clave y el nombre del modelo si es
necesario):

``` powershell
$headers = @{
  "X-API-Key" = "PEGA_AQUI_TU_CLAVE_ALEATORIA"
}

$body = @{
  model = "qwen3:4b"
  messages = @(
    @{ role = "user"; content = "Responde brevemente: ¿qué eres?" }
  )
  stream = $false
} | ConvertTo-Json -Depth 8

Invoke-RestMethod `
  -Method Post `
  -Uri "http://127.0.0.1:8001/api/chat" `
  -Headers $headers `
  -ContentType "application/json" `
  -Body $body
```

Si tu modelo tiene otro nombre, utiliza el nombre exacto de
`ollama list`.

## 4. Crear el túnel en Cloudflare

1.  Entra en el [panel de Cloudflare](https://dash.cloudflare.com/).
2.  Selecciona tu cuenta.
3.  Abre **Networking → Tunnels** (el nombre del menú puede variar).
4.  Selecciona **Create Tunnel** y elige `cloudflared`.
5.  Ponle un nombre, por ejemplo `ollama-local`.
6.  Selecciona Windows y copia el comando de instalación que muestra
    Cloudflare.
7.  Ejecuta el comando en PowerShell como administrador, siguiendo las
    instrucciones del panel.
8.  Cuando el conector aparezca como conectado o `Healthy`, configura la
    ruta pública.

Añade una ruta de aplicación publicada:

-   **Hostname:** `ollama-api.tudominio.com`
-   **Service / URL de destino:** `http://127.0.0.1:8001`

No uses `http://127.0.0.1:11434`: el túnel debe llegar al gateway, no
directamente a Ollama.

Cloudflare normalmente crea el registro DNS al configurar la ruta desde
el panel. Sigue el flujo de la documentación oficial si la interfaz de
tu cuenta difiere: [Configurar Cloudflare
Tunnel](https://developers.cloudflare.com/tunnel/get-started/).

### 4.1. Instalar el túnel como servicio

Utiliza el comando exacto que Cloudflare muestra para tu túnel. Para un
túnel administrado remotamente, normalmente tiene la forma:

``` powershell
cloudflared.exe service install TU_TOKEN_DEL_TUNEL
```

El token del túnel es secreto: no lo publiques ni lo incluyas en Git.
Cualquier persona que lo obtenga podría ejecutar un conector para ese
túnel. Si se filtra, rótalo desde Cloudflare. Consulta [Tunnel
tokens](https://developers.cloudflare.com/tunnel/reference/tunnel-tokens/).

## 5. Proteger el hostname con Cloudflare Access

No dejes el hostname público sin una política de acceso.

1.  En el panel de Cloudflare, abre la sección **Zero Trust / Access**.
2.  Crea una aplicación de tipo **Self-hosted** para
    `ollama-api.tudominio.com`.
3.  Crea una política que permita autenticación mediante **Service
    Auth** / **Service Token**.
4.  Genera un Service Token para tu API alojada en la nube.
5.  Guarda el **Client ID** y el **Client Secret** en el gestor de
    secretos de la nube.

Los nombres exactos de las pantallas pueden cambiar. Sigue la
[documentación de Cloudflare
Access](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/self-hosted-apps/).

La API en la nube tendrá que enviar estas cabeceras en cada solicitud:

-   `CF-Access-Client-Id`
-   `CF-Access-Client-Secret`

Cloudflare Access valida esas credenciales antes de permitir que la
solicitud llegue al gateway. El gateway valida además `X-API-Key`.

**No coloques el Service Token ni la API key en el código del frontend,
en Angular, ni en una aplicación distribuida a usuarios.** Deben
permanecer en el backend de la nube.

## 6. Llamar a Ollama desde la API alojada en la nube

Desde el backend de la nube, realiza una petición al hostname público.
Ejemplo con Python y `httpx`:

``` python
import os
import httpx

OLLAMA_GATEWAY_URL = "https://ollama-api.tudominio.com/api/chat"

headers = {
    "CF-Access-Client-Id": os.environ["CF_ACCESS_CLIENT_ID"],
    "CF-Access-Client-Secret": os.environ["CF_ACCESS_CLIENT_SECRET"],
    "X-API-Key": os.environ["OLLAMA_GATEWAY_API_KEY"],
}

payload = {
    "model": "qwen3:4b",
    "messages": [
        {"role": "user", "content": "Responde brevemente: ¿qué eres?"}
    ],
    "stream": False,
}

async def consultar_ollama():
    async with httpx.AsyncClient(timeout=300.0) as client:
        response = await client.post(
            OLLAMA_GATEWAY_URL,
            headers=headers,
            json=payload,
        )
        response.raise_for_status()
        return response.json()
```

Instala `httpx` en el entorno de la API en la nube si todavía no está
instalado. Configura las tres variables de entorno en el sistema de
secretos de tu plataforma. No las subas al repositorio.

La respuesta de Ollama incluye un objeto `message`; por ejemplo, el
texto generado normalmente estará en `respuesta["message"]["content"]`.

## 7. Probar la conexión de extremo a extremo

Haz las pruebas en este orden:

1.  **Ollama local:** `http://127.0.0.1:11434/api/tags`.
2.  **Gateway local:** `http://127.0.0.1:8001/health`.
3.  **Gateway autenticado local:** petición `POST /api/chat` con
    `X-API-Key`.
4.  **Túnel:** el panel de Cloudflare debe indicar que el conector está
    conectado.
5.  **Access:** prueba el hostname sin credenciales y confirma que no
    permite el acceso.
6.  **API en la nube:** realiza una petición con ambos Service Token
    headers y `X-API-Key`.

Un estado `Healthy` del túnel solo confirma que el conector está
conectado a Cloudflare; no garantiza que la aplicación local responda
correctamente. Comprueba también el gateway y los registros. Consulta
[solución de problemas de Cloudflare
Tunnel](https://developers.cloudflare.com/tunnel/troubleshooting/).

## 8. Recomendaciones para producción

-   Mantén Ollama en `127.0.0.1:11434` y el gateway en `127.0.0.1:8001`.
-   No abras puertos en el router ni publiques el puerto 11434.
-   Conserva Cloudflare Access y la API key del gateway: son dos
    controles diferentes.
-   Usa un gestor de secretos en la nube y variables de entorno en el PC
    local.
-   Limita los modelos permitidos y valida el tamaño del cuerpo, los
    roles y la cantidad de mensajes.
-   Añade límites de concurrencia, cuotas por cliente, métricas y
    registros sin guardar prompts sensibles.
-   Configura límites de tiempo adecuados: la inferencia local puede
    tardar bastante, especialmente con modelos grandes.
-   Si el PC se apaga, pierde Internet o suspende el sistema, la API en
    la nube no podrá usar el modelo.
-   Para varios clientes, añade autenticación por cliente y aislamiento
    de datos en tu propia API.
-   Considera que el tráfico de prompts y respuestas pasa por Cloudflare
    y por la infraestructura de tu proveedor de nube. Revisa las
    políticas de privacidad que correspondan a tus datos.
-   El gateway de ejemplo no implementa streaming, cuotas, gestión de
    usuarios ni límites avanzados de concurrencia; añádelos antes de
    ofrecer el servicio comercialmente.

## 9. Diagrama final

``` text
┌───────────────────────────────┐
│ Backend / FastAPI en la nube  │
│ Secretos: Access + API key    │
└───────────────┬───────────────┘
                │ HTTPS
                ▼
┌───────────────────────────────┐
│ Cloudflare Access             │
│ valida Service Token          │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│ Cloudflare Tunnel             │
│ conexión saliente desde local │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│ Gateway FastAPI local         │
│ valida X-API-Key              │
│ permite solo modelos definidos│
└───────────────┬───────────────┘
                │ loopback
                ▼
┌───────────────────────────────┐
│ Ollama: 127.0.0.1:11434       │
│ modelo local                  │
└───────────────────────────────┘
```

## Documentación oficial

-   [Cloudflare Tunnel:
    introducción](https://developers.cloudflare.com/tunnel/)
-   [Crear y publicar un
    túnel](https://developers.cloudflare.com/tunnel/get-started/)
-   [Cloudflare Access: aplicaciones
    self-hosted](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/self-hosted-apps/)
-   [Tokens de Cloudflare
    Tunnel](https://developers.cloudflare.com/tunnel/reference/tunnel-tokens/)
-   [API de
    Ollama](https://github.com/ollama/ollama/blob/main/docs/api.md)
-   [FastAPI](https://fastapi.tiangolo.com/)
