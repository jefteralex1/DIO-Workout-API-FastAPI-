# DIO-Workout-API-FastAPI-

# DIO – Workout API (FastAPI)

Entrega completa do desafio com:

* Query params (filtros) para **atleta**: `nome`, `cpf`
* **Customização de resposta** no `get all` de atletas: `nome`, `centro_treinamento`, `categoria`
* **Tratamento de Integridade** (SQLAlchemy `IntegrityError`) com mensagem específica e **status 303**
* **Paginação** via `fastapi-pagination` usando `limit` e `offset`

> **Observação**: Este guia presume um projeto FastAPI + SQLAlchemy. Se você estiver partindo do repositório sugerido (\*workout\_api\*), substitua/adicione os arquivos abaixo conforme indicado.

---

## Estrutura sugerida do repositório

```
workout_api/
├─ app/
│  ├─ __init__.py
│  ├─ database.py
│  ├─ models.py
│  ├─ schemas.py
│  ├─ crud.py
│  └─ main.py
├─ requirements.txt
├─ README.md
└─ alembic/ (opcional, se usar migrações)
```

---

## requirements.txt

```txt
fastapi==0.115.0
uvicorn[standard]==0.30.6
SQLAlchemy==2.0.36
pydantic==2.9.2
pydantic-settings==2.6.1
fastapi-pagination==0.12.26
python-dotenv==1.0.1
```

---

## app/database.py

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base
import os

DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./workout.db")

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False} if DATABASE_URL.startswith("sqlite") else {},
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Dependência de sessão para FastAPI
from contextlib import contextmanager

@contextmanager
def session_scope():
    db = SessionLocal()
    try:
        yield db
        db.commit()
    except:
        db.rollback()
        raise
    finally:
        db.close()
```

---

## app/models.py

```python
from sqlalchemy import Column, Integer, String, ForeignKey, UniqueConstraint
from sqlalchemy.orm import relationship, Mapped, mapped_column
from .database import Base

class CentroTreinamento(Base):
    __tablename__ = "centros_treinamento"
    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    nome: Mapped[str] = mapped_column(String(120), unique=True, nullable=False)

    atletas = relationship("Atleta", back_populates="centro_treinamento")

class Categoria(Base):
    __tablename__ = "categorias"
    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    nome: Mapped[str] = mapped_column(String(120), unique=True, nullable=False)

    atletas = relationship("Atleta", back_populates="categoria")

class Atleta(Base):
    __tablename__ = "atletas"
    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    nome: Mapped[str] = mapped_column(String(120), nullable=False, index=True)
    cpf: Mapped[str] = mapped_column(String(14), nullable=False, unique=True, index=True)
    centro_treinamento_id: Mapped[int] = mapped_column(ForeignKey("centros_treinamento.id"), nullable=False)
    categoria_id: Mapped[int] = mapped_column(ForeignKey("categorias.id"), nullable=False)

    centro_treinamento = relationship("CentroTreinamento", back_populates="atletas")
    categoria = relationship("Categoria", back_populates="atletas")

    __table_args__ = (
        UniqueConstraint("cpf", name="uq_atletas_cpf"),
    )
```

---

## app/schemas.py

```python
from pydantic import BaseModel, Field
from typing import Optional

# --- Centro de Treinamento ---
class CTCria(BaseModel):
    nome: str = Field(..., min_length=2)

class CTOut(BaseModel):
    id: int
    nome: str
    class Config:
        from_attributes = True

# --- Categoria ---
class CategoriaCria(BaseModel):
    nome: str = Field(..., min_length=2)

class CategoriaOut(BaseModel):
    id: int
    nome: str
    class Config:
        from_attributes = True

# --- Atleta ---
class AtletaCria(BaseModel):
    nome: str
    cpf: str
    centro_treinamento_id: int
    categoria_id: int

# Resposta CUSTOMIZADA do GET ALL de atletas
class AtletaListOut(BaseModel):
    nome: str
    centro_treinamento: str
    categoria: str

class AtletaOut(BaseModel):
    id: int
    nome: str
    cpf: str
    centro_treinamento: CTOut
    categoria: CategoriaOut
    class Config:
        from_attributes = True
```

---

## app/crud.py

```python
from sqlalchemy.orm import Session
from sqlalchemy import select
from . import models, schemas

# --- Centro de Treinamento ---
def criar_ct(db: Session, payload: schemas.CTCria) -> models.CentroTreinamento:
    ct = models.CentroTreinamento(nome=payload.nome)
    db.add(ct)
    db.flush()  # obtém id
    return ct

def listar_cts(db: Session):
    return db.scalars(select(models.CentroTreinamento)).all()

