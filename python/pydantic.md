# Pydantic + FastAPI: de principiante a pro

Guía práctica con ejemplos listos para copiar. Todo está escrito para **Pydantic v2** y versiones modernas de **FastAPI**.

## Índice

1. [Instalación y entorno](#1-instalación-y-entorno)
2. [Nivel 1 — Principiante: fundamentos de Pydantic](#2-nivel-1--principiante-fundamentos-de-pydantic)
3. [Nivel 1 — Principiante: tu primera API con FastAPI](#3-nivel-1--principiante-tu-primera-api-con-fastapi)
4. [Nivel 2 — Intermedio: validación y modelado serio](#4-nivel-2--intermedio-validación-y-modelado-serio)
5. [Nivel 2 — Intermedio: patrón Create / Update / Response](#5-nivel-2--intermedio-patrón-create--update--response)
6. [Nivel 3 — Avanzado: Pydantic a fondo](#6-nivel-3--avanzado-pydantic-a-fondo)
7. [Nivel 3 — Avanzado: FastAPI en serio](#7-nivel-3--avanzado-fastapi-en-serio)
8. [Nivel 4 — Pro: arquitectura, testing y rendimiento](#8-nivel-4--pro-arquitectura-testing-y-rendimiento)
9. [Referencia: Pydantic vs Spring Boot](#9-referencia-pydantic-vs-spring-boot)
10. [Referencia: migración v1 → v2](#10-referencia-migración-v1--v2)
11. [Cheatsheet](#11-cheatsheet)

---

## 1. Instalación y entorno

```bash
# Crear entorno virtual
python -m venv .venv
source .venv/bin/activate        # Linux/Mac
.venv\Scripts\activate           # Windows

# FastAPI con extras estándar (incluye uvicorn, CLI "fastapi", etc.)
pip install "fastapi[standard]"

# Extras útiles
pip install "pydantic[email]"    # EmailStr
pip install pydantic-settings    # configuración desde .env
pip install sqlalchemy           # ORM (nivel 3)
pip install pytest httpx         # testing (nivel 4)
```

Ejecutar la app:

```bash
fastapi dev main.py      # desarrollo, con autoreload
fastapi run main.py      # producción
```

Documentación automática disponible en:

| URL | Qué es |
|---|---|
| `http://127.0.0.1:8000/docs` | Swagger UI interactivo |
| `http://127.0.0.1:8000/redoc` | ReDoc |
| `http://127.0.0.1:8000/openapi.json` | Esquema OpenAPI crudo |

---

## 2. Nivel 1 — Principiante: fundamentos de Pydantic

### 2.1 ¿Qué es Pydantic?

Pydantic es una librería de **validación de datos basada en type hints**. Defines una clase con tipos, y Pydantic:

- valida que los datos cumplan los tipos,
- convierte (coerce) valores compatibles (`"42"` → `42`),
- lanza errores claros cuando algo no cuadra,
- serializa a dict/JSON.

FastAPI usa Pydantic para validar requests, serializar responses y generar la documentación OpenAPI.

### 2.2 Tu primer modelo

```python
from pydantic import BaseModel


class Usuario(BaseModel):
    id: int
    nombre: str
    activo: bool = True          # valor por defecto
    edad: int | None = None      # opcional


u = Usuario(id="1", nombre="Gil")
print(u)              # id=1 nombre='Gil' activo=True edad=None
print(u.id + 1)       # 2  -> "1" se convirtió a int
```

### 2.3 Errores de validación

```python
from pydantic import BaseModel, ValidationError


class Producto(BaseModel):
    nombre: str
    precio: float


try:
    Producto(nombre="Laptop", precio="caro")
except ValidationError as e:
    print(e.error_count())   # 1
    print(e.errors())
    # [{'type': 'float_parsing', 'loc': ('precio',),
    #   'msg': 'Input should be a valid number, unable to parse string as a number', ...}]
```

### 2.4 Tipos más comunes

| Tipo | Ejemplo | Notas |
|---|---|---|
| `int`, `float`, `str`, `bool` | `edad: int` | Coerción en modo lax (`"5"` → `5`) |
| `X \| None` | `bio: str \| None = None` | Opcional; necesita default para no ser requerido |
| `list[T]` | `tags: list[str] = []` | Pydantic copia los defaults mutables, es seguro |
| `dict[K, V]` | `meta: dict[str, int]` | |
| `tuple[int, int]` | `coords: tuple[float, float]` | |
| `set[T]` | `roles: set[str]` | |
| `datetime`, `date`, `time` | `creado: datetime` | Acepta ISO 8601 |
| `UUID` | `id: UUID` | `from uuid import UUID` |
| `Decimal` | `monto: Decimal` | Para dinero |
| `Literal[...]` | `rol: Literal["admin", "user"]` | Valores fijos |
| `Enum` | `estado: Estado` | Ver nivel 2 |
| `EmailStr` | `email: EmailStr` | Requiere `pydantic[email]` |
| `HttpUrl` | `web: HttpUrl` | |

> **Importante:** `edad: int | None` (sin `= None`) sigue siendo **requerido**, solo que acepta `None`. Para que sea opcional necesitas el default.

### 2.5 Convertir de y hacia dict/JSON

```python
from pydantic import BaseModel


class Usuario(BaseModel):
    id: int
    nombre: str
    email: str | None = None


# Crear desde dict
u = Usuario.model_validate({"id": 1, "nombre": "Gil"})

# Crear desde JSON (string o bytes) - más rápido que json.loads + validate
u2 = Usuario.model_validate_json('{"id": 2, "nombre": "Ana"}')

# A dict
print(u.model_dump())                     # {'id': 1, 'nombre': 'Gil', 'email': None}
print(u.model_dump(exclude_none=True))    # {'id': 1, 'nombre': 'Gil'}
print(u.model_dump(include={"nombre"}))   # {'nombre': 'Gil'}

# A JSON
print(u.model_dump_json(indent=2))

# Copia con cambios (no muta el original)
u3 = u.model_copy(update={"nombre": "Gilberto"})
```

### 2.6 Modelos anidados

```python
from pydantic import BaseModel


class Direccion(BaseModel):
    calle: str
    ciudad: str
    cp: str


class Cliente(BaseModel):
    nombre: str
    direccion: Direccion
    telefonos: list[str] = []


c = Cliente.model_validate({
    "nombre": "Gil",
    "direccion": {"calle": "Av. 20 de Noviembre", "ciudad": "Durango", "cp": "34000"},
    "telefonos": ["6181234567"],
})
print(c.direccion.ciudad)  # Durango
```

---

## 3. Nivel 1 — Principiante: tu primera API con FastAPI

### 3.1 Hola mundo

```python
# main.py
from fastapi import FastAPI

app = FastAPI(title="Mi API", version="1.0.0")


@app.get("/")
def raiz():
    return {"mensaje": "Hola mundo"}
```

### 3.2 Path params y query params

```python
from fastapi import FastAPI

app = FastAPI()


# Path param: /items/5
@app.get("/items/{item_id}")
def obtener_item(item_id: int):
    return {"item_id": item_id}


# Query params: /items?skip=0&limit=10&q=laptop
@app.get("/items")
def listar_items(skip: int = 0, limit: int = 10, q: str | None = None):
    return {"skip": skip, "limit": limit, "q": q}
```

Si mandas `/items/abc`, FastAPI responde **422** automáticamente porque `abc` no es `int`.

### 3.3 Request body con Pydantic

```python
from fastapi import FastAPI, status
from pydantic import BaseModel

app = FastAPI()


class ProductoIn(BaseModel):
    nombre: str
    precio: float
    stock: int = 0


class ProductoOut(BaseModel):
    id: int
    nombre: str
    precio: float
    stock: int


db: dict[int, ProductoOut] = {}


@app.post("/productos", response_model=ProductoOut, status_code=status.HTTP_201_CREATED)
def crear_producto(data: ProductoIn):
    nuevo_id = len(db) + 1
    producto = ProductoOut(id=nuevo_id, **data.model_dump())
    db[nuevo_id] = producto
    return producto


@app.get("/productos/{producto_id}", response_model=ProductoOut)
def obtener_producto(producto_id: int):
    return db[producto_id]  # (en el nivel 3 manejamos el 404)
```

### 3.4 Tipo de retorno en lugar de `response_model`

FastAPI también lee la anotación de retorno:

```python
@app.get("/productos")
def listar() -> list[ProductoOut]:
    return list(db.values())
```

| Forma | Cuándo usarla |
|---|---|
| `-> ProductoOut` | Lo normal; mejor soporte del editor |
| `response_model=ProductoOut` | Cuando devuelves otro objeto (ej. entidad ORM o dict con campos extra) y quieres que FastAPI lo filtre |

> `response_model` **filtra** los campos: si devuelves un objeto con `password`, y el modelo de respuesta no lo tiene, no se envía. Es tu primera línea de defensa contra fugas de datos.

---

## 4. Nivel 2 — Intermedio: validación y modelado serio

### 4.1 `Field` y restricciones

```python
from pydantic import BaseModel, Field


class Producto(BaseModel):
    nombre: str = Field(min_length=2, max_length=100, examples=["Laptop"])
    precio: float = Field(gt=0, description="Precio en MXN")
    stock: int = Field(default=0, ge=0)
    sku: str = Field(pattern=r"^[A-Z]{3}-\d{4}$")
    tags: list[str] = Field(default_factory=list, max_length=5)
```

| Restricción | Aplica a | Significado |
|---|---|---|
| `gt`, `ge`, `lt`, `le` | números | >, >=, <, <= |
| `multiple_of` | números | múltiplo de |
| `min_length`, `max_length` | str, list | longitud |
| `pattern` | str | regex |
| `max_digits`, `decimal_places` | Decimal | precisión |
| `default_factory` | cualquiera | default generado (listas, fechas, uuid) |
| `alias` | cualquiera | nombre externo del campo |
| `description`, `examples`, `title` | cualquiera | documentación OpenAPI |
| `exclude=True` | cualquiera | nunca se serializa |
| `frozen=True` | cualquiera | inmutable |

### 4.2 `Annotated`: tipos reutilizables

La forma moderna y más limpia: defines el tipo con sus reglas **una vez** y lo reusas.

```python
from typing import Annotated
from pydantic import BaseModel, Field

Nombre = Annotated[str, Field(min_length=2, max_length=50)]
Precio = Annotated[float, Field(gt=0, le=1_000_000)]
Cantidad = Annotated[int, Field(ge=0)]


class Producto(BaseModel):
    nombre: Nombre
    precio: Precio
    stock: Cantidad = 0


class Categoria(BaseModel):
    nombre: Nombre
```

### 4.3 Enums

```python
from enum import Enum
from pydantic import BaseModel


class EstadoPedido(str, Enum):
    PENDIENTE = "pendiente"
    PAGADO = "pagado"
    ENVIADO = "enviado"
    CANCELADO = "cancelado"


class Pedido(BaseModel):
    id: int
    estado: EstadoPedido = EstadoPedido.PENDIENTE


p = Pedido(id=1, estado="pagado")
print(p.estado)                         # EstadoPedido.PAGADO
print(p.model_dump(mode="json"))        # {'id': 1, 'estado': 'pagado'}
```

En FastAPI, los enums aparecen como dropdown en Swagger.

### 4.4 Validadores de campo: `@field_validator`

```python
from pydantic import BaseModel, field_validator


class Registro(BaseModel):
    username: str
    telefono: str

    @field_validator("username")
    @classmethod
    def username_sin_espacios(cls, v: str) -> str:
        if " " in v:
            raise ValueError("El username no puede tener espacios")
        return v.lower()

    @field_validator("telefono", mode="before")
    @classmethod
    def limpiar_telefono(cls, v):
        # mode="before": corre ANTES de la validación de tipo
        if isinstance(v, str):
            v = v.replace(" ", "").replace("-", "")
        return v
```

| Modo | Cuándo corre | Recibe |
|---|---|---|
| `mode="after"` (default) | Después de validar el tipo | Valor ya tipado |
| `mode="before"` | Antes de validar el tipo | Valor crudo (cualquier cosa) |
| `mode="wrap"` | Envuelve la validación | Valor + handler, control total |
| `mode="plain"` | Reemplaza la validación | Valor crudo; Pydantic no valida el tipo |

### 4.5 Validadores de modelo: `@model_validator`

Para reglas que involucran **varios campos**:

```python
from datetime import date
from typing_extensions import Self   # en Python 3.11+: from typing import Self
from pydantic import BaseModel, model_validator


class CambioPassword(BaseModel):
    password: str
    confirmar: str

    @model_validator(mode="after")
    def passwords_iguales(self) -> Self:
        if self.password != self.confirmar:
            raise ValueError("Las contraseñas no coinciden")
        return self


class RangoFechas(BaseModel):
    inicio: date
    fin: date

    @model_validator(mode="after")
    def validar_rango(self) -> Self:
        if self.fin < self.inicio:
            raise ValueError("'fin' debe ser posterior a 'inicio'")
        return self
```

### 4.6 Campos calculados: `@computed_field`

```python
from pydantic import BaseModel, computed_field


class LineaPedido(BaseModel):
    precio_unitario: float
    cantidad: int

    @computed_field
    @property
    def subtotal(self) -> float:
        return round(self.precio_unitario * self.cantidad, 2)


l = LineaPedido(precio_unitario=99.9, cantidad=3)
print(l.model_dump())  # {'precio_unitario': 99.9, 'cantidad': 3, 'subtotal': 299.7}
```

Aparece en la respuesta JSON y en el esquema OpenAPI.

### 4.7 Configuración del modelo: `ConfigDict`

```python
from pydantic import BaseModel, ConfigDict


class Estricto(BaseModel):
    model_config = ConfigDict(
        extra="forbid",              # error si llegan campos desconocidos
        str_strip_whitespace=True,   # quita espacios de los strings
        frozen=True,                 # inmutable (y hasheable)
        validate_assignment=True,    # valida también al asignar u.campo = x
    )

    nombre: str
```

| Opción | Valores | Uso típico |
|---|---|---|
| `extra` | `"ignore"` (default), `"forbid"`, `"allow"` | `forbid` en inputs de API para detectar typos |
| `from_attributes` | `bool` | Leer desde objetos ORM (SQLAlchemy) |
| `str_strip_whitespace` | `bool` | Limpiar inputs de formularios |
| `str_to_lower` | `bool` | Normalizar |
| `frozen` | `bool` | Value objects inmutables |
| `validate_assignment` | `bool` | Validar al mutar |
| `strict` | `bool` | Sin coerción (`"1"` ya no pasa como `int`) |
| `alias_generator` | función | camelCase automático |
| `populate_by_name` | `bool` | Aceptar nombre Python además del alias |
| `use_enum_values` | `bool` | Guardar el `.value` del enum en lugar del miembro |
| `json_schema_extra` | dict | Ejemplos para Swagger |

### 4.8 Alias y camelCase (ideal para un frontend Angular)

Python usa `snake_case`, el frontend JS/TS suele usar `camelCase`. Pydantic lo resuelve sin tocar cada campo:

```python
from pydantic import BaseModel, ConfigDict
from pydantic.alias_generators import to_camel


class CamelModel(BaseModel):
    """Base para todos los schemas que viajan al frontend."""
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True,   # permite construir con nombre_python también
    )


class UsuarioOut(CamelModel):
    id: int
    nombre_completo: str
    fecha_registro: str


u = UsuarioOut(id=1, nombre_completo="Gil", fecha_registro="2026-10-05")
print(u.model_dump(by_alias=True))
# {'id': 1, 'nombreCompleto': 'Gil', 'fechaRegistro': '2026-10-05'}
```

FastAPI serializa las respuestas **con alias por defecto** (`response_model_by_alias=True`), así que el JSON sale en camelCase y el request también se acepta en camelCase.

---

## 5. Nivel 2 — Intermedio: patrón Create / Update / Response

En una API real **nunca** uses un solo modelo para todo. Separa los schemas por intención (equivalente a DTOs en Spring):

| Schema | Uso | Características |
|---|---|---|
| `XBase` | Campos compartidos | No se usa directamente |
| `XCreate` | Body del `POST` | Sin `id`, incluye `password` si aplica |
| `XUpdate` | Body del `PATCH` | Todos los campos opcionales |
| `XOut` / `XResponse` | Respuesta | Con `id`, timestamps, **sin** datos sensibles |
| `XInDB` | Interno | Incluye `hashed_password`, nunca sale de la API |

```python
from datetime import datetime
from pydantic import BaseModel, ConfigDict, EmailStr, Field


class UsuarioBase(BaseModel):
    email: EmailStr
    nombre: str = Field(min_length=2, max_length=80)


class UsuarioCreate(UsuarioBase):
    model_config = ConfigDict(extra="forbid")
    password: str = Field(min_length=8)


class UsuarioUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid")
    email: EmailStr | None = None
    nombre: str | None = Field(default=None, min_length=2, max_length=80)


class UsuarioOut(UsuarioBase):
    model_config = ConfigDict(from_attributes=True)
    id: int
    creado_en: datetime
```

### 5.1 PATCH correcto con `exclude_unset`

El truco clave: distinguir entre "no lo mandaron" y "lo mandaron como `null`".

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()
usuarios: dict[int, dict] = {}


@app.patch("/usuarios/{usuario_id}", response_model=UsuarioOut)
def actualizar_usuario(usuario_id: int, data: UsuarioUpdate):
    actual = usuarios.get(usuario_id)
    if actual is None:
        raise HTTPException(status_code=404, detail="Usuario no encontrado")

    cambios = data.model_dump(exclude_unset=True)   # SOLO lo que vino en el JSON
    actual.update(cambios)
    return actual
```

| Body enviado | `model_dump()` | `model_dump(exclude_unset=True)` |
|---|---|---|
| `{"nombre": "Ana"}` | `{"email": None, "nombre": "Ana"}` | `{"nombre": "Ana"}` |
| `{}` | `{"email": None, "nombre": None}` | `{}` |

Sin `exclude_unset` borrarías el email del usuario por accidente.

---

## 6. Nivel 3 — Avanzado: Pydantic a fondo

### 6.1 Tipos personalizados con `Annotated` + validadores funcionales

Reutilizables en cualquier modelo, sin decoradores:

```python
from typing import Annotated
from pydantic import AfterValidator, BeforeValidator, BaseModel


def validar_rfc(v: str) -> str:
    v = v.upper()
    if len(v) not in (12, 13):
        raise ValueError("RFC debe tener 12 o 13 caracteres")
    return v


def a_lista(v):
    # "a,b,c" -> ["a", "b", "c"]
    if isinstance(v, str):
        return [x.strip() for x in v.split(",") if x.strip()]
    return v


RFC = Annotated[str, AfterValidator(validar_rfc)]
ListaCSV = Annotated[list[str], BeforeValidator(a_lista)]


class Empresa(BaseModel):
    rfc: RFC
    giros: ListaCSV = []


print(Empresa(rfc="abc010101xyz", giros="software, consultoría"))
# rfc='ABC010101XYZ' giros=['software', 'consultoría']
```

### 6.2 Serializadores personalizados

```python
from datetime import datetime
from decimal import Decimal
from pydantic import BaseModel, field_serializer


class Factura(BaseModel):
    total: Decimal
    emitida: datetime

    @field_serializer("total")
    def serializar_total(self, v: Decimal) -> str:
        return f"{v:.2f}"

    @field_serializer("emitida")
    def serializar_fecha(self, v: datetime) -> str:
        return v.strftime("%d/%m/%Y %H:%M")
```

Versión reutilizable con `PlainSerializer`:

```python
from decimal import Decimal
from typing import Annotated
from pydantic import PlainSerializer

Dinero = Annotated[Decimal, PlainSerializer(lambda v: f"{v:.2f}", return_type=str, when_used="json")]
```

### 6.3 Uniones discriminadas (polimorfismo)

Cuando un campo puede ser de varios tipos y uno de sus campos indica cuál. Rápido y con errores claros:

```python
from typing import Annotated, Literal, Union
from pydantic import BaseModel, Field


class PagoTarjeta(BaseModel):
    tipo: Literal["tarjeta"]
    ultimos4: str


class PagoOxxo(BaseModel):
    tipo: Literal["oxxo"]
    referencia: str


class PagoTransferencia(BaseModel):
    tipo: Literal["transferencia"]
    clabe: str


MetodoPago = Annotated[
    Union[PagoTarjeta, PagoOxxo, PagoTransferencia],
    Field(discriminator="tipo"),
]


class Orden(BaseModel):
    id: int
    pago: MetodoPago


o = Orden.model_validate({"id": 1, "pago": {"tipo": "oxxo", "referencia": "9876"}})
print(type(o.pago).__name__)  # PagoOxxo
```

En Swagger aparece como `oneOf` con el discriminador, y en TypeScript se mapea perfecto a una *discriminated union*.

### 6.4 Modelos genéricos (respuestas paginadas)

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")


class Pagina(BaseModel, Generic[T]):
    items: list[T]
    total: int
    pagina: int
    tamano: int

    @property
    def paginas(self) -> int:
        return (self.total + self.tamano - 1) // self.tamano


class ProductoOut(BaseModel):
    id: int
    nombre: str


# En FastAPI:
# @app.get("/productos", response_model=Pagina[ProductoOut])
```

En Python 3.12+ puedes usar la sintaxis nueva: `class Pagina[T](BaseModel): ...`

### 6.5 `TypeAdapter`: validar cosas que no son modelos

```python
from pydantic import TypeAdapter

adapter = TypeAdapter(list[ProductoOut])

productos = adapter.validate_python([{"id": 1, "nombre": "Mouse"}, {"id": 2, "nombre": "Teclado"}])
json_bytes = adapter.dump_json(productos)

# También sirve para tipos simples
TypeAdapter(int).validate_python("42")  # 42
```

> Crea el `TypeAdapter` **una vez** a nivel de módulo; construirlo es costoso.

### 6.6 Modo estricto

```python
from pydantic import BaseModel, ConfigDict, Field, StrictInt


class Lax(BaseModel):
    n: int


class Estricto(BaseModel):
    model_config = ConfigDict(strict=True)
    n: int


class Mixto(BaseModel):
    n: StrictInt                     # solo este campo es estricto
    m: int = Field(strict=True)      # alternativa


Lax(n="5")        # OK -> 5
# Estricto(n="5") # ValidationError
```

### 6.7 Configuración con `pydantic-settings`

Equivalente al `application.properties` + `@ConfigurationProperties` de Spring:

```python
# app/core/config.py
from functools import lru_cache
from pydantic import Field, PostgresDsn, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )

    app_name: str = "Mi API"
    debug: bool = False
    database_url: PostgresDsn
    jwt_secret: SecretStr                      # no se imprime en logs
    jwt_expira_minutos: int = Field(default=60, gt=0)
    cors_origins: list[str] = ["http://localhost:4200"]


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

```env
# .env
DATABASE_URL=postgresql://user:pass@localhost:5432/midb
JWT_SECRET=super-secreto
DEBUG=true
CORS_ORIGINS=["http://localhost:4200","https://miapp.com"]
```

Las variables de entorno no distinguen mayúsculas por defecto, y las listas se pasan en formato JSON.

Uso del secreto: `settings.jwt_secret.get_secret_value()`.

### 6.8 JSON Schema y ejemplos

```python
from pydantic import BaseModel, ConfigDict


class ProductoCreate(BaseModel):
    model_config = ConfigDict(
        json_schema_extra={
            "examples": [{"nombre": "Monitor 27\"", "precio": 4599.0, "stock": 10}]
        }
    )
    nombre: str
    precio: float
    stock: int = 0


print(ProductoCreate.model_json_schema())
```

---

## 7. Nivel 3 — Avanzado: FastAPI en serio

### 7.1 Parámetros con validación: `Path`, `Query`, `Body`, `Header`

```python
from typing import Annotated
from fastapi import FastAPI, Header, Path, Query

app = FastAPI()


@app.get("/productos/{producto_id}")
def obtener(
    producto_id: Annotated[int, Path(gt=0, description="ID del producto")],
    q: Annotated[str | None, Query(min_length=3, max_length=50)] = None,
    tags: Annotated[list[str], Query()] = [],     # ?tags=a&tags=b
    x_request_id: Annotated[str | None, Header()] = None,
):
    return {"id": producto_id, "q": q, "tags": tags, "request_id": x_request_id}
```

### 7.2 Query params agrupados en un modelo

Muy útil para filtros de tablas (por ejemplo, un `p-table` con filtros y paginación):

```python
from typing import Annotated, Literal
from fastapi import FastAPI, Query
from pydantic import BaseModel, Field

app = FastAPI()


class FiltrosProducto(BaseModel):
    model_config = {"extra": "forbid"}

    pagina: int = Field(1, ge=1)
    tamano: int = Field(20, ge=1, le=100)
    orden: Literal["nombre", "precio", "creado_en"] = "creado_en"
    direccion: Literal["asc", "desc"] = "desc"
    busqueda: str | None = None


@app.get("/productos")
def listar(filtros: Annotated[FiltrosProducto, Query()]):
    return filtros
```

### 7.3 Inyección de dependencias con `Depends`

```python
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException, status

app = FastAPI()


def paginacion(pagina: int = 1, tamano: int = 20) -> dict:
    return {"offset": (pagina - 1) * tamano, "limit": tamano}


Paginacion = Annotated[dict, Depends(paginacion)]


@app.get("/clientes")
def listar_clientes(pag: Paginacion):
    return pag


# Dependencia que lanza error (ej. autenticación)
def usuario_actual(authorization: str | None = None) -> dict:
    if not authorization:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="No autenticado")
    return {"id": 1, "rol": "admin"}


UsuarioActual = Annotated[dict, Depends(usuario_actual)]


def requiere_admin(usuario: UsuarioActual) -> dict:
    if usuario["rol"] != "admin":
        raise HTTPException(status_code=403, detail="Sin permisos")
    return usuario


@app.delete("/clientes/{cliente_id}", dependencies=[Depends(requiere_admin)])
def eliminar_cliente(cliente_id: int):
    return {"eliminado": cliente_id}
```

### 7.4 Integración con SQLAlchemy 2.0

```python
# app/db.py
from collections.abc import Generator
from datetime import datetime
from sqlalchemy import String, create_engine, func
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column, sessionmaker

engine = create_engine("sqlite:///./app.db", connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(bind=engine, autoflush=False)


class Base(DeclarativeBase):
    pass


class ProductoORM(Base):
    __tablename__ = "productos"

    id: Mapped[int] = mapped_column(primary_key=True)
    nombre: Mapped[str] = mapped_column(String(100), index=True)
    precio: Mapped[float]
    stock: Mapped[int] = mapped_column(default=0)
    creado_en: Mapped[datetime] = mapped_column(server_default=func.now())


def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

```python
# app/main.py
from typing import Annotated
from datetime import datetime
from fastapi import Depends, FastAPI, HTTPException, status
from pydantic import BaseModel, ConfigDict, Field
from sqlalchemy import func, select
from sqlalchemy.orm import Session

from app.db import Base, ProductoORM, engine, get_db

Base.metadata.create_all(engine)
app = FastAPI()
DB = Annotated[Session, Depends(get_db)]


class ProductoCreate(BaseModel):
    nombre: str = Field(min_length=2, max_length=100)
    precio: float = Field(gt=0)
    stock: int = Field(0, ge=0)


class ProductoUpdate(BaseModel):
    nombre: str | None = Field(None, min_length=2, max_length=100)
    precio: float | None = Field(None, gt=0)
    stock: int | None = Field(None, ge=0)


class ProductoOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)   # <- clave para leer del ORM
    id: int
    nombre: str
    precio: float
    stock: int
    creado_en: datetime


class Pagina(BaseModel):
    items: list[ProductoOut]
    total: int


@app.post("/productos", response_model=ProductoOut, status_code=status.HTTP_201_CREATED)
def crear(data: ProductoCreate, db: DB):
    producto = ProductoORM(**data.model_dump())
    db.add(producto)
    db.commit()
    db.refresh(producto)
    return producto           # FastAPI lo convierte con from_attributes


@app.get("/productos", response_model=Pagina)
def listar(db: DB, pagina: int = 1, tamano: int = 20):
    total = db.scalar(select(func.count()).select_from(ProductoORM))
    items = db.scalars(
        select(ProductoORM).offset((pagina - 1) * tamano).limit(tamano)
    ).all()
    return {"items": items, "total": total}


@app.get("/productos/{producto_id}", response_model=ProductoOut)
def obtener(producto_id: int, db: DB):
    producto = db.get(ProductoORM, producto_id)
    if producto is None:
        raise HTTPException(status_code=404, detail="Producto no encontrado")
    return producto


@app.patch("/productos/{producto_id}", response_model=ProductoOut)
def actualizar(producto_id: int, data: ProductoUpdate, db: DB):
    producto = db.get(ProductoORM, producto_id)
    if producto is None:
        raise HTTPException(status_code=404, detail="Producto no encontrado")
    for campo, valor in data.model_dump(exclude_unset=True).items():
        setattr(producto, campo, valor)
    db.commit()
    db.refresh(producto)
    return producto


@app.delete("/productos/{producto_id}", status_code=status.HTTP_204_NO_CONTENT)
def eliminar(producto_id: int, db: DB):
    producto = db.get(ProductoORM, producto_id)
    if producto is None:
        raise HTTPException(status_code=404, detail="Producto no encontrado")
    db.delete(producto)
    db.commit()
```

> Conversión manual fuera de FastAPI: `ProductoOut.model_validate(producto_orm)`.

### 7.5 Manejo de errores personalizado

Formato de errores uniforme (útil para mostrarlos en un `p-toast` del frontend):

```python
from fastapi import FastAPI, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI()


class ErrorDetalle(BaseModel):
    campo: str
    mensaje: str


class ErrorRespuesta(BaseModel):
    codigo: str
    mensaje: str
    detalles: list[ErrorDetalle] = []


# Excepción de dominio propia
class NegocioError(Exception):
    def __init__(self, codigo: str, mensaje: str, status_code: int = 400):
        self.codigo = codigo
        self.mensaje = mensaje
        self.status_code = status_code


@app.exception_handler(NegocioError)
async def negocio_handler(request: Request, exc: NegocioError):
    body = ErrorRespuesta(codigo=exc.codigo, mensaje=exc.mensaje)
    return JSONResponse(status_code=exc.status_code, content=body.model_dump())


@app.exception_handler(RequestValidationError)
async def validacion_handler(request: Request, exc: RequestValidationError):
    detalles = [
        ErrorDetalle(
            campo=".".join(str(p) for p in err["loc"] if p != "body"),
            mensaje=err["msg"],
        )
        for err in exc.errors()
    ]
    body = ErrorRespuesta(codigo="VALIDACION", mensaje="Datos inválidos", detalles=detalles)
    return JSONResponse(status_code=status.HTTP_422_UNPROCESSABLE_ENTITY, content=body.model_dump())


@app.post("/retiros")
def retirar(monto: float):
    if monto > 10_000:
        raise NegocioError("LIMITE_EXCEDIDO", "El monto excede el límite diario")
    return {"ok": True}
```

Documentar las respuestas de error en OpenAPI:

```python
@app.get(
    "/productos/{producto_id}",
    response_model=ProductoOut,
    responses={404: {"model": ErrorRespuesta, "description": "No encontrado"}},
)
def obtener(producto_id: int): ...
```

### 7.6 Routers (equivalente a `@RestController`)

```python
# app/routers/productos.py
from fastapi import APIRouter

router = APIRouter(prefix="/productos", tags=["Productos"])


@router.get("")
def listar():
    return []
```

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.core.config import get_settings
from app.routers import productos

settings = get_settings()
app = FastAPI(title=settings.app_name)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(productos.router, prefix="/api/v1")
```

### 7.7 Async vs sync

| Usa `async def` | Usa `def` |
|---|---|
| Llamas librerías async (`httpx.AsyncClient`, SQLAlchemy async, `asyncpg`) | Llamas librerías bloqueantes (SQLAlchemy sync, `requests`) |
| Haces `await` dentro | No hay ningún `await` |

FastAPI ejecuta las funciones `def` en un threadpool, así que no bloquean. El error grave es usar código **bloqueante dentro de `async def`**: congela todo el event loop.

---

## 8. Nivel 4 — Pro: arquitectura, testing y rendimiento

### 8.1 Estructura de proyecto recomendada

```
app/
├── main.py                 # crea la app, middlewares, routers
├── core/
│   ├── config.py           # Settings (pydantic-settings)
│   ├── security.py         # JWT, hashing
│   └── exceptions.py       # excepciones de dominio + handlers
├── db/
│   ├── session.py          # engine, SessionLocal, get_db
│   └── base.py             # DeclarativeBase
├── models/                 # ORM (≈ @Entity)
│   └── producto.py
├── schemas/                # Pydantic (≈ DTOs)
│   ├── common.py           # CamelModel, Pagina[T], ErrorRespuesta
│   └── producto.py
├── repositories/           # acceso a datos (≈ @Repository)
│   └── producto_repo.py
├── services/               # lógica de negocio (≈ @Service)
│   └── producto_service.py
├── api/
│   ├── deps.py             # dependencias comunes (DB, usuario actual)
│   └── v1/
│       └── productos.py    # routers (≈ @RestController)
tests/
├── conftest.py
└── test_productos.py
```

### 8.2 Capa de servicio inyectable

```python
# app/services/producto_service.py
from typing import Annotated
from fastapi import Depends
from sqlalchemy.orm import Session

from app.api.deps import get_db
from app.core.exceptions import NegocioError
from app.models.producto import ProductoORM
from app.schemas.producto import ProductoCreate


class ProductoService:
    def __init__(self, db: Annotated[Session, Depends(get_db)]):
        self.db = db

    def crear(self, data: ProductoCreate) -> ProductoORM:
        existe = self.db.query(ProductoORM).filter_by(nombre=data.nombre).first()
        if existe:
            raise NegocioError("DUPLICADO", f"Ya existe '{data.nombre}'", 409)
        producto = ProductoORM(**data.model_dump())
        self.db.add(producto)
        self.db.commit()
        self.db.refresh(producto)
        return producto


ProductoServiceDep = Annotated[ProductoService, Depends()]
```

```python
# app/api/v1/productos.py
from fastapi import APIRouter, status
from app.schemas.producto import ProductoCreate, ProductoOut
from app.services.producto_service import ProductoServiceDep

router = APIRouter(prefix="/productos", tags=["Productos"])


@router.post("", response_model=ProductoOut, status_code=status.HTTP_201_CREATED)
def crear(data: ProductoCreate, service: ProductoServiceDep):
    return service.crear(data)
```

### 8.3 Testing con `TestClient` y override de dependencias

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import StaticPool

from app.db import Base, get_db
from app.main import app

engine = create_engine(
    "sqlite://",
    connect_args={"check_same_thread": False},
    poolclass=StaticPool,
)
TestingSession = sessionmaker(bind=engine)


@pytest.fixture
def client():
    Base.metadata.create_all(engine)

    def override_get_db():
        db = TestingSession()
        try:
            yield db
        finally:
            db.close()

    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
    Base.metadata.drop_all(engine)
```

```python
# tests/test_productos.py
def test_crear_producto(client):
    r = client.post("/productos", json={"nombre": "Mouse", "precio": 299.0})
    assert r.status_code == 201
    data = r.json()
    assert data["nombre"] == "Mouse"
    assert "id" in data


def test_precio_invalido(client):
    r = client.post("/productos", json={"nombre": "Mouse", "precio": -5})
    assert r.status_code == 422


def test_producto_no_existe(client):
    r = client.get("/productos/999")
    assert r.status_code == 404
```

Testear schemas directamente (sin levantar la API):

```python
import pytest
from pydantic import ValidationError
from app.schemas.producto import ProductoCreate


@pytest.mark.parametrize("precio", [0, -1, "abc"])
def test_precio_rechazado(precio):
    with pytest.raises(ValidationError):
        ProductoCreate(nombre="X1", precio=precio)
```

```bash
pytest -v
```

### 8.4 Rendimiento: buenas prácticas

| Práctica | Por qué |
|---|---|
| `Model.model_validate_json(raw)` en lugar de `json.loads` + `model_validate` | El parseo ocurre en Rust (pydantic-core) |
| Crear `TypeAdapter` a nivel de módulo | Construirlo compila el validador |
| Uniones discriminadas en lugar de `Union` simple | Pydantic va directo al tipo correcto sin probar todos |
| `Literal` / `Enum` en lugar de validadores con `if` | Validación nativa, más rápida |
| Evitar `mode="wrap"` salvo que sea necesario | Es el más lento de los validadores |
| `model_construct()` para datos ya confiables | Salta la validación (¡úsalo con cuidado!) |
| `response_model` solo cuando haga falta filtrar | Cada respuesta se valida y serializa |
| Async real para I/O de red | Mejor concurrencia |

### 8.5 Seguridad: checklist

| Riesgo | Solución con Pydantic/FastAPI |
|---|---|
| Campos inesperados (mass assignment) | `extra="forbid"` en schemas de entrada |
| Fuga de `password` / datos internos | Schemas `Out` separados + `response_model` |
| Secretos en logs | `SecretStr` en settings |
| Inputs gigantes | `max_length` en strings y listas |
| IDs negativos o cero | `Path(gt=0)` |
| Paginación abusiva | `tamano: int = Field(20, le=100)` |
| CORS abierto | `allow_origins` explícito desde settings |

### 8.6 Generar tipos TypeScript para el frontend

Como FastAPI expone `openapi.json`, puedes generar interfaces y servicios para Angular automáticamente:

```bash
# Desde el proyecto Angular, con la API corriendo
npx @openapitools/openapi-generator-cli generate \
  -i http://localhost:8000/openapi.json \
  -g typescript-angular \
  -o src/app/api
```

Con alias camelCase (sección 4.8) las interfaces generadas quedan en camelCase, igual que en el resto del código Angular.

---

## 9. Referencia: Pydantic vs Spring Boot

| Concepto | Spring Boot (Java 17) | FastAPI + Pydantic |
|---|---|---|
| DTO | `record ProductoDTO(...)` | `class ProductoCreate(BaseModel)` |
| `@NotNull` | `@NotNull String nombre` | `nombre: str` (requerido por defecto) |
| `@Size(min=2, max=100)` | `@Size(...)` | `Field(min_length=2, max_length=100)` |
| `@Min(0)` / `@Positive` | `@Min(0)` / `@Positive` | `Field(ge=0)` / `Field(gt=0)` |
| `@Email` | `@Email` | `EmailStr` |
| `@Pattern` | `@Pattern(regexp=...)` | `Field(pattern=...)` |
| Validador custom | `ConstraintValidator` | `@field_validator` / `AfterValidator` |
| Validación de clase | `@ScriptAssert` / validador de clase | `@model_validator(mode="after")` |
| `@Valid @RequestBody` | `@Valid @RequestBody ProductoDTO dto` | `data: ProductoCreate` (automático) |
| `@PathVariable` | `@PathVariable Long id` | `id: int` en la ruta `/{id}` |
| `@RequestParam` | `@RequestParam int page` | `page: int = 1` |
| `@RestController` + `@RequestMapping` | Clase anotada | `APIRouter(prefix=...)` |
| `@Autowired` / constructor injection | Contenedor IoC | `Depends(...)` |
| `@ControllerAdvice` | `@ExceptionHandler` | `@app.exception_handler(...)` |
| `@ConfigurationProperties` | `application.yml` | `BaseSettings` + `.env` |
| `@JsonProperty("x")` | Jackson | `Field(alias="x")` |
| `PropertyNamingStrategies.LOWER_CAMEL_CASE` | Jackson global | `alias_generator=to_camel` |
| `@JsonIgnore` | Jackson | `Field(exclude=True)` o schema `Out` sin el campo |
| `@JsonSubTypes` | Polimorfismo Jackson | Unión discriminada |
| MapStruct | Mapper de entidad a DTO | `from_attributes=True` + `model_validate` |
| Springdoc OpenAPI | Dependencia extra | Incluido (`/docs`) |
| `MockMvc` + `@MockBean` | Test de controller | `TestClient` + `dependency_overrides` |
| Respuesta de validación | 400 Bad Request | 422 Unprocessable Entity |

---

## 10. Referencia: migración v1 → v2

| Pydantic v1 | Pydantic v2 |
|---|---|
| `.dict()` | `.model_dump()` |
| `.json()` | `.model_dump_json()` |
| `.parse_obj(d)` | `.model_validate(d)` |
| `.parse_raw(s)` | `.model_validate_json(s)` |
| `.copy(update=...)` | `.model_copy(update=...)` |
| `.schema()` | `.model_json_schema()` |
| `.construct()` | `.model_construct()` |
| `class Config:` | `model_config = ConfigDict(...)` |
| `orm_mode = True` | `from_attributes=True` |
| `allow_population_by_field_name` | `populate_by_name` |
| `@validator` | `@field_validator` (con `@classmethod`) |
| `@root_validator` | `@model_validator` |
| `parse_obj_as(List[X], d)` | `TypeAdapter(list[X]).validate_python(d)` |
| `BaseSettings` en `pydantic` | `BaseSettings` en `pydantic_settings` |
| `Optional[X]` era opcional implícitamente | `X \| None` sigue siendo **requerido** sin default |
| `const=True` | `Literal[...]` |
| `regex=` | `pattern=` |

---

## 11. Cheatsheet

```python
from typing import Annotated, Literal
from pydantic import (
    BaseModel, ConfigDict, Field, EmailStr, SecretStr,
    field_validator, model_validator, computed_field,
    AfterValidator, BeforeValidator, TypeAdapter, ValidationError,
)
from pydantic.alias_generators import to_camel
from pydantic_settings import BaseSettings, SettingsConfigDict
from fastapi import FastAPI, APIRouter, Depends, HTTPException, Path, Query, status
```

| Necesito... | Uso |
|---|---|
| Campo requerido | `x: int` |
| Campo opcional | `x: int \| None = None` |
| Default mutable | `x: list[str] = Field(default_factory=list)` |
| Validar rango | `Field(ge=0, le=100)` |
| Validar regex | `Field(pattern=r"...")` |
| Tipo reutilizable | `Annotated[str, Field(...)]` |
| Lógica de un campo | `@field_validator("x")` |
| Lógica entre campos | `@model_validator(mode="after")` |
| Campo derivado | `@computed_field` + `@property` |
| Rechazar campos extra | `ConfigDict(extra="forbid")` |
| Leer de ORM | `ConfigDict(from_attributes=True)` |
| camelCase | `ConfigDict(alias_generator=to_camel, populate_by_name=True)` |
| PATCH parcial | `model_dump(exclude_unset=True)` |
| Polimorfismo | `Annotated[A \| B, Field(discriminator="tipo")]` |
| Validar listas sueltas | `TypeAdapter(list[X])` |
| Config de entorno | `BaseSettings` + `.env` |
| 404 | `raise HTTPException(404, "No encontrado")` |
| Inyección | `Annotated[T, Depends(fn)]` |
| Mock en tests | `app.dependency_overrides[dep] = fake` |
