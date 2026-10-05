# SQLAlchemy + FastAPI: de principiante a pro

Guía práctica con ejemplos listos para copiar, escrita para **SQLAlchemy 2.0** (estilo moderno con `Mapped`, `mapped_column` y `select()`) y **FastAPI**. Complementa la guía `pydantic.md`: aquí SQLAlchemy es la capa de persistencia y Pydantic la capa de entrada y salida de la API.

## Índice

1. [Instalación y entorno](#1-instalación-y-entorno)
2. [Nivel 1 — Principiante: conceptos y primer modelo](#2-nivel-1--principiante-conceptos-y-primer-modelo)
3. [Nivel 1 — Principiante: CRUD con la Session](#3-nivel-1--principiante-crud-con-la-session)
4. [Nivel 1 — Principiante: conectar con FastAPI](#4-nivel-1--principiante-conectar-con-fastapi)
5. [Nivel 2 — Intermedio: columnas, tipos y restricciones](#5-nivel-2--intermedio-columnas-tipos-y-restricciones)
6. [Nivel 2 — Intermedio: relaciones](#6-nivel-2--intermedio-relaciones)
7. [Nivel 2 — Intermedio: consultas](#7-nivel-2--intermedio-consultas)
8. [Nivel 2 — Intermedio: carga de relaciones y el problema N+1](#8-nivel-2--intermedio-carga-de-relaciones-y-el-problema-n1)
9. [Nivel 3 — Avanzado: migraciones con Alembic](#9-nivel-3--avanzado-migraciones-con-alembic)
10. [Nivel 3 — Avanzado: transacciones](#10-nivel-3--avanzado-transacciones)
11. [Nivel 3 — Avanzado: SQLAlchemy async con FastAPI](#11-nivel-3--avanzado-sqlalchemy-async-con-fastapi)
12. [Nivel 3 — Avanzado: técnicas de modelado](#12-nivel-3--avanzado-técnicas-de-modelado)
13. [Nivel 4 — Pro: arquitectura y repositorios](#13-nivel-4--pro-arquitectura-y-repositorios)
14. [Nivel 4 — Pro: testing](#14-nivel-4--pro-testing)
15. [Nivel 4 — Pro: rendimiento y producción](#15-nivel-4--pro-rendimiento-y-producción)
16. [Referencia: SQLAlchemy vs JPA/Hibernate](#16-referencia-sqlalchemy-vs-jpahibernate)
17. [Referencia: migración 1.x → 2.0](#17-referencia-migración-1x--20)
18. [Cheatsheet](#18-cheatsheet)

---

## 1. Instalación y entorno

```bash
python -m venv .venv
source .venv/bin/activate        # Linux/Mac
.venv\Scripts\activate           # Windows

pip install "fastapi[standard]" sqlalchemy pydantic-settings

# Driver según la base de datos
pip install "psycopg[binary]"    # PostgreSQL (sync y async)
pip install asyncpg              # PostgreSQL async (alternativa muy rápida)
pip install pymysql              # MySQL / MariaDB
pip install aiosqlite            # SQLite async

# Migraciones y testing
pip install alembic pytest httpx
```

### URLs de conexión

| Base de datos | URL sync | URL async |
|---|---|---|
| SQLite (archivo) | `sqlite:///./app.db` | `sqlite+aiosqlite:///./app.db` |
| SQLite (memoria) | `sqlite://` | `sqlite+aiosqlite://` |
| PostgreSQL (psycopg 3) | `postgresql+psycopg://user:pass@host:5432/db` | `postgresql+psycopg://user:pass@host:5432/db` |
| PostgreSQL (asyncpg) | — | `postgresql+asyncpg://user:pass@host:5432/db` |
| MySQL | `mysql+pymysql://user:pass@host:3306/db` | `mysql+aiomysql://user:pass@host:3306/db` |
| SQL Server | `mssql+pyodbc://user:pass@dsn` | `mssql+aioodbc://user:pass@dsn` |

> Si la contraseña tiene caracteres especiales (`@`, `:`, `/`), escápala con `urllib.parse.quote_plus` o construye la URL con `sqlalchemy.URL.create(...)`.

---

## 2. Nivel 1 — Principiante: conceptos y primer modelo

### 2.1 Las piezas principales

| Pieza | Qué es | Equivalente aproximado en Spring |
|---|---|---|
| `Engine` | Pool de conexiones + dialecto de la BD. Uno por aplicación | `DataSource` |
| `Session` | Unidad de trabajo: rastrea objetos, agrupa cambios, maneja la transacción | `EntityManager` / persistence context |
| `sessionmaker` | Fábrica configurada de sesiones | `EntityManagerFactory` |
| `DeclarativeBase` | Clase base de tus modelos; guarda el `MetaData` | `@Entity` + escaneo de entidades |
| `Mapped[T]` | Anotación que declara una columna/relación tipada | Campo de la entidad |
| `mapped_column()` | Configuración de la columna | `@Column` |
| `relationship()` | Relación entre modelos | `@OneToMany`, `@ManyToOne`... |
| `select()` | Constructor de consultas | Criteria API / JPQL |

SQLAlchemy tiene dos capas:

| Capa | Para qué |
|---|---|
| **Core** | SQL expresado en Python: tablas, `select`, `insert`, `update`. Sin objetos de dominio |
| **ORM** | Mapea clases a tablas y añade la `Session`, relaciones e identity map |

En esta guía usamos el ORM, pero las consultas (`select`, `update`, `delete`) son las mismas en ambas capas desde la versión 2.0.

### 2.2 Primer modelo

```python
from datetime import datetime
from sqlalchemy import String, create_engine, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Producto(Base):
    __tablename__ = "productos"

    id: Mapped[int] = mapped_column(primary_key=True)             # autoincremental
    nombre: Mapped[str] = mapped_column(String(100))               # NOT NULL
    descripcion: Mapped[str | None]                                # NULL permitido
    precio: Mapped[float]
    stock: Mapped[int] = mapped_column(default=0)
    creado_en: Mapped[datetime] = mapped_column(server_default=func.now())

    def __repr__(self) -> str:
        return f"Producto(id={self.id!r}, nombre={self.nombre!r})"


engine = create_engine("sqlite:///./tienda.db", echo=True)  # echo=True imprime el SQL
Base.metadata.create_all(engine)                              # crea las tablas que falten
```

Reglas de nulabilidad con `Mapped`:

| Declaración | Columna generada |
|---|---|
| `Mapped[str]` | `NOT NULL` |
| `Mapped[str \| None]` | `NULL` |
| `Mapped[str] = mapped_column(nullable=True)` | `NULL` (la anotación manda menos que el argumento explícito) |

> `create_all` solo crea tablas que **no existen**; no modifica columnas de tablas existentes. Para cambios de esquema en proyectos reales se usa Alembic (sección 9).

---

## 3. Nivel 1 — Principiante: CRUD con la Session

### 3.1 Crear una sesión

```python
from sqlalchemy.orm import Session, sessionmaker

SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)

with SessionLocal() as session:
    ...  # la sesión se cierra al salir del bloque
```

### 3.2 Crear (INSERT)

```python
with SessionLocal() as session:
    mouse = Producto(nombre="Mouse", precio=299.0, stock=50)
    teclado = Producto(nombre="Teclado", precio=799.0, stock=20)

    session.add(mouse)
    session.add_all([teclado])
    session.commit()          # INSERT + COMMIT

    print(mouse.id)           # el id ya está asignado
```

### 3.3 Leer (SELECT)

```python
from sqlalchemy import select

with SessionLocal() as session:
    # Por llave primaria (usa el identity map, puede no ir a la BD)
    p = session.get(Producto, 1)

    # Todos
    todos = session.scalars(select(Producto)).all()

    # Con filtro
    baratos = session.scalars(
        select(Producto).where(Producto.precio < 500).order_by(Producto.precio)
    ).all()

    # Uno solo
    uno = session.scalars(select(Producto).where(Producto.nombre == "Mouse")).first()

    # Exactamente uno (error si hay 0 o más de 1)
    exacto = session.scalars(select(Producto).where(Producto.id == 1)).one()

    # Uno o None (error si hay más de 1)
    quizas = session.scalars(select(Producto).where(Producto.id == 999)).one_or_none()

    # Un valor escalar
    total = session.scalar(select(func.count()).select_from(Producto))
```

| Método | Devuelve | Si no hay filas | Si hay varias |
|---|---|---|---|
| `.all()` | `list` | `[]` | todas |
| `.first()` | objeto o `None` | `None` | la primera |
| `.one()` | objeto | error `NoResultFound` | error `MultipleResultsFound` |
| `.one_or_none()` | objeto o `None` | `None` | error `MultipleResultsFound` |
| `session.scalar(stmt)` | primer valor de la primera fila | `None` | el primero |
| `session.get(Modelo, pk)` | objeto o `None` | `None` | — |

> `session.execute(stmt)` devuelve **filas** (tuplas); `session.scalars(stmt)` devuelve la **primera columna** de cada fila, que es lo que quieres al seleccionar un modelo completo.

### 3.4 Actualizar (UPDATE)

```python
with SessionLocal() as session:
    p = session.get(Producto, 1)
    p.precio = 349.0          # la Session detecta el cambio (dirty tracking)
    session.commit()          # UPDATE automático
```

Actualización masiva sin cargar objetos:

```python
from sqlalchemy import update

with SessionLocal() as session:
    session.execute(
        update(Producto).where(Producto.stock == 0).values(precio=Producto.precio * 0.9)
    )
    session.commit()
```

### 3.5 Eliminar (DELETE)

```python
from sqlalchemy import delete

with SessionLocal() as session:
    p = session.get(Producto, 2)
    if p:
        session.delete(p)
        session.commit()

    # Masivo
    session.execute(delete(Producto).where(Producto.stock < 0))
    session.commit()
```

### 3.6 Estados de un objeto

| Estado | Significado | Cómo se llega |
|---|---|---|
| Transient | Objeto nuevo, la Session no lo conoce | `Producto(...)` |
| Pending | Añadido, aún sin INSERT | `session.add(obj)` |
| Persistent | Tiene fila en la BD y está en la Session | `flush`/`commit`, o cargado con una consulta |
| Detached | Tiene fila pero la Session se cerró | `session.close()` |
| Deleted | Marcado para borrar | `session.delete(obj)` + `flush` |

`flush()` envía el SQL pendiente **sin** hacer commit (útil para obtener el `id` dentro de una transacción). `commit()` hace `flush` y confirma la transacción.

---

## 4. Nivel 1 — Principiante: conectar con FastAPI

La regla básica: **una Session por request**, entregada con una dependencia.

```python
# main.py
from collections.abc import Generator
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, status
from pydantic import BaseModel, ConfigDict, Field
from sqlalchemy import String, create_engine, select
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column, sessionmaker

# ---------- BD ----------
engine = create_engine("sqlite:///./app.db", connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


class ProductoORM(Base):
    __tablename__ = "productos"
    id: Mapped[int] = mapped_column(primary_key=True)
    nombre: Mapped[str] = mapped_column(String(100), unique=True)
    precio: Mapped[float]
    stock: Mapped[int] = mapped_column(default=0)


Base.metadata.create_all(engine)


def get_db() -> Generator[Session, None, None]:
    with SessionLocal() as session:
        yield session


DB = Annotated[Session, Depends(get_db)]


# ---------- Schemas (Pydantic) ----------
class ProductoCreate(BaseModel):
    nombre: str = Field(min_length=2, max_length=100)
    precio: float = Field(gt=0)
    stock: int = Field(0, ge=0)


class ProductoUpdate(BaseModel):
    nombre: str | None = Field(None, min_length=2, max_length=100)
    precio: float | None = Field(None, gt=0)
    stock: int | None = Field(None, ge=0)


class ProductoOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    nombre: str
    precio: float
    stock: int


# ---------- API ----------
app = FastAPI()


def obtener_o_404(db: Session, producto_id: int) -> ProductoORM:
    producto = db.get(ProductoORM, producto_id)
    if producto is None:
        raise HTTPException(status_code=404, detail="Producto no encontrado")
    return producto


@app.post("/productos", response_model=ProductoOut, status_code=status.HTTP_201_CREATED)
def crear(data: ProductoCreate, db: DB):
    producto = ProductoORM(**data.model_dump())
    db.add(producto)
    db.commit()
    db.refresh(producto)
    return producto


@app.get("/productos", response_model=list[ProductoOut])
def listar(db: DB, skip: int = 0, limit: int = 20):
    return db.scalars(select(ProductoORM).offset(skip).limit(limit)).all()


@app.get("/productos/{producto_id}", response_model=ProductoOut)
def obtener(producto_id: int, db: DB):
    return obtener_o_404(db, producto_id)


@app.patch("/productos/{producto_id}", response_model=ProductoOut)
def actualizar(producto_id: int, data: ProductoUpdate, db: DB):
    producto = obtener_o_404(db, producto_id)
    for campo, valor in data.model_dump(exclude_unset=True).items():
        setattr(producto, campo, valor)
    db.commit()
    return producto


@app.delete("/productos/{producto_id}", status_code=status.HTTP_204_NO_CONTENT)
def eliminar(producto_id: int, db: DB):
    db.delete(obtener_o_404(db, producto_id))
    db.commit()
```

| Detalle | Por qué |
|---|---|
| `check_same_thread=False` | Solo SQLite: FastAPI ejecuta los endpoints `def` en varios hilos |
| `expire_on_commit=False` | Tras `commit` los atributos siguen disponibles sin otra consulta; evita sorpresas al serializar |
| `from_attributes=True` | Permite que Pydantic lea los atributos del objeto ORM |
| `with SessionLocal() as session: yield session` | Garantiza que la sesión se cierre aunque haya excepción |

---

## 5. Nivel 2 — Intermedio: columnas, tipos y restricciones

### 5.1 Tipos de columna

| Python (`Mapped[...]`) | Tipo SQL por defecto | Para precisar |
|---|---|---|
| `int` | `INTEGER` | `BigInteger`, `SmallInteger` |
| `float` | `FLOAT` | `Float(precision=...)` |
| `Decimal` | `NUMERIC` | `Numeric(10, 2)` (dinero) |
| `str` | `VARCHAR` sin longitud | `String(100)`, `Text` |
| `bool` | `BOOLEAN` | |
| `datetime` | `DATETIME` / `TIMESTAMP` | `DateTime(timezone=True)` |
| `date`, `time` | `DATE`, `TIME` | |
| `uuid.UUID` | `UUID` (o `CHAR(32)` en BD sin UUID) | `Uuid` |
| `bytes` | `BLOB` / `BYTEA` | `LargeBinary` |
| `enum.Enum` | `ENUM` / `VARCHAR` | `Enum(MiEnum, native_enum=False)` |
| `dict`, `list` | — (hay que indicarlo) | `JSON`, `JSONB` (PostgreSQL) |

> PostgreSQL y MySQL exigen longitud o tipo explícito en varios casos. Ponle `String(n)` a los textos cortos desde el principio.

### 5.2 Opciones de `mapped_column`

```python
import uuid
from datetime import datetime
from decimal import Decimal
from sqlalchemy import CheckConstraint, DateTime, Index, Numeric, String, UniqueConstraint, func
from sqlalchemy.orm import Mapped, mapped_column


class Articulo(Base):
    __tablename__ = "articulos"
    __table_args__ = (
        UniqueConstraint("sku", "proveedor", name="uq_articulo_sku_proveedor"),
        CheckConstraint("precio >= 0", name="ck_articulo_precio_positivo"),
        Index("ix_articulo_nombre_activo", "nombre", "activo"),
    )

    id: Mapped[int] = mapped_column(primary_key=True)
    codigo: Mapped[uuid.UUID] = mapped_column(default=uuid.uuid4, unique=True)
    sku: Mapped[str] = mapped_column(String(20))
    proveedor: Mapped[str] = mapped_column(String(50))
    nombre: Mapped[str] = mapped_column(String(120), index=True)
    precio: Mapped[Decimal] = mapped_column(Numeric(10, 2))
    activo: Mapped[bool] = mapped_column(default=True, server_default="1")
    creado_en: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
    actualizado_en: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
    )
```

| Opción | Qué hace |
|---|---|
| `primary_key=True` | Llave primaria |
| `nullable=False/True` | Fuerza la nulabilidad (si no, la decide `Mapped`) |
| `unique=True` | Restricción UNIQUE |
| `index=True` | Crea índice |
| `default=valor_o_funcion` | Default calculado **en Python** al hacer INSERT |
| `server_default=...` | Default definido **en la BD** (aparece en el DDL) |
| `onupdate=...` | Valor que se pone en cada UPDATE desde el ORM |
| `name="col_bd"` | Nombre real de la columna si difiere del atributo |
| `comment="..."` | Comentario en el esquema |

> `default` vs `server_default`: usa `server_default` cuando otros sistemas también insertan en la tabla, o cuando quieres que la regla viva en la BD.

### 5.3 Enums

```python
import enum
from sqlalchemy import Enum


class EstadoPedido(str, enum.Enum):
    PENDIENTE = "pendiente"
    PAGADO = "pagado"
    ENVIADO = "enviado"
    CANCELADO = "cancelado"


class PedidoSimple(Base):
    __tablename__ = "pedidos_simples"
    id: Mapped[int] = mapped_column(primary_key=True)
    # native_enum=False -> VARCHAR + CHECK; más fácil de migrar que un ENUM nativo
    estado: Mapped[EstadoPedido] = mapped_column(
        Enum(EstadoPedido, native_enum=False, length=20,
             values_callable=lambda e: [m.value for m in e]),
        default=EstadoPedido.PENDIENTE,
    )
```

`values_callable` guarda `"pendiente"` en la BD en lugar del nombre del miembro (`"PENDIENTE"`).

### 5.4 Mixins: columnas reutilizables

```python
from datetime import datetime
from sqlalchemy import DateTime, func
from sqlalchemy.orm import Mapped, mapped_column


class IdMixin:
    id: Mapped[int] = mapped_column(primary_key=True)


class TimestampMixin:
    creado_en: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
    actualizado_en: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
    )


class Proveedor(IdMixin, TimestampMixin, Base):
    __tablename__ = "proveedores"
    nombre: Mapped[str] = mapped_column(String(100))
```

### 5.5 Tipos con nombre propio (`Annotated`)

Igual que en Pydantic, defines un tipo una vez y lo reutilizas:

```python
from typing import Annotated
from sqlalchemy import String
from sqlalchemy.orm import mapped_column

intpk = Annotated[int, mapped_column(primary_key=True)]
str50 = Annotated[str, mapped_column(String(50))]
str255 = Annotated[str, mapped_column(String(255))]
creado = Annotated[datetime, mapped_column(DateTime(timezone=True), server_default=func.now())]


class Marca(Base):
    __tablename__ = "marcas"
    id: Mapped[intpk]
    nombre: Mapped[str50]
    sitio_web: Mapped[str255 | None]
    creado_en: Mapped[creado]
```

---

## 6. Nivel 2 — Intermedio: relaciones

Usaremos este dominio para el resto de la guía:

| Relación | Ejemplo |
|---|---|
| Uno a uno | `Cliente` ↔ `PerfilCliente` |
| Uno a muchos / muchos a uno | `Categoria` 1—N `Producto`, `Cliente` 1—N `Pedido` |
| Muchos a muchos simple | `Producto` N—M `Etiqueta` |
| Muchos a muchos con datos (association object) | `Pedido` N—M `Producto` vía `LineaPedido` (cantidad, precio) |

### 6.1 Modelo completo

```python
# app/models.py
from datetime import datetime
from decimal import Decimal
from sqlalchemy import Column, DateTime, ForeignKey, Numeric, String, Table, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


# ---------- Muchos a muchos simple: tabla de asociación ----------
producto_etiqueta = Table(
    "producto_etiqueta",
    Base.metadata,
    Column("producto_id", ForeignKey("productos.id", ondelete="CASCADE"), primary_key=True),
    Column("etiqueta_id", ForeignKey("etiquetas.id", ondelete="CASCADE"), primary_key=True),
)


class Cliente(Base):
    __tablename__ = "clientes"

    id: Mapped[int] = mapped_column(primary_key=True)
    nombre: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True)

    # 1 a 1
    perfil: Mapped["PerfilCliente | None"] = relationship(
        back_populates="cliente", cascade="all, delete-orphan"
    )
    # 1 a N
    pedidos: Mapped[list["Pedido"]] = relationship(
        back_populates="cliente", cascade="all, delete-orphan"
    )


class PerfilCliente(Base):
    __tablename__ = "perfiles_cliente"

    id: Mapped[int] = mapped_column(primary_key=True)
    cliente_id: Mapped[int] = mapped_column(ForeignKey("clientes.id"), unique=True)
    telefono: Mapped[str | None] = mapped_column(String(20))
    ciudad: Mapped[str | None] = mapped_column(String(80))

    cliente: Mapped["Cliente"] = relationship(back_populates="perfil")


class Categoria(Base):
    __tablename__ = "categorias"

    id: Mapped[int] = mapped_column(primary_key=True)
    nombre: Mapped[str] = mapped_column(String(80), unique=True)

    productos: Mapped[list["Producto"]] = relationship(back_populates="categoria")


class Etiqueta(Base):
    __tablename__ = "etiquetas"

    id: Mapped[int] = mapped_column(primary_key=True)
    nombre: Mapped[str] = mapped_column(String(40), unique=True)

    productos: Mapped[list["Producto"]] = relationship(
        secondary=producto_etiqueta, back_populates="etiquetas"
    )


class Producto(Base):
    __tablename__ = "productos"

    id: Mapped[int] = mapped_column(primary_key=True)
    nombre: Mapped[str] = mapped_column(String(100))
    precio: Mapped[Decimal] = mapped_column(Numeric(10, 2))
    stock: Mapped[int] = mapped_column(default=0)
    categoria_id: Mapped[int | None] = mapped_column(
        ForeignKey("categorias.id", ondelete="SET NULL"), index=True
    )

    # N a 1
    categoria: Mapped["Categoria | None"] = relationship(back_populates="productos")
    # N a M simple
    etiquetas: Mapped[list["Etiqueta"]] = relationship(
        secondary=producto_etiqueta, back_populates="productos"
    )
    # N a M con datos
    lineas: Mapped[list["LineaPedido"]] = relationship(back_populates="producto")


class Pedido(Base):
    __tablename__ = "pedidos"

    id: Mapped[int] = mapped_column(primary_key=True)
    cliente_id: Mapped[int] = mapped_column(ForeignKey("clientes.id"), index=True)
    creado_en: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

    cliente: Mapped["Cliente"] = relationship(back_populates="pedidos")
    lineas: Mapped[list["LineaPedido"]] = relationship(
        back_populates="pedido", cascade="all, delete-orphan"
    )

    @property
    def total(self) -> Decimal:
        return sum((l.subtotal for l in self.lineas), Decimal("0"))


class LineaPedido(Base):
    """Association object: N a M con columnas propias."""
    __tablename__ = "lineas_pedido"

    pedido_id: Mapped[int] = mapped_column(ForeignKey("pedidos.id"), primary_key=True)
    producto_id: Mapped[int] = mapped_column(ForeignKey("productos.id"), primary_key=True)
    cantidad: Mapped[int]
    precio_unitario: Mapped[Decimal] = mapped_column(Numeric(10, 2))

    pedido: Mapped["Pedido"] = relationship(back_populates="lineas")
    producto: Mapped["Producto"] = relationship(back_populates="lineas")

    @property
    def subtotal(self) -> Decimal:
        return self.precio_unitario * self.cantidad
```

### 6.2 Usar las relaciones

```python
from decimal import Decimal
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

engine = create_engine("sqlite://")
Base.metadata.create_all(engine)

with Session(engine) as session:
    perifericos = Categoria(nombre="Periféricos")
    oferta, gamer = Etiqueta(nombre="oferta"), Etiqueta(nombre="gamer")

    mouse = Producto(nombre="Mouse", precio=Decimal("299.00"), stock=50,
                     categoria=perifericos, etiquetas=[oferta, gamer])
    teclado = Producto(nombre="Teclado", precio=Decimal("799.00"), stock=20,
                       categoria=perifericos, etiquetas=[gamer])

    gil = Cliente(nombre="Gil", email="gil@example.com",
                  perfil=PerfilCliente(telefono="6181234567", ciudad="Durango"))

    pedido = Pedido(cliente=gil)
    pedido.lineas.append(LineaPedido(producto=mouse, cantidad=2, precio_unitario=mouse.precio))
    pedido.lineas.append(LineaPedido(producto=teclado, cantidad=1, precio_unitario=teclado.precio))

    session.add(gil)          # por cascada se guardan perfil, pedido y líneas;
    session.commit()          # productos, categoría y etiquetas se guardan por la relación

    print(pedido.total)                         # 1397.00
    print([e.nombre for e in mouse.etiquetas])  # ['oferta', 'gamer']
    print(perifericos.productos)                # back_populates sincroniza ambos lados
```

### 6.3 `cascade` y `ondelete`

| Configuración | Dónde actúa | Efecto |
|---|---|---|
| `cascade="save-update, merge"` (default) | ORM | Al añadir el padre, se añaden los hijos |
| `cascade="all, delete-orphan"` | ORM | Borrar el padre borra los hijos; quitar un hijo de la lista lo borra |
| `ForeignKey(..., ondelete="CASCADE")` | Base de datos | La BD borra los hijos (también funciona con `delete()` masivo) |
| `ForeignKey(..., ondelete="SET NULL")` | Base de datos | La FK de los hijos queda en NULL |
| `relationship(passive_deletes=True)` | ORM | El ORM no carga los hijos para borrarlos; confía en el `ondelete` de la BD |

> `delete-orphan` solo tiene sentido cuando el hijo **no puede existir sin el padre** (líneas de pedido, perfil). No lo pongas en `Categoria.productos`.

> SQLite no aplica llaves foráneas por defecto. Actívalas con un evento (ver sección 15.4).

### 6.4 `back_populates` vs `backref`

| Opción | Recomendación |
|---|---|
| `back_populates="x"` en ambos lados | Recomendado: explícito y con tipos en ambas clases |
| `backref="x"` en un solo lado | Estilo antiguo; crea el atributo "mágicamente", sin tipado |

---

## 7. Nivel 2 — Intermedio: consultas

Todas las consultas siguen el patrón **construir `select()` → ejecutarlo con la session**.

### 7.1 Filtros

```python
from sqlalchemy import and_, between, not_, or_, select

stmt = select(Producto).where(
    Producto.precio >= 100,                          # varios where = AND
    Producto.nombre.ilike("%mouse%"),                # LIKE sin mayúsculas
)

stmt = select(Producto).where(
    or_(Producto.stock == 0, Producto.precio > 5000)
)
```

| Operador | Ejemplo |
|---|---|
| `==`, `!=`, `<`, `<=`, `>`, `>=` | `Producto.precio > 100` |
| `IN` / `NOT IN` | `Producto.id.in_([1, 2, 3])` / `.not_in([...])` |
| `IS NULL` | `Producto.categoria_id.is_(None)` |
| `IS NOT NULL` | `Producto.categoria_id.is_not(None)` |
| `LIKE` / `ILIKE` | `.like("Mo%")` / `.ilike("%mo%")` |
| Empieza / contiene | `.startswith("Mo")` / `.contains("ous")` |
| `BETWEEN` | `Producto.precio.between(100, 500)` |
| `AND` / `OR` / `NOT` | `and_(...)`, `or_(...)`, `not_(...)` |
| Relación 1-N tiene algún... | `Cliente.pedidos.any()` |
| Relación N-1 cumple... | `Producto.categoria.has(Categoria.nombre == "Periféricos")` |

### 7.2 Ordenar, paginar, distinct

```python
stmt = (
    select(Producto)
    .where(Producto.stock > 0)
    .order_by(Producto.precio.desc(), Producto.nombre)
    .offset(20)
    .limit(10)
)

nombres = session.scalars(select(Producto.nombre).distinct()).all()
```

### 7.3 Filtros dinámicos (típico de un endpoint con filtros opcionales)

```python
def buscar_productos(
    session: Session,
    texto: str | None = None,
    precio_min: Decimal | None = None,
    precio_max: Decimal | None = None,
    categoria_id: int | None = None,
):
    stmt = select(Producto)
    if texto:
        stmt = stmt.where(Producto.nombre.ilike(f"%{texto}%"))
    if precio_min is not None:
        stmt = stmt.where(Producto.precio >= precio_min)
    if precio_max is not None:
        stmt = stmt.where(Producto.precio <= precio_max)
    if categoria_id is not None:
        stmt = stmt.where(Producto.categoria_id == categoria_id)
    return session.scalars(stmt.order_by(Producto.nombre)).all()
```

Los parámetros siempre viajan como **bind parameters**, así que el `f"%{texto}%"` no abre una inyección SQL.

### 7.4 Ordenamiento dinámico seguro

```python
COLUMNAS_ORDEN = {
    "nombre": Producto.nombre,
    "precio": Producto.precio,
    "stock": Producto.stock,
}


def ordenar(stmt, campo: str, direccion: str):
    columna = COLUMNAS_ORDEN.get(campo, Producto.id)    # lista blanca
    return stmt.order_by(columna.desc() if direccion == "desc" else columna.asc())
```

### 7.5 Joins

```python
# Join usando la relación (SQLAlchemy deduce el ON)
stmt = (
    select(Producto)
    .join(Producto.categoria)
    .where(Categoria.nombre == "Periféricos")
)

# Left join
stmt = select(Producto).outerjoin(Producto.categoria)

# Seleccionar columnas de varias tablas
stmt = select(Producto.nombre, Categoria.nombre.label("categoria")).join(Producto.categoria)
for fila in session.execute(stmt):
    print(fila.nombre, fila.categoria)

# Join con condición explícita
stmt = select(Pedido).join(Cliente, Pedido.cliente_id == Cliente.id)

# Self-join o tabla repetida: aliased
from sqlalchemy.orm import aliased
CatPadre = aliased(Categoria)
```

### 7.6 Agregaciones y GROUP BY

```python
from sqlalchemy import func

# Total vendido por producto
stmt = (
    select(
        Producto.nombre,
        func.sum(LineaPedido.cantidad).label("unidades"),
        func.sum(LineaPedido.cantidad * LineaPedido.precio_unitario).label("ingreso"),
    )
    .join(LineaPedido, LineaPedido.producto_id == Producto.id)
    .group_by(Producto.id, Producto.nombre)
    .having(func.sum(LineaPedido.cantidad) > 0)
    .order_by(func.sum(LineaPedido.cantidad).desc())
)
for fila in session.execute(stmt):
    print(fila.nombre, fila.unidades, fila.ingreso)

# Conteo con filtro
total_activos = session.scalar(
    select(func.count()).select_from(Producto).where(Producto.stock > 0)
)
```

| Función | Uso |
|---|---|
| `func.count()` | `COUNT(*)` |
| `func.count(Modelo.col.distinct())` | `COUNT(DISTINCT col)` |
| `func.sum`, `func.avg`, `func.min`, `func.max` | Agregados |
| `func.coalesce(x, 0)` | Valor por defecto si NULL |
| `func.lower`, `func.upper`, `func.length` | Funciones de texto |
| `func.now()`, `func.date(...)` | Fechas |
| `case((cond, valor), else_=otro)` | `CASE WHEN` |
| `cast(col, Integer)` | `CAST` |

### 7.7 Subconsultas y EXISTS

```python
# Subconsulta escalar: número de pedidos por cliente como columna
num_pedidos = (
    select(func.count(Pedido.id))
    .where(Pedido.cliente_id == Cliente.id)
    .correlate(Cliente)
    .scalar_subquery()
)
stmt = select(Cliente.nombre, num_pedidos.label("pedidos"))

# Subconsulta como tabla derivada
ventas = (
    select(LineaPedido.producto_id, func.sum(LineaPedido.cantidad).label("vendidos"))
    .group_by(LineaPedido.producto_id)
    .subquery()
)
stmt = (
    select(Producto.nombre, func.coalesce(ventas.c.vendidos, 0).label("vendidos"))
    .outerjoin(ventas, ventas.c.producto_id == Producto.id)
)

# EXISTS: clientes que tienen al menos un pedido
stmt = select(Cliente).where(Cliente.pedidos.any())

# NOT EXISTS: productos que nunca se vendieron
stmt = select(Producto).where(~Producto.lineas.any())
```

### 7.8 CTE (WITH)

```python
ventas_cte = (
    select(Pedido.cliente_id, func.count(Pedido.id).label("n"))
    .group_by(Pedido.cliente_id)
    .cte("ventas_por_cliente")
)
stmt = (
    select(Cliente.nombre, ventas_cte.c.n)
    .join(ventas_cte, ventas_cte.c.cliente_id == Cliente.id)
    .where(ventas_cte.c.n >= 1)
)
```

### 7.9 Paginación con total

```python
def paginar(session: Session, stmt, pagina: int, tamano: int) -> dict:
    total = session.scalar(select(func.count()).select_from(stmt.order_by(None).subquery()))
    items = session.scalars(stmt.offset((pagina - 1) * tamano).limit(tamano)).all()
    return {"items": items, "total": total, "pagina": pagina, "tamano": tamano}
```

### 7.10 SQL crudo cuando haga falta

```python
from sqlalchemy import text

filas = session.execute(
    text("SELECT nombre, precio FROM productos WHERE precio > :minimo"),
    {"minimo": 500},
).all()

# Mapear SQL crudo a objetos ORM
stmt = select(Producto).from_statement(text("SELECT * FROM productos WHERE stock > 0"))
productos = session.scalars(stmt).all()
```

> Siempre con parámetros `:nombre`. Nunca concatenes valores en el string.

---

## 8. Nivel 2 — Intermedio: carga de relaciones y el problema N+1

### 8.1 El problema

```python
pedidos = session.scalars(select(Pedido)).all()       # 1 consulta
for p in pedidos:
    print(p.cliente.nombre)                           # +1 consulta POR pedido (lazy load)
```

Con 100 pedidos son 101 consultas. Es el error de rendimiento más común con cualquier ORM (el mismo `LAZY` de Hibernate).

### 8.2 Estrategias de carga

```python
from sqlalchemy.orm import joinedload, raiseload, selectinload

# selectinload: 1 consulta extra con WHERE id IN (...). Ideal para colecciones (1-N, N-M)
stmt = select(Cliente).options(selectinload(Cliente.pedidos))

# joinedload: LEFT JOIN en la misma consulta. Ideal para N-1 y 1-1
stmt = select(Pedido).options(joinedload(Pedido.cliente))

# Encadenar niveles: pedidos -> líneas -> producto
stmt = select(Pedido).options(
    joinedload(Pedido.cliente),
    selectinload(Pedido.lineas).joinedload(LineaPedido.producto),
)
pedidos = session.scalars(stmt).all()

# joinedload sobre una colección exige .unique() (el JOIN duplica filas del padre)
clientes = session.scalars(
    select(Cliente).options(joinedload(Cliente.pedidos))
).unique().all()
```

| Estrategia | SQL generado | Mejor para | Cuidado |
|---|---|---|---|
| `lazyload` (default) | 1 consulta al acceder | Relaciones que casi nunca usas | N+1 en bucles |
| `selectinload` | 2ª consulta `WHERE fk IN (...)` | Colecciones 1-N y N-M | Listas muy largas de IDs se parten en lotes |
| `joinedload` | `LEFT JOIN` | N-1 y 1-1 | Con colecciones multiplica filas, requiere `.unique()` |
| `subqueryload` | 2ª consulta con subquery | Legado | Prefiere `selectinload` |
| `raiseload` | Ninguno: lanza error si accedes | Detectar N+1 en desarrollo | |
| `noload` | Ninguno: la relación queda vacía | Casos muy específicos | |

### 8.3 Estrategia por defecto en el modelo

```python
class PedidoConDefaults(Base):
    __tablename__ = "pedidos_defaults"
    id: Mapped[int] = mapped_column(primary_key=True)
    cliente_id: Mapped[int] = mapped_column(ForeignKey("clientes.id"))

    cliente: Mapped["Cliente"] = relationship(lazy="joined")    # siempre con JOIN
```

Valores de `lazy`: `"select"` (default), `"selectin"`, `"joined"`, `"raise"`, `"raise_on_sql"`, `"noload"`, `"write_only"`, `"dynamic"`.

> Recomendación pro: deja `lazy="raise"` en las relaciones y carga explícitamente con `options(...)` en cada consulta. Así un N+1 se convierte en un error visible en lugar de una lentitud silenciosa. Es **obligatorio** en async, donde el lazy load implícito no está permitido (sección 11).

### 8.4 Colecciones enormes: `write_only`

Para relaciones con miles de hijos que nunca quieres cargar completas:

```python
from sqlalchemy.orm import WriteOnlyMapped


class ClienteGrande(Base):
    __tablename__ = "clientes_grandes"
    id: Mapped[int] = mapped_column(primary_key=True)
    eventos: WriteOnlyMapped["Evento"] = relationship(cascade="all, delete-orphan")


class Evento(Base):
    __tablename__ = "eventos"
    id: Mapped[int] = mapped_column(primary_key=True)
    cliente_id: Mapped[int] = mapped_column(ForeignKey("clientes_grandes.id"))
    tipo: Mapped[str] = mapped_column(String(30))


# Uso:
# c.eventos.add(Evento(tipo="login"))
# ultimos = session.scalars(c.eventos.select().order_by(Evento.id.desc()).limit(10)).all()
```

### 8.5 Serializar relaciones con Pydantic en FastAPI

```python
from decimal import Decimal
from pydantic import BaseModel, ConfigDict


class ORMModel(BaseModel):
    model_config = ConfigDict(from_attributes=True)


class ProductoMini(ORMModel):
    id: int
    nombre: str


class LineaOut(ORMModel):
    producto: ProductoMini
    cantidad: int
    precio_unitario: Decimal
    subtotal: Decimal          # @property del modelo ORM


class ClienteMini(ORMModel):
    id: int
    nombre: str


class PedidoOut(ORMModel):
    id: int
    cliente: ClienteMini
    lineas: list[LineaOut]
    total: Decimal             # @property del modelo ORM


# En el endpoint, carga TODO lo que el schema va a leer:
# @app.get("/pedidos/{pedido_id}", response_model=PedidoOut)
# def obtener_pedido(pedido_id: int, db: DB):
#     stmt = (
#         select(Pedido)
#         .where(Pedido.id == pedido_id)
#         .options(
#             joinedload(Pedido.cliente),
#             selectinload(Pedido.lineas).joinedload(LineaPedido.producto),
#         )
#     )
#     pedido = db.scalars(stmt).one_or_none()
#     if pedido is None:
#         raise HTTPException(404, "Pedido no encontrado")
#     return pedido
```

> Regla: **el schema de respuesta define qué relaciones hay que cargar**. Si el `response_model` lee `pedido.lineas[].producto`, la consulta debe traerlos con `options()`.

---

## 9. Nivel 3 — Avanzado: migraciones con Alembic

`create_all` sirve para prototipos. En cualquier proyecto real usa **Alembic** (el equivalente de Flyway o Liquibase).

### 9.1 Convención de nombres (hazlo antes de la primera migración)

Sin nombres explícitos, cada BD inventa los suyos para índices y constraints, y luego Alembic no puede borrarlos ni renombrarlos de forma fiable.

```python
# app/db/base.py
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)
```

### 9.2 Inicializar

```bash
alembic init migrations            # proyecto sync
alembic init -t async migrations   # proyecto async (sección 11)
```

Edita `migrations/env.py` para que use tus modelos y tu configuración:

```python
# migrations/env.py (fragmento relevante)
from app.core.config import get_settings
from app.db.base import Base
import app.models  # noqa: F401  <- importa TODOS los modelos para que se registren

config = context.config
config.set_main_option("sqlalchemy.url", str(get_settings().database_url))
target_metadata = Base.metadata
```

En `context.configure(...)` añade estas opciones para que el autogenerate detecte más cambios:

```python
context.configure(
    connection=connection,
    target_metadata=target_metadata,
    compare_type=True,              # detecta cambios de tipo (String(50) -> String(100))
    render_as_batch=True,           # necesario en SQLite para ALTER TABLE
)
```

### 9.3 Flujo de trabajo

```bash
# 1. Cambias un modelo (agregas una columna, etc.)
# 2. Generas la migración
alembic revision --autogenerate -m "agregar sku a productos"

# 3. REVISAS el archivo generado en migrations/versions/
# 4. Aplicas
alembic upgrade head
```

| Comando | Qué hace |
|---|---|
| `alembic revision --autogenerate -m "..."` | Compara modelos vs BD y genera migración |
| `alembic revision -m "..."` | Migración vacía para escribir a mano |
| `alembic upgrade head` | Aplica todas las pendientes |
| `alembic upgrade +1` | Aplica la siguiente |
| `alembic downgrade -1` | Revierte la última |
| `alembic downgrade base` | Revierte todo |
| `alembic current` | Revisión actual de la BD |
| `alembic history --verbose` | Historial |
| `alembic upgrade head --sql` | Genera el SQL sin ejecutarlo (para DBAs) |
| `alembic stamp head` | Marca la BD como actualizada sin ejecutar nada |

### 9.4 Migración típica, revisada a mano

```python
"""agregar sku a productos"""
from alembic import op
import sqlalchemy as sa

revision = "a1b2c3d4e5f6"
down_revision = "f6e5d4c3b2a1"


def upgrade() -> None:
    # 1. Agregar como nullable para no romper filas existentes
    op.add_column("productos", sa.Column("sku", sa.String(20), nullable=True))
    # 2. Rellenar datos (migración de datos)
    op.execute("UPDATE productos SET sku = 'SKU-' || id WHERE sku IS NULL")
    # 3. Ahora sí, NOT NULL + índice único
    with op.batch_alter_table("productos") as batch:
        batch.alter_column("sku", nullable=False)
        batch.create_unique_constraint("uq_productos_sku", ["sku"])


def downgrade() -> None:
    with op.batch_alter_table("productos") as batch:
        batch.drop_constraint("uq_productos_sku", type_="unique")
        batch.drop_column("sku")
```

### 9.5 Lo que el autogenerate NO detecta bien

| Cambio | Qué hacer |
|---|---|
| Renombrar tabla o columna | Lo ve como borrar + crear: cámbialo a `op.rename_table` / `alter_column(new_column_name=...)` |
| Cambios en `server_default` | Activa `compare_server_default=True` y revisa |
| Valores de un `ENUM` nativo | Escribe la migración a mano |
| Migraciones de datos | Siempre a mano (`op.execute` o `op.bulk_insert`) |
| Vistas, triggers, funciones | A mano |

> Nunca edites una migración que ya se aplicó en otro entorno; crea una nueva.

---

## 10. Nivel 3 — Avanzado: transacciones

### 10.1 Patrones básicos

```python
# Patrón 1: commit explícito
with SessionLocal() as session:
    session.add(Producto(nombre="Monitor", precio=Decimal("3999")))
    session.commit()

# Patrón 2: begin() -> commit automático al salir, rollback si hay excepción
with SessionLocal() as session, session.begin():
    session.add(Producto(nombre="Webcam", precio=Decimal("899")))

# Patrón 3: equivalente en una línea
with SessionLocal.begin() as session:
    session.add(Producto(nombre="Micrófono", precio=Decimal("1299")))
```

### 10.2 Rollback manual

```python
from sqlalchemy.exc import IntegrityError

with SessionLocal() as session:
    try:
        session.add(Cliente(nombre="Repetido", email="gil@example.com"))
        session.commit()
    except IntegrityError:
        session.rollback()          # la sesión queda usable de nuevo
        print("Email duplicado")
```

### 10.3 Savepoints (transacciones anidadas)

```python
with SessionLocal() as session, session.begin():
    session.add(Categoria(nombre="Audio"))

    try:
        with session.begin_nested():         # SAVEPOINT
            session.add(Categoria(nombre="Audio"))   # duplicado -> falla
    except IntegrityError:
        pass                                 # solo se revierte el savepoint

    session.add(Categoria(nombre="Video"))   # la transacción externa sigue viva
# COMMIT: se guardan "Audio" y "Video"
```

### 10.4 Una transacción por request en FastAPI

Variante de `get_db` que hace commit automático si el endpoint terminó bien y rollback si lanzó excepción. Los endpoints ya no llaman `commit()`, solo `flush()` cuando necesitan el `id`:

```python
def get_db_tx() -> Generator[Session, None, None]:
    with SessionLocal() as session:
        try:
            yield session
            session.commit()
        except Exception:
            session.rollback()
            raise


# scope="function": el commit ocurre ANTES de enviar la respuesta (FastAPI 0.121+)
DBTx = Annotated[Session, Depends(get_db_tx, scope="function")]


@app.post("/clientes/{cliente_id}/pedidos", status_code=201)
def crear_pedido(cliente_id: int, items: dict[int, int], db: DBTx):
    """items: {producto_id: cantidad}"""
    pedido = Pedido(cliente_id=cliente_id)
    db.add(pedido)
    for producto_id, cantidad in items.items():
        producto = db.get(Producto, producto_id, with_for_update=True)  # bloquea la fila
        if producto is None or producto.stock < cantidad:
            raise HTTPException(409, f"Stock insuficiente para {producto_id}")
        producto.stock -= cantidad
        pedido.lineas.append(
            LineaPedido(producto=producto, cantidad=cantidad, precio_unitario=producto.precio)
        )
    db.flush()                 # obtiene pedido.id sin hacer commit
    return {"pedido_id": pedido.id}
```

> Desde FastAPI 0.118, el código posterior al `yield` corre por defecto **después** de enviar la respuesta (`scope="request"`). Si el commit falla en ese punto, el cliente ya recibió un 201. Con `Depends(..., scope="function")` (FastAPI 0.121+) el commit se ejecuta al terminar el endpoint y **antes** de responder, así un error de commit se convierte en un 500. En versiones anteriores, haz el `commit()` explícito en el endpoint o en la capa de servicio.

### 10.5 Bloqueos

| Técnica | Código | Uso |
|---|---|---|
| Bloqueo pesimista | `select(Producto).where(...).with_for_update()` | Descontar stock, saldos |
| Sin esperar | `.with_for_update(nowait=True)` | Fallar de inmediato si está bloqueado |
| Saltar bloqueadas | `.with_for_update(skip_locked=True)` | Colas de trabajo en PostgreSQL |
| Bloqueo optimista | `version_id_col` (sección 12.4) | Edición concurrente desde formularios |

---

## 11. Nivel 3 — Avanzado: SQLAlchemy async con FastAPI

### 11.1 Configuración

```python
# app/db/session.py
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncAttrs, AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost:5432/tienda",
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
)

AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)


class Base(AsyncAttrs, DeclarativeBase):
    pass


async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

`expire_on_commit=False` es prácticamente **obligatorio** en async: si los atributos expiran tras el commit, al leerlos se intentaría una carga implícita, que en async lanza `MissingGreenlet`.

### 11.2 Crear tablas al iniciar (solo desarrollo)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI):
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    await engine.dispose()


app = FastAPI(lifespan=lifespan)
```

### 11.3 CRUD async

```python
from typing import Annotated
from fastapi import Depends, HTTPException, status
from sqlalchemy import func, select
from sqlalchemy.orm import selectinload

SessionDep = Annotated[AsyncSession, Depends(get_session)]


@app.post("/productos", response_model=ProductoOut, status_code=status.HTTP_201_CREATED)
async def crear(data: ProductoCreate, session: SessionDep):
    producto = ProductoORM(**data.model_dump())
    session.add(producto)                     # add NO es async
    await session.commit()
    await session.refresh(producto)
    return producto


@app.get("/productos", response_model=list[ProductoOut])
async def listar(session: SessionDep, skip: int = 0, limit: int = 20):
    result = await session.scalars(select(ProductoORM).offset(skip).limit(limit))
    return result.all()


@app.get("/productos/{producto_id}", response_model=ProductoOut)
async def obtener(producto_id: int, session: SessionDep):
    producto = await session.get(ProductoORM, producto_id)
    if producto is None:
        raise HTTPException(404, "Producto no encontrado")
    return producto


@app.delete("/productos/{producto_id}", status_code=status.HTTP_204_NO_CONTENT)
async def eliminar(producto_id: int, session: SessionDep):
    producto = await session.get(ProductoORM, producto_id)
    if producto is None:
        raise HTTPException(404, "Producto no encontrado")
    await session.delete(producto)            # delete SÍ es async
    await session.commit()
```

### 11.4 Sync vs async: qué cambia

| Operación | Sync | Async |
|---|---|---|
| Engine | `create_engine(...)` | `create_async_engine(...)` |
| Fábrica | `sessionmaker(...)` | `async_sessionmaker(...)` |
| Ejecutar | `session.execute(stmt)` | `await session.execute(stmt)` |
| Escalares | `session.scalars(stmt).all()` | `(await session.scalars(stmt)).all()` |
| Por PK | `session.get(M, id)` | `await session.get(M, id)` |
| Añadir | `session.add(obj)` | `session.add(obj)` (igual, no es I/O) |
| Borrar | `session.delete(obj)` | `await session.delete(obj)` |
| Commit / flush / refresh | `session.commit()` | `await session.commit()` |
| Lazy load implícito | Funciona | **Error** `MissingGreenlet` |
| Cargar relación ya obtenida | `obj.relacion` | `await obj.awaitable_attrs.relacion` (con `AsyncAttrs`) |
| DDL | `Base.metadata.create_all(engine)` | `await conn.run_sync(Base.metadata.create_all)` |

### 11.5 Relaciones en async

```python
# Correcto: cargar con options en la misma consulta
stmt = select(Cliente).options(selectinload(Cliente.pedidos)).where(Cliente.id == 1)
cliente = (await session.scalars(stmt)).one()
print(len(cliente.pedidos))          # ya está cargado

# Alternativa puntual con AsyncAttrs
cliente = await session.get(Cliente, 1)
pedidos = await cliente.awaitable_attrs.pedidos

# Incorrecto: dispara lazy load -> MissingGreenlet
# cliente = await session.get(Cliente, 1); cliente.pedidos
```

### 11.6 ¿Sync o async?

| Elige sync si... | Elige async si... |
|---|---|
| El equipo es nuevo en SQLAlchemy | Muchas peticiones concurrentes con I/O (otras APIs, websockets) |
| Mucho código síncrono existente | Ya usas `httpx.AsyncClient` y otras librerías async |
| CPU-bound o tráfico moderado | Quieres evitar el límite del threadpool de FastAPI |

FastAPI ejecuta los endpoints `def` en un threadpool, así que la versión sync **no bloquea** el servidor. Lo que nunca debes hacer es usar una `Session` sync dentro de un `async def`.

---

## 12. Nivel 3 — Avanzado: técnicas de modelado

### 12.1 `hybrid_property`: un atributo que también sirve en consultas

```python
from sqlalchemy.ext.hybrid import hybrid_property


class Empleado(Base):
    __tablename__ = "empleados"
    id: Mapped[int] = mapped_column(primary_key=True)
    nombre: Mapped[str] = mapped_column(String(50))
    apellido: Mapped[str] = mapped_column(String(50))
    salario_base: Mapped[Decimal] = mapped_column(Numeric(10, 2))
    bono: Mapped[Decimal] = mapped_column(Numeric(10, 2), default=Decimal("0"))

    @hybrid_property
    def nombre_completo(self) -> str:
        return f"{self.nombre} {self.apellido}"

    @nombre_completo.inplace.expression
    @classmethod
    def _nombre_completo_expr(cls):
        return cls.nombre + " " + cls.apellido

    @hybrid_property
    def ingreso_total(self) -> Decimal:
        return self.salario_base + self.bono


# En Python:  empleado.ingreso_total
# En SQL:     select(Empleado).where(Empleado.ingreso_total > 20000)
#             select(Empleado).order_by(Empleado.nombre_completo)
```

La expresión simple (`salario_base + bono`) funciona igual en Python y en SQL; cuando difieren (concatenación de strings), defines la versión SQL con `.inplace.expression`.

### 12.2 `column_property`: columna calculada con subconsulta

```python
from sqlalchemy.orm import column_property


class CategoriaConConteo(Base):
    __tablename__ = "categorias_conteo"
    id: Mapped[int] = mapped_column(primary_key=True)
    nombre: Mapped[str] = mapped_column(String(80))


class ProductoConteo(Base):
    __tablename__ = "productos_conteo"
    id: Mapped[int] = mapped_column(primary_key=True)
    categoria_id: Mapped[int] = mapped_column(ForeignKey("categorias_conteo.id"))


CategoriaConConteo.num_productos = column_property(
    select(func.count(ProductoConteo.id))
    .where(ProductoConteo.categoria_id == CategoriaConConteo.id)
    .correlate_except(ProductoConteo)
    .scalar_subquery(),
    deferred=True,     # solo se calcula si lo pides con undefer()
)
```

### 12.3 Soft delete global

```python
from datetime import datetime, timezone
from sqlalchemy import event
from sqlalchemy.orm import ORMExecuteState, Session, with_loader_criteria


class SoftDeleteMixin:
    eliminado_en: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), default=None)

    def eliminar_logico(self) -> None:
        self.eliminado_en = datetime.now(timezone.utc)


@event.listens_for(Session, "do_orm_execute")
def _filtrar_eliminados(state: ORMExecuteState) -> None:
    if (
        state.is_select
        and not state.is_column_load
        and not state.is_relationship_load
        and not state.execution_options.get("incluir_eliminados", False)
    ):
        state.statement = state.statement.options(
            with_loader_criteria(
                SoftDeleteMixin,
                lambda cls: cls.eliminado_en.is_(None),
                include_aliases=True,
            )
        )


class Nota(SoftDeleteMixin, Base):
    __tablename__ = "notas"
    id: Mapped[int] = mapped_column(primary_key=True)
    texto: Mapped[str] = mapped_column(String(200))


# Todas las consultas a Nota excluyen las eliminadas automáticamente.
# Para incluirlas:
# session.scalars(select(Nota).execution_options(incluir_eliminados=True))
```

### 12.4 Bloqueo optimista (`@Version` de JPA)

```python
from sqlalchemy.orm.exc import StaleDataError


class Documento(Base):
    __tablename__ = "documentos"
    id: Mapped[int] = mapped_column(primary_key=True)
    contenido: Mapped[str] = mapped_column(String(2000))
    version: Mapped[int] = mapped_column(nullable=False)

    __mapper_args__ = {"version_id_col": version}


# Cada UPDATE incluye "WHERE id = ? AND version = ?" e incrementa la versión.
# Si otro usuario guardó antes, se lanza StaleDataError -> respóndelo como 409 Conflict.
```

### 12.5 Columnas JSON

```python
from sqlalchemy import JSON
from sqlalchemy.ext.mutable import MutableDict


class Configuracion(Base):
    __tablename__ = "configuraciones"
    id: Mapped[int] = mapped_column(primary_key=True)
    # MutableDict hace que config.datos["x"] = 1 se detecte como cambio
    datos: Mapped[dict] = mapped_column(MutableDict.as_mutable(JSON), default=dict)


# En PostgreSQL usa JSONB (indexable):
# from sqlalchemy.dialects.postgresql import JSONB
# datos: Mapped[dict] = mapped_column(MutableDict.as_mutable(JSONB), default=dict)
# Consulta: select(Configuracion).where(Configuracion.datos["tema"].astext == "oscuro")
```

### 12.6 Herencia

| Estrategia | Tablas | Equivalente JPA |
|---|---|---|
| Single table | 1 tabla con columna discriminadora | `SINGLE_TABLE` |
| Joined | 1 tabla base + 1 por subclase | `JOINED` |
| Concrete | 1 tabla completa por subclase | `TABLE_PER_CLASS` |

Ejemplo single table:

```python
class Pago(Base):
    __tablename__ = "pagos"
    id: Mapped[int] = mapped_column(primary_key=True)
    tipo: Mapped[str] = mapped_column(String(20))
    monto: Mapped[Decimal] = mapped_column(Numeric(10, 2))

    __mapper_args__ = {"polymorphic_on": "tipo", "polymorphic_identity": "pago"}


class PagoTarjeta(Pago):
    ultimos4: Mapped[str | None] = mapped_column(String(4))
    __mapper_args__ = {"polymorphic_identity": "tarjeta"}


class PagoOxxo(Pago):
    referencia: Mapped[str | None] = mapped_column(String(30))
    __mapper_args__ = {"polymorphic_identity": "oxxo"}


# select(Pago) devuelve instancias de PagoTarjeta / PagoOxxo según 'tipo'
```

Encaja directamente con las uniones discriminadas de Pydantic (sección 6.3 de `pydantic.md`).

---

## 13. Nivel 4 — Pro: arquitectura y repositorios

### 13.1 Estructura de proyecto

```
app/
├── main.py                    # FastAPI, lifespan, routers, handlers
├── core/
│   ├── config.py              # Settings (pydantic-settings)
│   └── exceptions.py          # NoEncontrado, Conflicto, handlers
├── db/
│   ├── base.py                # Base + naming convention + mixins
│   └── session.py             # engine, SessionLocal, get_db
├── models/                    # ORM (≈ @Entity)
│   ├── __init__.py            # importa todos los modelos (para Alembic)
│   ├── cliente.py
│   └── producto.py
├── schemas/                   # Pydantic (≈ DTOs)
├── repositories/              # acceso a datos (≈ JpaRepository)
│   ├── base.py
│   └── producto_repo.py
├── services/                  # reglas de negocio + transacción (≈ @Service @Transactional)
└── api/v1/                    # routers (≈ @RestController)
migrations/                    # Alembic
tests/
```

| Capa | Responsabilidad | NO debe |
|---|---|---|
| Router | HTTP: parámetros, status codes, `response_model` | Contener consultas SQL |
| Service | Reglas de negocio, orquestar repositorios, commit | Conocer `Request` ni `HTTPException` |
| Repository | Consultas a la BD | Hacer commit ni tener reglas de negocio |
| Model | Estructura de tablas y relaciones | Validar entradas de la API |
| Schema | Contrato de la API | Saber de la BD |

### 13.2 Repositorio genérico

```python
# app/repositories/base.py
from collections.abc import Sequence
from typing import Any, Generic, TypeVar

from sqlalchemy import Select, func, select
from sqlalchemy.orm import Session

from app.db.base import Base

ModelT = TypeVar("ModelT", bound=Base)


class Repositorio(Generic[ModelT]):
    modelo: type[ModelT]

    def __init__(self, session: Session):
        self.session = session

    def obtener(self, id_: Any) -> ModelT | None:
        return self.session.get(self.modelo, id_)

    def listar(self, *, offset: int = 0, limit: int = 20, stmt: Select | None = None) -> Sequence[ModelT]:
        stmt = stmt if stmt is not None else select(self.modelo)
        return self.session.scalars(stmt.offset(offset).limit(limit)).all()

    def contar(self, stmt: Select | None = None) -> int:
        stmt = stmt if stmt is not None else select(self.modelo)
        return self.session.scalar(select(func.count()).select_from(stmt.order_by(None).subquery())) or 0

    def agregar(self, entidad: ModelT) -> ModelT:
        self.session.add(entidad)
        self.session.flush()
        return entidad

    def actualizar(self, entidad: ModelT, cambios: dict[str, Any]) -> ModelT:
        for campo, valor in cambios.items():
            setattr(entidad, campo, valor)
        self.session.flush()
        return entidad

    def eliminar(self, entidad: ModelT) -> None:
        self.session.delete(entidad)
        self.session.flush()
```

```python
# app/repositories/producto_repo.py
from sqlalchemy import select
from sqlalchemy.orm import selectinload

from app.models import Producto
from app.repositories.base import Repositorio


class ProductoRepositorio(Repositorio[Producto]):
    modelo = Producto

    def por_nombre(self, nombre: str) -> Producto | None:
        return self.session.scalars(select(Producto).where(Producto.nombre == nombre)).first()

    def con_etiquetas(self, producto_id: int) -> Producto | None:
        stmt = (
            select(Producto)
            .where(Producto.id == producto_id)
            .options(selectinload(Producto.etiquetas), selectinload(Producto.categoria))
        )
        return self.session.scalars(stmt).one_or_none()
```

### 13.3 Servicio con excepciones de dominio

```python
# app/core/exceptions.py
class ErrorDominio(Exception):
    status_code = 400

    def __init__(self, mensaje: str):
        self.mensaje = mensaje


class NoEncontrado(ErrorDominio):
    status_code = 404


class Conflicto(ErrorDominio):
    status_code = 409
```

```python
# app/services/producto_service.py
from typing import Annotated
from fastapi import Depends
from sqlalchemy.orm import Session

from app.core.exceptions import Conflicto, NoEncontrado
from app.db.session import get_db
from app.models import Producto
from app.repositories.producto_repo import ProductoRepositorio
from app.schemas.producto import ProductoCreate, ProductoUpdate


class ProductoService:
    def __init__(self, db: Annotated[Session, Depends(get_db)]):
        self.db = db
        self.repo = ProductoRepositorio(db)

    def obtener(self, producto_id: int) -> Producto:
        producto = self.repo.con_etiquetas(producto_id)
        if producto is None:
            raise NoEncontrado(f"Producto {producto_id} no existe")
        return producto

    def crear(self, data: ProductoCreate) -> Producto:
        if self.repo.por_nombre(data.nombre):
            raise Conflicto(f"Ya existe un producto llamado '{data.nombre}'")
        producto = self.repo.agregar(Producto(**data.model_dump()))
        self.db.commit()
        return producto

    def actualizar(self, producto_id: int, data: ProductoUpdate) -> Producto:
        producto = self.obtener(producto_id)
        self.repo.actualizar(producto, data.model_dump(exclude_unset=True))
        self.db.commit()
        return producto


ProductoServiceDep = Annotated[ProductoService, Depends()]
```

```python
# app/main.py (fragmento)
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from sqlalchemy.exc import IntegrityError

from app.core.exceptions import ErrorDominio

app = FastAPI()


@app.exception_handler(ErrorDominio)
async def error_dominio_handler(request: Request, exc: ErrorDominio):
    return JSONResponse(status_code=exc.status_code, content={"detail": exc.mensaje})


@app.exception_handler(IntegrityError)
async def integridad_handler(request: Request, exc: IntegrityError):
    # Red de seguridad: UNIQUE o FK violadas que el servicio no anticipó
    return JSONResponse(status_code=409, content={"detail": "Conflicto de integridad de datos"})
```

```python
# app/api/v1/productos.py
from fastapi import APIRouter, status
from app.schemas.producto import ProductoCreate, ProductoOut, ProductoUpdate
from app.services.producto_service import ProductoServiceDep

router = APIRouter(prefix="/productos", tags=["Productos"])


@router.get("/{producto_id}", response_model=ProductoOut)
def obtener(producto_id: int, service: ProductoServiceDep):
    return service.obtener(producto_id)


@router.post("", response_model=ProductoOut, status_code=status.HTTP_201_CREATED)
def crear(data: ProductoCreate, service: ProductoServiceDep):
    return service.crear(data)


@router.patch("/{producto_id}", response_model=ProductoOut)
def actualizar(producto_id: int, data: ProductoUpdate, service: ProductoServiceDep):
    return service.actualizar(producto_id, data)
```

---

## 14. Nivel 4 — Pro: testing

### 14.1 Fixture con rollback por test

Cada test corre dentro de una transacción que se revierte al final: la BD queda limpia sin recrear tablas, y los `commit()` del código se convierten en savepoints.

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine, event
from sqlalchemy.orm import Session
from sqlalchemy.pool import StaticPool

from app.db.base import Base
from app.db.session import get_db
from app.main import app

engine = create_engine(
    "sqlite://",
    connect_args={"check_same_thread": False},
    poolclass=StaticPool,
)


@event.listens_for(engine, "connect")
def _fk_sqlite(dbapi_conn, _):
    dbapi_conn.execute("PRAGMA foreign_keys=ON")


@pytest.fixture(scope="session", autouse=True)
def _crear_esquema():
    Base.metadata.create_all(engine)
    yield
    Base.metadata.drop_all(engine)


@pytest.fixture
def db():
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection, join_transaction_mode="create_savepoint")
    try:
        yield session
    finally:
        session.close()
        transaction.rollback()
        connection.close()


@pytest.fixture
def client(db):
    app.dependency_overrides[get_db] = lambda: db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

> Para tests más fieles a producción usa la misma BD que en producción (PostgreSQL) levantada con Docker o `testcontainers`. SQLite no se comporta igual en tipos, bloqueos ni `ILIKE`.

### 14.2 Tests

```python
# tests/test_productos.py
from decimal import Decimal
from app.models import Producto


def test_crear_producto(client):
    r = client.post("/productos", json={"nombre": "Mouse", "precio": 299, "stock": 5})
    assert r.status_code == 201
    assert r.json()["nombre"] == "Mouse"


def test_nombre_duplicado_devuelve_409(client, db):
    db.add(Producto(nombre="Mouse", precio=Decimal("299")))
    db.flush()
    r = client.post("/productos", json={"nombre": "Mouse", "precio": 100})
    assert r.status_code == 409


def test_repositorio_por_nombre(db):
    from app.repositories.producto_repo import ProductoRepositorio
    repo = ProductoRepositorio(db)
    repo.agregar(Producto(nombre="Teclado", precio=Decimal("799")))
    assert repo.por_nombre("Teclado") is not None
    assert repo.por_nombre("Nada") is None
```

### 14.3 Contar consultas (detectar N+1 en tests)

```python
from contextlib import contextmanager
from sqlalchemy import event


@contextmanager
def contar_consultas(engine):
    contador = {"n": 0}

    def _antes(conn, cursor, statement, params, context, executemany):
        contador["n"] += 1

    event.listen(engine, "before_cursor_execute", _antes)
    try:
        yield contador
    finally:
        event.remove(engine, "before_cursor_execute", _antes)


def test_listado_pedidos_sin_n_mas_1(client):
    with contar_consultas(engine) as c:
        client.get("/pedidos")
    assert c["n"] <= 3
```

---

## 15. Nivel 4 — Pro: rendimiento y producción

### 15.1 Configuración del pool

```python
engine = create_engine(
    settings.database_url,
    pool_size=10,          # conexiones abiertas permanentemente
    max_overflow=20,       # extra temporales en picos
    pool_timeout=30,       # segundos esperando una conexión libre
    pool_recycle=1800,     # renovar conexiones cada 30 min (evita cortes del servidor/proxy)
    pool_pre_ping=True,    # verifica la conexión antes de usarla
    echo=False,            # nunca True en producción
)
```

| Situación | Ajuste |
|---|---|
| `QueuePool limit ... overflow ... reached` | Sube `pool_size`/`max_overflow` o revisa sesiones que no se cierran |
| Errores "server closed the connection" | `pool_pre_ping=True` y `pool_recycle` menor al timeout del servidor |
| Varios workers (`uvicorn --workers 4`) | Cada worker tiene su pool: total = workers × (pool_size + max_overflow) |
| PgBouncer en modo transacción | `poolclass=NullPool` y deja el pooling a PgBouncer |

### 15.2 Inserciones y actualizaciones masivas

```python
from sqlalchemy import insert, update

# INSERT masivo (executemany optimizado), sin crear objetos ORM
session.execute(
    insert(Producto),
    [{"nombre": f"Producto {i}", "precio": Decimal("10.00"), "stock": i} for i in range(10_000)],
)

# UPDATE masivo por llave primaria
session.execute(
    update(Producto),
    [{"id": 1, "stock": 100}, {"id": 2, "stock": 0}],
)
session.commit()

# INSERT ... RETURNING (PostgreSQL, SQLite 3.35+, MariaDB)
nuevos = session.scalars(
    insert(Producto).returning(Producto),
    [{"nombre": "A", "precio": Decimal("1")}, {"nombre": "B", "precio": Decimal("2")}],
).all()
```

Upsert en PostgreSQL:

```python
from sqlalchemy.dialects.postgresql import insert as pg_insert

stmt = pg_insert(Producto).values(nombre="Mouse", precio=Decimal("299"), stock=10)
stmt = stmt.on_conflict_do_update(
    index_elements=[Producto.nombre],
    set_={"precio": stmt.excluded.precio, "stock": Producto.stock + stmt.excluded.stock},
)
session.execute(stmt)
```

### 15.3 Leer solo lo necesario

```python
from sqlalchemy.orm import defer, load_only

# Solo algunas columnas del modelo
stmt = select(Producto).options(load_only(Producto.id, Producto.nombre))

# Excluir una columna pesada
stmt = select(Articulo).options(defer(Articulo.nombre))

# Sin objetos ORM: filas ligeras (más rápido para reportes)
filas = session.execute(select(Producto.id, Producto.nombre)).all()

# Procesar millones de filas en lotes sin cargar todo en memoria
for producto in session.scalars(
    select(Producto).execution_options(yield_per=1000)
):
    ...
```

### 15.4 Eventos útiles

```python
import logging
import time
from sqlalchemy import event
from sqlalchemy.engine import Engine

log = logging.getLogger("sql.lento")


# Activar llaves foráneas en SQLite
@event.listens_for(Engine, "connect")
def _sqlite_fk(dbapi_conn, connection_record):
    if dbapi_conn.__class__.__module__.startswith("sqlite3"):
        cursor = dbapi_conn.cursor()
        cursor.execute("PRAGMA foreign_keys=ON")
        cursor.close()


# Registrar consultas lentas
@event.listens_for(Engine, "before_cursor_execute")
def _inicio(conn, cursor, statement, params, context, executemany):
    conn.info.setdefault("t0", []).append(time.perf_counter())


@event.listens_for(Engine, "after_cursor_execute")
def _fin(conn, cursor, statement, params, context, executemany):
    duracion = time.perf_counter() - conn.info["t0"].pop()
    if duracion > 0.5:
        log.warning("Consulta lenta (%.2fs): %s", duracion, statement)
```

### 15.5 Checklist de producción

| Tema | Recomendación |
|---|---|
| Esquema | Solo Alembic; nunca `create_all` en producción |
| Constraints | Naming convention definida desde el día 1 |
| Índices | En todas las FK y columnas de filtros/orden frecuentes |
| Relaciones | `lazy="raise"` + `options()` explícitos |
| Sesiones | Una por request, siempre cerrada (`with` / dependencia `yield`) |
| Transacciones | Commit en una sola capa (servicio); repositorios solo `flush` |
| Concurrencia | `with_for_update` o `version_id_col` donde haya carreras |
| Secretos | URL de la BD desde variables de entorno (`SecretStr`) |
| Logs | `echo=False`; registrar consultas lentas con eventos |
| Pool | `pool_pre_ping=True`, `pool_recycle` < timeout del servidor |
| Dinero | `Numeric(10, 2)` + `Decimal`, nunca `float` |
| Fechas | `DateTime(timezone=True)` y guardar en UTC |
| Tests | Misma BD que producción + rollback por test |

---

## 16. Referencia: SQLAlchemy vs JPA/Hibernate

| Concepto | Spring Data JPA / Hibernate | SQLAlchemy 2.0 |
|---|---|---|
| Entidad | `@Entity @Table(name="productos")` | `class Producto(Base): __tablename__ = "productos"` |
| Id | `@Id @GeneratedValue(strategy = IDENTITY)` | `id: Mapped[int] = mapped_column(primary_key=True)` |
| Columna | `@Column(length=100, nullable=false)` | `mapped_column(String(100))` + `Mapped[str]` |
| Nullable | `@Column(nullable=true)` | `Mapped[str \| None]` |
| Enum | `@Enumerated(EnumType.STRING)` | `Mapped[MiEnum]` + `Enum(..., native_enum=False)` |
| `@ManyToOne` + `@JoinColumn` | Anotaciones | `ForeignKey(...)` + `relationship()` |
| `@OneToMany(mappedBy=...)` | Anotaciones | `relationship(back_populates=...)` |
| `@ManyToMany` + `@JoinTable` | Anotaciones | `relationship(secondary=tabla)` |
| `cascade = ALL, orphanRemoval = true` | Anotaciones | `cascade="all, delete-orphan"` |
| `FetchType.LAZY` / `EAGER` | Anotaciones | `lazy="select"` / `lazy="joined"` o `"selectin"` |
| `JOIN FETCH` / `@EntityGraph` | JPQL | `options(joinedload(...))` / `selectinload(...)` |
| `LazyInitializationException` | Sesión cerrada | `DetachedInstanceError` / `MissingGreenlet` (async) |
| `EntityManager` | Contexto de persistencia | `Session` |
| `@Transactional` | AOP | `with session.begin():` o commit en el servicio |
| `JpaRepository<T, ID>` | Interfaz generada | Repositorio genérico propio (sección 13.2) |
| Query methods (`findByNombre`) | Derivadas del nombre | Métodos explícitos con `select()` |
| `@Query("...")` JPQL | String | `select(...)` tipado o `text(...)` |
| Specifications / Criteria | API verbosa | `stmt = stmt.where(...)` encadenado |
| `Pageable` / `Page<T>` | Integrado | `offset` / `limit` + conteo (sección 7.9) |
| `@Version` | Optimistic locking | `__mapper_args__ = {"version_id_col": ...}` |
| `@CreationTimestamp` | Hibernate | `server_default=func.now()` |
| `@UpdateTimestamp` | Hibernate | `onupdate=func.now()` |
| `@Formula` | Hibernate | `column_property(...)` |
| `@Where(clause="deleted=false")` | Hibernate | `with_loader_criteria` (sección 12.3) |
| `@Inheritance` | 3 estrategias | `polymorphic_on` / `polymorphic_identity` |
| `@Lock(PESSIMISTIC_WRITE)` | Repositorio | `.with_for_update()` |
| Flyway / Liquibase | Migraciones | Alembic |
| `spring.jpa.show-sql=true` | Properties | `create_engine(..., echo=True)` |
| HikariCP | Pool | Pool integrado (`QueuePool`) |
| `@DataJpaTest` | Test con rollback | Fixture con savepoints (sección 14.1) |

---

## 17. Referencia: migración 1.x → 2.0

| Estilo 1.x (legado) | Estilo 2.0 |
|---|---|
| `Base = declarative_base()` | `class Base(DeclarativeBase): pass` |
| `nombre = Column(String(100), nullable=False)` | `nombre: Mapped[str] = mapped_column(String(100))` |
| `session.query(Producto).all()` | `session.scalars(select(Producto)).all()` |
| `session.query(Producto).get(1)` | `session.get(Producto, 1)` |
| `session.query(Producto).filter(...).first()` | `session.scalars(select(Producto).where(...)).first()` |
| `session.query(Producto).filter_by(nombre="x")` | `select(Producto).filter_by(nombre="x")` |
| `query.count()` | `session.scalar(select(func.count()).select_from(...))` |
| `query.update({...})` | `session.execute(update(Producto).where(...).values(...))` |
| `query.delete()` | `session.execute(delete(Producto).where(...))` |
| `select([Producto.id])` | `select(Producto.id)` |
| `engine.execute(...)` | `with engine.connect() as conn: conn.execute(...)` |
| `autocommit=True` | Eliminado; usa `session.begin()` |
| `relationship(..., backref="x")` | `relationship(..., back_populates="x")` en ambos lados |
| `lazy="dynamic"` | `WriteOnlyMapped` + `lazy="write_only"` |
| `session.bulk_insert_mappings(...)` | `session.execute(insert(Modelo), [dicts])` |

`session.query()` sigue funcionando en 2.0 como API legada, pero el código nuevo debería usar `select()`.

---

## 18. Cheatsheet

```python
from sqlalchemy import (
    ForeignKey, Numeric, String, DateTime, Enum, JSON, Table, Column,
    UniqueConstraint, CheckConstraint, Index,
    create_engine, select, insert, update, delete, func, and_, or_, text, event,
)
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, relationship, Session, sessionmaker,
    selectinload, joinedload, raiseload, load_only, aliased, column_property,
)
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine, AsyncAttrs
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.exc import IntegrityError, NoResultFound
```

| Necesito... | Código |
|---|---|
| Columna requerida | `x: Mapped[str] = mapped_column(String(50))` |
| Columna opcional | `x: Mapped[str \| None]` |
| FK | `cliente_id: Mapped[int] = mapped_column(ForeignKey("clientes.id"))` |
| N a 1 | `cliente: Mapped["Cliente"] = relationship(back_populates="pedidos")` |
| 1 a N | `pedidos: Mapped[list["Pedido"]] = relationship(back_populates="cliente")` |
| N a M | `relationship(secondary=tabla_asoc, back_populates=...)` |
| Por id | `session.get(M, id)` |
| Lista | `session.scalars(select(M).where(...)).all()` |
| Uno o None | `session.scalars(stmt).one_or_none()` |
| Contar | `session.scalar(select(func.count()).select_from(M))` |
| Filas con columnas | `session.execute(select(M.a, M.b)).all()` |
| Paginar | `.offset((p - 1) * n).limit(n)` |
| Cargar colección | `.options(selectinload(M.hijos))` |
| Cargar padre | `.options(joinedload(M.padre))` |
| Guardar | `session.add(obj); session.commit()` |
| Obtener id sin commit | `session.flush()` |
| Update masivo | `session.execute(update(M).where(...).values(...))` |
| Insert masivo | `session.execute(insert(M), [dicts])` |
| Transacción | `with session.begin(): ...` |
| Savepoint | `with session.begin_nested(): ...` |
| Bloquear fila | `select(M).where(...).with_for_update()` |
| Pydantic desde ORM | `ConfigDict(from_attributes=True)` |
| Nueva migración | `alembic revision --autogenerate -m "..."` |
| Aplicar migraciones | `alembic upgrade head` |