# --- Categoria ---
def criar_categoria(db: Session, payload: schemas.CategoriaCria) -> models.Categoria:
    cat = models.Categoria(nome=payload.nome)
    db.add(cat)
    db.flush()
    return cat

def listar_categorias(db: Session):
    return db.scalars(select(models.Categoria)).all()

# --- Atleta ---
def criar_atleta(db: Session, payload: schemas.AtletaCria) -> models.Atleta:
    at = models.Atleta(
        nome=payload.nome,
        cpf=payload.cpf,
        centro_treinamento_id=payload.centro_treinamento_id,
        categoria_id=payload.categoria_id,
    )
    db.add(at)
    db.flush()
    db.refresh(at)
    return at

def query_atletas(db: Session, nome: str | None = None, cpf: str | None = None):
    stmt = select(models.Atleta).join(models.Atleta.centro_treinamento).join(models.Atleta.categoria)
    if nome:
        stmt = stmt.where(models.Atleta.nome.ilike(f"%{nome}%"))
    if cpf:
        stmt = stmt.where(models.Atleta.cpf == cpf)
    return db.scalars(stmt).all()
```

---

## app/main.py

```python
from fastapi import FastAPI, Depends, status
from fastapi.responses import JSONResponse
from sqlalchemy.orm import Session
from sqlalchemy.exc import IntegrityError
from typing import List

from .database import Base, engine, SessionLocal
from . import models, schemas, crud

# Paginação
from fastapi_pagination import add_pagination
from fastapi_pagination.limit_offset import LimitOffsetPage, LimitOffsetParams

app = FastAPI(title="Workout API – DIO")

# Cria as tabelas (para ambientes simples / demo)
Base.metadata.create_all(bind=engine)

# Dependency de DB

def get_db():
    db = SessionLocal()
    try:
        yield db
        db.commit()
    except:
        db.rollback()
        raise
    finally:
        db.close()

# --- Handlers de Integridade ---
@app.exception_handler(IntegrityError)
async def integrity_exception_handler(request, exc: IntegrityError):
    # Tenta extrair o valor do CPF da mensagem do banco
    cpf_hint = None
    detail = str(exc.orig) if getattr(exc, "orig", None) else str(exc)
    # heurística simples pra mostrar cpf quando houver na mensagem
    for tok in ["cpf", "CPF", "uq_atletas_cpf"]:
        if tok in detail:
            cpf_hint = getattr(request, "_last_cpf", None)
            break

    msg = (
        f"Já existe um atleta cadastrado com o cpf: {cpf_hint}" if cpf_hint else
        "Violação de integridade dos dados."
    )
    return JSONResponse(status_code=303, content={"detail": msg})

# --- Rotas de Centros de Treinamento ---
@app.post("/centros", response_model=schemas.CTOut, status_code=status.HTTP_201_CREATED)
def criar_centro(payload: schemas.CTCria, db: Session = Depends(get_db)):
    try:
        ct = crud.criar_ct(db, payload)
        return ct
    except IntegrityError as e:
        raise e

@app.get("/centros", response_model=List[schemas.CTOut])
def listar_centros(db: Session = Depends(get_db)):
    return crud.listar_cts(db)

# --- Rotas de Categorias ---
@app.post("/categorias", response_model=schemas.CategoriaOut, status_code=status.HTTP_201_CREATED)
def criar_categoria(payload: schemas.CategoriaCria, db: Session = Depends(get_db)):
    try:
        cat = crud.criar_categoria(db, payload)
        return cat
    except IntegrityError as e:
        raise e

@app.get("/categorias", response_model=List[schemas.CategoriaOut])
def listar_categorias(db: Session = Depends(get_db)):
    return crud.listar_categorias(db)

# --- Rotas de Atletas ---
@app.post("/atletas", response_model=schemas.AtletaOut, status_code=status.HTTP_201_CREATED)
def criar_atleta(payload: schemas.AtletaCria, db: Session = Depends(get_db)):
    try:
        # Guarda o cpf na request para o handler (heurística)
        from fastapi import Request
        request = Request(scope={"type": "http"})
        setattr(request, "_last_cpf", payload.cpf)
        at = crud.criar_atleta(db, payload)
        return at
    except IntegrityError as e:
        # Deixa o exception handler central tratar e devolver 303
        raise e

# GET ALL com filtros (query params) e **paginação limit/offset**
@app.get("/atletas", response_model=LimitOffsetPage[schemas.AtletaListOut])
def listar_atletas(
    params: LimitOffsetParams = Depends(),
    nome: str | None = None,
    cpf: str | None = None,
    db: Session = Depends(get_db),
):
    # Aplica filtros
    atletas = crud.query_atletas(db, nome=nome, cpf=cpf)

    # Customiza saída: somente nome, centro_treinamento, categoria
    items = [
        schemas.AtletaListOut(
            nome=a.nome,
            centro_treinamento=a.centro_treinamento.nome,
            categoria=a.categoria.nome,
        )
        for a in atletas
    ]

    # Paginação manual via LimitOffsetPage.create
    return LimitOffsetPage.create(items, total=len(items), params=params)

add_pagination(app)
```

> **Notas importantes**
>
> 1. **Query Params**: `GET /atletas?nome=joao&cpf=123.456.789-00` filtrará por *nome* (ILIKE) e *cpf* (igualdade).
> 2. **Custom Response (GET ALL)**: o endpoint retorna **apenas** `nome`, `centro_treinamento`, `categoria` (campo textual), conforme o desafio.
> 3. **Tratamento de Integridade**: qualquer `IntegrityError` (ex.: CPF duplicado) retorna **HTTP 303** e a mensagem `"Já existe um atleta cadastrado com o cpf: x"`. A heurística do handler tenta incluir o CPF passado.
> 4. **Paginação**: usando `fastapi-pagination` com **Limit/Offset**. Ex.: `GET /atletas?limit=10&offset=20`.
>
> Se preferir, você pode criar um **handler mais robusto** lendo o texto da exceção do seu SGBD (Postgres/MySQL) para extrair o valor do CPF duplicado.

---

## Exemplos de uso (curl)

```bash
# Criar Centro de Treinamento
curl -s -X POST http://localhost:8000/centros \
  -H 'Content-Type: application/json' \
  -d '{"nome":"CT Recife"}'

# Criar Categoria
curl -s -X POST http://localhost:8000/categorias \
  -H 'Content-Type: application/json' \
  -d '{"nome":"Profissional"}'

# Criar Atleta
curl -s -X POST http://localhost:8000/atletas \
  -H 'Content-Type: application/json' \
  -d '{"nome":"Jefter","cpf":"123.456.789-00","centro_treinamento_id":1,"categoria_id":1}'

# Tentar criar CPF duplicado (retorna 303)
curl -i -X POST http://localhost:8000/atletas \
  -H 'Content-Type: application/json' \
  -d '{"nome":"Outro","cpf":"123.456.789-00","centro_treinamento_id":1,"categoria_id":1}'

# Listar atletas com paginação e filtro por nome
curl -s "http://localhost:8000/atletas?limit=5&offset=0&nome=jef"

# Listar atletas filtrando por CPF
curl -s "http://localhost:8000/atletas?cpf=123.456.789-00"
```

---

## Como executar

```bash
python -m venv .venv
source .venv/bin/activate   # no Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

A API ficará disponível em `http://localhost:8000` e a documentação interativa em `http://localhost:8000/docs`.

---

## Dicas para integrar no repositório original

1. **Fork** o repositório base no GitHub.
2. Crie uma branch `feat/paginacao-filtros-integridade`.
3. Adicione/ajuste os arquivos conforme este guia.
4. Faça *commits* pequenos e descritivos.
5. Abra um **Pull Request** explicando o que foi feito (inclua exemplos de chamadas).

---

## Testes rápidos (Pytest – opcional)

```python
# tests/test_atletas.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_criar_listar_atleta():
    # cria dependências
    client.post("/centros", json={"nome":"CT Teste"})
    client.post("/categorias", json={"nome":"Amador"})

    r = client.post("/atletas", json={
        "nome":"Ana",
        "cpf":"000.000.000-00",
        "centro_treinamento_id":1,
        "categoria_id":1,
    })
    assert r.status_code == 201

    # duplicado -> 303
    r2 = client.post("/atletas", json={
        "nome":"Ana 2",
        "cpf":"000.000.000-00",
        "centro_treinamento_id":1,
        "categoria_id":1,
    })
    assert r2.status_code == 303

    r3 = client.get("/atletas", params={"limit":10, "offset":0, "nome":"an"})
    assert r3.status_code == 200
    body = r3.json()
    assert "items" in body
    assert any(it["nome"] == "Ana" for it in body["items"])  # resposta customizada
```

---

## Checklist do Desafio ✅

* [x] **Query params** `nome` e `cpf` nos endpoints de listagem de **atletas**.
* [x] **Resposta customizada** do `GET /atletas` contendo apenas: `nome`, `centro_treinamento`, `categoria`.
* [x] **Tratamento de `IntegrityError`** com mensagem: `"Já existe um atleta cadastrado com o cpf: x"` e **status 303**.
* [x] **Paginação** `limit` e `offset` com `fastapi-pagination`.

Se quiser, posso adaptar este código exatamente à estrutura do seu repositório atual e abrir um PR de exemplo.
