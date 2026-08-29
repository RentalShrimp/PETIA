# PETIA Command OS (primeira fatia) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Entregar o núcleo Command OS da PETIA para uma unidade: tutor/pet, agenda, OS de banho & tosa, copiloto WhatsApp com confirmação, PIX/link com baixa por webhook e timeline do pet.

**Architecture:** Um único caminho de escrita: tela e WhatsApp disparam o mesmo `Command` nomeado; o orquestrador só chama o agente; o handler persiste com SQL parametrizado, emite `DomainEvent` e grava `ComandoLog` (SQL visível no audit). LLM não redige SQL. Leitura vai a queries, não ao agente.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2 (`text()` + binds), Alembic, PostgreSQL 16, Pydantic v2, pytest, Jinja2 (telas da loja). Portas: `Agent`, `WhatsAppGateway`, `Psp` — fakes nos testes; clientes reais atrás da mesma porta.

**Spec:** `.cursor/docs/superpowers/specs/2026-08-29-petia-command-os-design.md`

## Global Constraints

- Pet é entidade de primeira classe, não um campo do cliente.
- Toda escrita é comando nomeado com `tenant_id`; operação do dia também com `unidade_id`.
- LLM só existe no chat; a tela não conversa para persistir.
- Padrão v1: nada grava no chat sem confirmação (“Confirmo o banho do Thor sábado 11:30?”).
- Orquestrador é obrigatório na chamada do agente; não é dono da regra de negócio.
- SQL só no handler, parametrizado; o modelo não redige SQL; audit mostra o SQL.
- Isolamento: consulta e comando sempre filtrados por tenant; agendamento/OS não atravessam unidade.
- `EmitirCobranca` e `BaixarPagamento` são idempotentes; webhook duplicado não duplica baixa.
- Reenvio de “sim” no chat não cria segundo banho no mesmo slot.
- Confirmação expirada não persiste.
- Agente caído: a tela da loja continua gravando pelos mesmos comandos.
- Pedido fora do catálogo é recusa explícita; agente não inventa horário, preço ou PIX.
- Cadastro 360 desta fatia é operacional (sem LTV, churn, campanhas).
- Agenda v1: por profissional, duração fixa por serviço (sem lista de espera, encaixe, recorrência, equipamentos).
- `RecomendarProduto` não movimenta estoque.
- `BaixarPagamento` não pede confirmação no chat (webhook).
- Uma unidade no v1; modelo já nasce Tenant → Unidade.
- Fora desta fatia: holding, estoque, planos, delivery, CRM de campanha, copiloto executivo, app do tutor, NFe.

---

## File structure

```text
pyproject.toml
docker-compose.yml
alembic.ini
alembic/env.py
alembic/versions/0001_command_os.py
src/petia/runtime/db.py
src/petia/runtime/app.py
src/petia/command_os/envelope.py
src/petia/command_os/catalog.py
src/petia/command_os/bus.py
src/petia/identity/tables.py
src/petia/identity/handlers.py
src/petia/identity/queries.py
src/petia/cadastro/tables.py
src/petia/cadastro/handlers.py
src/petia/cadastro/queries.py
src/petia/agenda/tables.py
src/petia/agenda/handlers.py
src/petia/agenda/queries.py
src/petia/os_banho/tables.py
src/petia/os_banho/handlers.py
src/petia/os_banho/queries.py
src/petia/timeline/tables.py
src/petia/timeline/projector.py
src/petia/cobranca/tables.py
src/petia/cobranca/handlers.py
src/petia/cobranca/psp.py
src/petia/copilot/confirmation.py
src/petia/copilot/orchestrator.py
src/petia/copilot/agent.py
src/petia/whatsapp/gateway.py
src/petia/web/routes_commands.py
src/petia/web/routes_queries.py
src/petia/web/routes_webhooks.py
src/petia/web/routes_shop.py
src/petia/web/templates/agenda.html
src/petia/web/templates/os.html
src/petia/web/templates/pet.html
tests/conftest.py
tests/test_command_bus.py
tests/test_cadastro.py
tests/test_agenda.py
tests/test_os.py
tests/test_timeline.py
tests/test_cobranca.py
tests/test_copilot.py
tests/test_isolamento.py
tests/test_shop_loop.py
```

Domínios colocalizam tabela + handler + query. `command_os` é o único write-path. `web` só adapta HTTP → comando ou query.

---

### Task 1: Scaffold, Postgres e sessão de teste

**Files:**
- Create: `pyproject.toml`
- Create: `docker-compose.yml`
- Create: `src/petia/__init__.py`
- Create: `src/petia/runtime/db.py`
- Create: `tests/conftest.py`
- Create: `tests/test_db_smoke.py`

**Interfaces:**
- Consumes: nothing
- Produces: `get_engine() -> Engine`, `session_scope() -> Iterator[Session]`, fixture `db_session`, `DATABASE_URL` default `postgresql+psycopg://petia:petia@localhost:5432/petia_test`

- [ ] **Step 1: Write the failing test**

Create `tests/test_db_smoke.py`:

```python
def test_database_executes_select(db_session):
    value = db_session.execute(__import__("sqlalchemy").text("SELECT 1")).scalar()
    assert value == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_db_smoke.py::test_database_executes_select -v`

Expected: FAIL with `db_session` fixture not found or connection refused.

- [ ] **Step 3: Write minimal implementation**

`pyproject.toml`:

```toml
[project]
name = "petia"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
  "fastapi>=0.115",
  "uvicorn[standard]>=0.32",
  "sqlalchemy>=2.0.36",
  "psycopg[binary]>=3.2",
  "pydantic>=2.9",
  "jinja2>=3.1",
  "python-multipart>=0.0.12",
  "alembic>=1.14",
]

[project.optional-dependencies]
dev = ["pytest>=8.3", "httpx>=0.27"]

[tool.pytest.ini_options]
pythonpath = ["src"]
```

`docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: petia
      POSTGRES_PASSWORD: petia
      POSTGRES_DB: petia_test
    ports:
      - "5432:5432"
```

`src/petia/runtime/db.py`:

```python
import os
from collections.abc import Iterator
from contextlib import contextmanager

from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

DATABASE_URL = os.environ.get(
    "DATABASE_URL",
    "postgresql+psycopg://petia:petia@localhost:5432/petia_test",
)

_engine = None
_Session = None


def get_engine():
    global _engine, _Session
    if _engine is None:
        _engine = create_engine(DATABASE_URL, future=True)
        _Session = sessionmaker(_engine, expire_on_commit=False, class_=Session)
    return _engine


@contextmanager
def session_scope() -> Iterator[Session]:
    session = _Session()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()
```

`tests/conftest.py`:

```python
import pytest
from sqlalchemy import text

from petia.runtime.db import get_engine, session_scope


@pytest.fixture
def db_session():
    engine = get_engine()
    with engine.connect() as conn:
        conn.execute(text("SELECT 1"))
    with session_scope() as session:
        yield session
```

`src/petia/__init__.py` empty.

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `docker compose up -d postgres` then `pip install -e ".[dev]"` then `pytest tests/test_db_smoke.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add pyproject.toml docker-compose.yml src/petia/__init__.py src/petia/runtime/db.py tests/conftest.py tests/test_db_smoke.py
git commit -m "chore: scaffold PETIA runtime and postgres test session"
```

---

### Task 2: Envelope, catálogo e CommandBus

**Files:**
- Create: `src/petia/command_os/envelope.py`
- Create: `src/petia/command_os/catalog.py`
- Create: `src/petia/command_os/bus.py`
- Create: `src/petia/command_os/tables.py`
- Create: `tests/test_command_bus.py`
- Modify: `tests/conftest.py`

**Interfaces:**
- Consumes: `Session` from Task 1
- Produces:
  - `Actor(kind: ActorKind, id: str)`
  - `Command(name: str, tenant_id: UUID, payload: dict, actor: Actor, unidade_id: UUID | None = None, idempotency_key: str | None = None, command_id: UUID = ...)`
  - `CommandError(code: str, message: str)`
  - `DomainEvent(name: str, tenant_id: UUID, unidade_id: UUID | None, pet_id: UUID | None, tutor_id: UUID | None, payload: dict)`
  - `CommandResult(ok: bool, data: dict, error: CommandError | None, events: list[DomainEvent], sql_statements: list[str])`
  - `Handler = Callable[[Session, Command], CommandResult]`
  - `Catalog.register(name: str, handler: Handler) -> None`
  - `Catalog.get(name: str) -> Handler | None`
  - `CommandBus.execute(session, command) -> CommandResult` — rejeita `FORA_DO_CATALOGO`; grava `comando_log`; se `idempotency_key` já existir no mesmo tenant, devolve o resultado anterior sem segundo efeito

- [ ] **Step 1: Write the failing test**

`tests/test_command_bus.py`:

```python
from uuid import uuid4

from petia.command_os.bus import CommandBus
from petia.command_os.catalog import Catalog
from petia.command_os.envelope import Actor, ActorKind, Command, CommandResult, DomainEvent


def _actor():
    return Actor(kind=ActorKind.USUARIO, id=str(uuid4()))


def test_unknown_command_is_rejected(db_session):
    bus = CommandBus(Catalog())
    result = bus.execute(
        db_session,
        Command(name="NaoExiste", tenant_id=uuid4(), payload={}, actor=_actor()),
    )
    assert result.ok is False
    assert result.error.code == "FORA_DO_CATALOGO"


def test_handler_runs_and_is_logged(db_session):
    from sqlalchemy import text

    catalog = Catalog()

    def ping(session, command):
        sql = "SELECT 1"
        session.execute(text(sql))
        return CommandResult(
            ok=True,
            data={"pong": True},
            error=None,
            events=[
                DomainEvent(
                    name="Ping",
                    tenant_id=command.tenant_id,
                    unidade_id=command.unidade_id,
                    pet_id=None,
                    tutor_id=None,
                    payload={},
                )
            ],
            sql_statements=[sql],
        )

    catalog.register("Ping", ping)
    bus = CommandBus(catalog)
    tenant_id = uuid4()
    result = bus.execute(
        db_session,
        Command(name="Ping", tenant_id=tenant_id, payload={}, actor=_actor()),
    )
    assert result.ok is True
    row = db_session.execute(
        text("SELECT name, sql_audit FROM comando_log WHERE tenant_id = :t"),
        {"t": tenant_id},
    ).mappings().one()
    assert row["name"] == "Ping"
    assert "SELECT 1" in row["sql_audit"]


def test_idempotency_key_does_not_double_write(db_session):
    catalog = Catalog()
    calls = {"n": 0}

    def ping(session, command):
        calls["n"] += 1
        return CommandResult(ok=True, data={"n": calls["n"]}, error=None, events=[], sql_statements=[])

    catalog.register("Ping", ping)
    bus = CommandBus(catalog)
    tenant_id = uuid4()
    cmd = lambda: Command(
        name="Ping",
        tenant_id=tenant_id,
        payload={},
        actor=_actor(),
        idempotency_key="k1",
    )
    first = bus.execute(db_session, cmd())
    second = bus.execute(db_session, cmd())
    assert first.data == second.data
    assert calls["n"] == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_command_bus.py -v`

Expected: FAIL with `ModuleNotFoundError: petia.command_os`

- [ ] **Step 3: Write minimal implementation**

`src/petia/command_os/envelope.py`:

```python
from dataclasses import dataclass, field
from enum import Enum
from uuid import UUID, uuid4


class ActorKind(str, Enum):
    USUARIO = "usuario"
    TUTOR_WHATSAPP = "tutor_whatsapp"
    SISTEMA = "sistema"


@dataclass(frozen=True)
class Actor:
    kind: ActorKind
    id: str


@dataclass(frozen=True)
class Command:
    name: str
    tenant_id: UUID
    payload: dict
    actor: Actor
    unidade_id: UUID | None = None
    idempotency_key: str | None = None
    command_id: UUID = field(default_factory=uuid4)


@dataclass(frozen=True)
class CommandError:
    code: str
    message: str


@dataclass(frozen=True)
class DomainEvent:
    name: str
    tenant_id: UUID
    unidade_id: UUID | None
    pet_id: UUID | None
    tutor_id: UUID | None
    payload: dict


@dataclass(frozen=True)
class CommandResult:
    ok: bool
    data: dict
    error: CommandError | None
    events: list[DomainEvent]
    sql_statements: list[str]
```

`src/petia/command_os/catalog.py`:

```python
from collections.abc import Callable

from sqlalchemy.orm import Session

from petia.command_os.envelope import Command, CommandResult

Handler = Callable[[Session, Command], CommandResult]


class Catalog:
    def __init__(self) -> None:
        self._handlers: dict[str, Handler] = {}

    def register(self, name: str, handler: Handler) -> None:
        self._handlers[name] = handler

    def get(self, name: str) -> Handler | None:
        return self._handlers.get(name)
```

`src/petia/command_os/tables.py`:

```python
from sqlalchemy import text
from sqlalchemy.orm import Session


CREATE_COMANDO_LOG = """
CREATE TABLE IF NOT EXISTS comando_log (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  unidade_id UUID,
  name TEXT NOT NULL,
  actor_kind TEXT NOT NULL,
  actor_id TEXT NOT NULL,
  payload JSONB NOT NULL,
  idempotency_key TEXT,
  ok BOOLEAN NOT NULL,
  error_code TEXT,
  sql_audit TEXT NOT NULL,
  result_data JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (tenant_id, idempotency_key)
)
"""


def ensure_comando_log(session: Session) -> None:
    session.execute(text(CREATE_COMANDO_LOG))
```

`src/petia/command_os/bus.py`:

```python
import json
from uuid import uuid4

from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.command_os.catalog import Catalog
from petia.command_os.envelope import Command, CommandError, CommandResult
from petia.command_os.tables import ensure_comando_log


class CommandBus:
    def __init__(self, catalog: Catalog) -> None:
        self._catalog = catalog

    def execute(self, session: Session, command: Command) -> CommandResult:
        ensure_comando_log(session)
        if command.idempotency_key:
            row = session.execute(
                text(
                    """
                    SELECT ok, error_code, result_data, sql_audit
                    FROM comando_log
                    WHERE tenant_id = :tenant_id AND idempotency_key = :key
                    """
                ),
                {"tenant_id": command.tenant_id, "key": command.idempotency_key},
            ).mappings().first()
            if row:
                error = (
                    CommandError(code=row["error_code"], message="idempotent replay")
                    if row["error_code"]
                    else None
                )
                return CommandResult(
                    ok=row["ok"],
                    data=row["result_data"],
                    error=error,
                    events=[],
                    sql_statements=[row["sql_audit"]],
                )

        handler = self._catalog.get(command.name)
        if handler is None:
            result = CommandResult(
                ok=False,
                data={},
                error=CommandError(code="FORA_DO_CATALOGO", message=f"comando {command.name} nao existe"),
                events=[],
                sql_statements=[],
            )
        else:
            result = handler(session, command)

        session.execute(
            text(
                """
                INSERT INTO comando_log (
                  id, tenant_id, unidade_id, name, actor_kind, actor_id,
                  payload, idempotency_key, ok, error_code, sql_audit, result_data
                ) VALUES (
                  :id, :tenant_id, :unidade_id, :name, :actor_kind, :actor_id,
                  CAST(:payload AS JSONB), :idempotency_key, :ok, :error_code, :sql_audit, CAST(:result_data AS JSONB)
                )
                """
            ),
            {
                "id": command.command_id if command.command_id else uuid4(),
                "tenant_id": command.tenant_id,
                "unidade_id": command.unidade_id,
                "name": command.name,
                "actor_kind": command.actor.kind.value,
                "actor_id": command.actor.id,
                "payload": json.dumps(command.payload),
                "idempotency_key": command.idempotency_key,
                "ok": result.ok,
                "error_code": None if result.error is None else result.error.code,
                "sql_audit": "\n".join(result.sql_statements),
                "result_data": json.dumps(result.data),
            },
        )
        return result
```

Update `tests/conftest.py` to call `ensure_comando_log` is unnecessary if bus creates the table. Keep as-is.

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest tests/test_command_bus.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/petia/command_os tests/test_command_bus.py tests/conftest.py
git commit -m "feat: add command catalog, bus, and SQL audit log"
```

---

### Task 3: Tenant, Unidade, Usuario e isolamento

**Files:**
- Create: `src/petia/identity/tables.py`
- Create: `src/petia/identity/handlers.py`
- Create: `src/petia/identity/bootstrap.py`
- Create: `tests/test_isolamento.py`
- Modify: `src/petia/command_os/catalog.py` usage via `build_catalog()` — Create: `src/petia/command_os/build.py`

**Interfaces:**
- Consumes: `CommandBus.execute`, `Command`
- Produces:
  - `ensure_identity_tables(session)`
  - `seed_piloto(session) -> tuple[UUID, UUID, UUID]`  # tenant_id, unidade_id, usuario_id
  - `assert_unidade_in_tenant(session, tenant_id, unidade_id) -> CommandError | None` — `TENANT_MISMATCH`
  - commands (admin interno, não WhatsApp): não adicionar `RegistrarTenant` ao copiloto; seed no bootstrap
  - `build_catalog() -> Catalog` starts empty of business commands except after later registers

- [ ] **Step 1: Write the failing test**

`tests/test_isolamento.py`:

```python
from uuid import uuid4

from sqlalchemy import text

from petia.command_os.bus import CommandBus
from petia.command_os.build import build_catalog
from petia.command_os.envelope import Actor, ActorKind, Command
from petia.identity.bootstrap import seed_piloto
from petia.identity.tables import ensure_identity_tables


def test_unidade_de_outro_tenant_e_rejeitada(db_session):
    ensure_identity_tables(db_session)
    tenant_a, unidade_a, usuario_a = seed_piloto(db_session, nome="A")
    tenant_b, unidade_b, usuario_b = seed_piloto(db_session, nome="B")
    bus = CommandBus(build_catalog())
    result = bus.execute(
        db_session,
        Command(
            name="Agendar",
            tenant_id=tenant_a,
            unidade_id=unidade_b,
            payload={
                "pet_id": str(uuid4()),
                "servico_id": str(uuid4()),
                "profissional_id": str(uuid4()),
                "inicio": "2026-09-01T10:00:00-03:00",
            },
            actor=Actor(ActorKind.USUARIO, str(usuario_a)),
        ),
    )
    assert result.ok is False
    assert result.error.code == "TENANT_MISMATCH"


def test_seed_piloto_cria_uma_unidade(db_session):
    from petia.identity.tables import ensure_identity_tables
    ensure_identity_tables(db_session)
    tenant_id, unidade_id, usuario_id = seed_piloto(db_session, nome="Piloto")
    n = db_session.execute(
        text("SELECT count(*) FROM unidade WHERE tenant_id = :t"),
        {"t": tenant_id},
    ).scalar()
    assert n == 1
    assert usuario_id is not None
    assert unidade_id is not None
```

Note: `Agendar` may still be `FORA_DO_CATALOGO` until Task 5. For this task, register a tiny `_RequireUnidade` command in identity handlers used only to prove isolation, named `PingUnidade`, so the test does not depend on `Agendar`.

Replace the first test with `PingUnidade`:

```python
def test_unidade_de_outro_tenant_e_rejeitada(db_session):
    from petia.identity.tables import ensure_identity_tables
    from petia.identity.bootstrap import seed_piloto
    from petia.command_os.build import build_catalog
    from petia.command_os.bus import CommandBus
    from petia.command_os.envelope import Actor, ActorKind, Command

    ensure_identity_tables(db_session)
    tenant_a, unidade_a, usuario_a = seed_piloto(db_session, nome="A")
    tenant_b, unidade_b, _ = seed_piloto(db_session, nome="B")
    bus = CommandBus(build_catalog())
    result = bus.execute(
        db_session,
        Command(
            name="PingUnidade",
            tenant_id=tenant_a,
            unidade_id=unidade_b,
            payload={},
            actor=Actor(ActorKind.USUARIO, str(usuario_a)),
        ),
    )
    assert result.ok is False
    assert result.error.code == "TENANT_MISMATCH"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_isolamento.py -v`

Expected: FAIL with missing modules.

- [ ] **Step 3: Write minimal implementation**

`src/petia/identity/tables.py`:

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

DDL = """
CREATE TABLE IF NOT EXISTS tenant (
  id UUID PRIMARY KEY,
  nome TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE IF NOT EXISTS unidade (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL REFERENCES tenant(id),
  nome TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE IF NOT EXISTS usuario (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL REFERENCES tenant(id),
  unidade_id UUID REFERENCES unidade(id),
  nome TEXT NOT NULL,
  papel TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
"""


def ensure_identity_tables(session: Session) -> None:
    for stmt in DDL.strip().split(";"):
        if stmt.strip():
            session.execute(text(stmt))
```

`src/petia/identity/handlers.py`:

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.command_os.envelope import Command, CommandError, CommandResult


def assert_unidade_in_tenant(session: Session, command: Command) -> CommandError | None:
    if command.unidade_id is None:
        return CommandError(code="UNIDADE_OBRIGATORIA", message="unidade_id e obrigatorio")
    row = session.execute(
        text("SELECT tenant_id FROM unidade WHERE id = :id"),
        {"id": command.unidade_id},
    ).first()
    if row is None or str(row[0]) != str(command.tenant_id):
        return CommandError(code="TENANT_MISMATCH", message="unidade nao pertence ao tenant")
    return None


def handle_ping_unidade(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    sql = "SELECT id FROM unidade WHERE id = :id AND tenant_id = :tenant_id"
    session.execute(text(sql), {"id": command.unidade_id, "tenant_id": command.tenant_id})
    return CommandResult(ok=True, data={"ok": True}, error=None, events=[], sql_statements=[sql])
```

`src/petia/identity/bootstrap.py`:

```python
from uuid import uuid4

from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.identity.tables import ensure_identity_tables


def seed_piloto(session: Session, nome: str) -> tuple:
    ensure_identity_tables(session)
    tenant_id = uuid4()
    unidade_id = uuid4()
    usuario_id = uuid4()
    session.execute(text("INSERT INTO tenant (id, nome) VALUES (:id, :nome)"), {"id": tenant_id, "nome": nome})
    session.execute(
        text("INSERT INTO unidade (id, tenant_id, nome) VALUES (:id, :tenant_id, :nome)"),
        {"id": unidade_id, "tenant_id": tenant_id, "nome": f"Loja {nome}"},
    )
    session.execute(
        text(
            """
            INSERT INTO usuario (id, tenant_id, unidade_id, nome, papel)
            VALUES (:id, :tenant_id, :unidade_id, :nome, :papel)
            """
        ),
        {
            "id": usuario_id,
            "tenant_id": tenant_id,
            "unidade_id": unidade_id,
            "nome": "Recepcao",
            "papel": "recepcao",
        },
    )
    return tenant_id, unidade_id, usuario_id
```

`src/petia/command_os/build.py`:

```python
from petia.command_os.catalog import Catalog
from petia.identity.handlers import handle_ping_unidade


def build_catalog() -> Catalog:
    catalog = Catalog()
    catalog.register("PingUnidade", handle_ping_unidade)
    return catalog
```

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest tests/test_isolamento.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/petia/identity src/petia/command_os/build.py tests/test_isolamento.py
git commit -m "feat: add tenant-unit identity and isolation guard"
```

---

### Task 4: Cadastro 360 operacional — Tutor, Pet, Vínculo

**Files:**
- Create: `src/petia/cadastro/tables.py`
- Create: `src/petia/cadastro/handlers.py`
- Create: `src/petia/cadastro/queries.py`
- Create: `tests/test_cadastro.py`
- Modify: `src/petia/command_os/build.py`

**Interfaces:**
- Consumes: `assert_unidade_in_tenant`, `CommandBus`, `DomainEvent`
- Produces:
  - `handle_registrar_tutor` payload `{nome, telefone, endereco?, preferencias_comunicacao?}` → `{tutor_id}` event `TutorRegistrado`
  - `handle_registrar_pet` payload `{nome, especie, raca?, sexo?, nascimento?, peso?, porte?, pelagem?, temperamento?, preferencias?, restricoes?, alergias?, observacoes?}` → `{pet_id}` event `PetRegistrado`
  - `handle_vincular_pet` payload `{tutor_id, pet_id, principal: bool}` — v1 um tutor principal

- [ ] **Step 1: Write the failing test**

`tests/test_cadastro.py`:

```python
from petia.command_os.bus import CommandBus
from petia.command_os.build import build_catalog
from petia.command_os.envelope import Actor, ActorKind, Command
from petia.identity.bootstrap import seed_piloto
from petia.identity.tables import ensure_identity_tables
from petia.cadastro.tables import ensure_cadastro_tables


def test_registrar_tutor_pet_e_vincular(db_session):
    ensure_identity_tables(db_session)
    ensure_cadastro_tables(db_session)
    tenant_id, unidade_id, usuario_id = seed_piloto(db_session, nome="Cad")
    bus = CommandBus(build_catalog())
    actor = Actor(ActorKind.USUARIO, str(usuario_id))
    tutor = bus.execute(
        db_session,
        Command(
            name="RegistrarTutor",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"nome": "Ana", "telefone": "11999999999"},
            actor=actor,
        ),
    )
    assert tutor.ok
    pet = bus.execute(
        db_session,
        Command(
            name="RegistrarPet",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"nome": "Thor", "especie": "cao", "raca": "Golden Retriever", "peso": 32.4},
            actor=actor,
        ),
    )
    assert pet.ok
    assert any(e.name == "PetRegistrado" for e in pet.events)
    vinculo = bus.execute(
        db_session,
        Command(
            name="VincularPet",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"tutor_id": tutor.data["tutor_id"], "pet_id": pet.data["pet_id"], "principal": True},
            actor=actor,
        ),
    )
    assert vinculo.ok
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_cadastro.py -v`

Expected: FAIL `FORA_DO_CATALOGO` or import error.

- [ ] **Step 3: Write minimal implementation**

`src/petia/cadastro/tables.py`:

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

DDL = """
CREATE TABLE IF NOT EXISTS tutor (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  nome TEXT NOT NULL,
  telefone TEXT NOT NULL,
  endereco TEXT,
  preferencias_comunicacao TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE IF NOT EXISTS pet (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  nome TEXT NOT NULL,
  especie TEXT NOT NULL,
  raca TEXT,
  sexo TEXT,
  nascimento DATE,
  peso NUMERIC,
  porte TEXT,
  pelagem TEXT,
  temperamento TEXT,
  preferencias TEXT,
  restricoes TEXT,
  alergias TEXT,
  observacoes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE IF NOT EXISTS vinculo_tutor_pet (
  tutor_id UUID NOT NULL,
  pet_id UUID NOT NULL,
  tenant_id UUID NOT NULL,
  principal BOOLEAN NOT NULL DEFAULT true,
  PRIMARY KEY (tenant_id, tutor_id, pet_id)
);
"""


def ensure_cadastro_tables(session: Session) -> None:
    for stmt in DDL.strip().split(";"):
        if stmt.strip():
            session.execute(text(stmt))
```

`src/petia/cadastro/handlers.py`:

```python
from uuid import uuid4

from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.command_os.envelope import Command, CommandResult, DomainEvent
from petia.identity.handlers import assert_unidade_in_tenant


def handle_registrar_tutor(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    tutor_id = uuid4()
    sql = """
    INSERT INTO tutor (id, tenant_id, nome, telefone, endereco, preferencias_comunicacao)
    VALUES (:id, :tenant_id, :nome, :telefone, :endereco, :preferencias_comunicacao)
    """
    session.execute(
        text(sql),
        {
            "id": tutor_id,
            "tenant_id": command.tenant_id,
            "nome": command.payload["nome"],
            "telefone": command.payload["telefone"],
            "endereco": command.payload.get("endereco"),
            "preferencias_comunicacao": command.payload.get("preferencias_comunicacao"),
        },
    )
    return CommandResult(
        ok=True,
        data={"tutor_id": str(tutor_id)},
        error=None,
        events=[
            DomainEvent(
                name="TutorRegistrado",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=None,
                tutor_id=tutor_id,
                payload={"nome": command.payload["nome"]},
            )
        ],
        sql_statements=[sql],
    )


def handle_registrar_pet(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    pet_id = uuid4()
    sql = """
    INSERT INTO pet (
      id, tenant_id, nome, especie, raca, sexo, nascimento, peso, porte, pelagem,
      temperamento, preferencias, restricoes, alergias, observacoes
    ) VALUES (
      :id, :tenant_id, :nome, :especie, :raca, :sexo, :nascimento, :peso, :porte, :pelagem,
      :temperamento, :preferencias, :restricoes, :alergias, :observacoes
    )
    """
    session.execute(
        text(sql),
        {
            "id": pet_id,
            "tenant_id": command.tenant_id,
            "nome": command.payload["nome"],
            "especie": command.payload["especie"],
            "raca": command.payload.get("raca"),
            "sexo": command.payload.get("sexo"),
            "nascimento": command.payload.get("nascimento"),
            "peso": command.payload.get("peso"),
            "porte": command.payload.get("porte"),
            "pelagem": command.payload.get("pelagem"),
            "temperamento": command.payload.get("temperamento"),
            "preferencias": command.payload.get("preferencias"),
            "restricoes": command.payload.get("restricoes"),
            "alergias": command.payload.get("alergias"),
            "observacoes": command.payload.get("observacoes"),
        },
    )
    return CommandResult(
        ok=True,
        data={"pet_id": str(pet_id)},
        error=None,
        events=[
            DomainEvent(
                name="PetRegistrado",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=pet_id,
                tutor_id=None,
                payload={"nome": command.payload["nome"]},
            )
        ],
        sql_statements=[sql],
    )


def handle_vincular_pet(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    sql = """
    INSERT INTO vinculo_tutor_pet (tutor_id, pet_id, tenant_id, principal)
    VALUES (:tutor_id, :pet_id, :tenant_id, :principal)
    """
    session.execute(
        text(sql),
        {
            "tutor_id": command.payload["tutor_id"],
            "pet_id": command.payload["pet_id"],
            "tenant_id": command.tenant_id,
            "principal": command.payload.get("principal", True),
        },
    )
    return CommandResult(ok=True, data={}, error=None, events=[], sql_statements=[sql])
```

`src/petia/cadastro/queries.py`:

```python
from uuid import UUID

from sqlalchemy import text
from sqlalchemy.orm import Session


def get_pet(session: Session, tenant_id: UUID, pet_id: UUID) -> dict | None:
    sql = "SELECT id, nome, especie, raca, peso FROM pet WHERE tenant_id = :tenant_id AND id = :id"
    row = session.execute(text(sql), {"tenant_id": tenant_id, "id": pet_id}).mappings().first()
    return dict(row) if row else None
```

Register in `build_catalog()`:

```python
from petia.cadastro.handlers import handle_registrar_pet, handle_registrar_tutor, handle_vincular_pet
from petia.command_os.catalog import Catalog
from petia.identity.handlers import handle_ping_unidade


def build_catalog() -> Catalog:
    catalog = Catalog()
    catalog.register("PingUnidade", handle_ping_unidade)
    catalog.register("RegistrarTutor", handle_registrar_tutor)
    catalog.register("RegistrarPet", handle_registrar_pet)
    catalog.register("VincularPet", handle_vincular_pet)
    return catalog
```

Replace the entire `src/petia/command_os/build.py` with the block above (keep PingUnidade).

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest tests/test_cadastro.py tests/test_isolamento.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/petia/cadastro src/petia/command_os/build.py tests/test_cadastro.py
git commit -m "feat: add operational tutor and pet registration commands"
```

---

### Task 5: Serviços, profissional e Agendar (duração fixa, slot ocupado)

**Files:**
- Create: `src/petia/agenda/tables.py`
- Create: `src/petia/agenda/handlers.py`
- Create: `src/petia/agenda/queries.py`
- Create: `src/petia/agenda/seed.py`
- Create: `tests/test_agenda.py`
- Modify: `src/petia/command_os/build.py`

**Interfaces:**
- Consumes: `assert_unidade_in_tenant`, cadastro pet
- Produces: `handle_agendar` → `{agendamento_id}` event `AgendamentoCriado`; errors `SLOT_OCUPADO`, `SERVICO_INEXISTENTE`
- Payload `Agendar`: `{pet_id, servico_id, profissional_id, inicio}` ISO-8601. `fim = inicio + servico.duracao_minutos`. Status inicial `agendado`.
- `seed_servicos_piloto(session, tenant_id, unidade_id) -> dict` with `banho_id`, `tosa_id`, `profissional_id`

- [ ] **Step 1: Write the failing test**

`tests/test_agenda.py`:

```python
from datetime import datetime, timezone

from petia.agenda.seed import seed_servicos_piloto
from petia.agenda.tables import ensure_agenda_tables
from petia.cadastro.tables import ensure_cadastro_tables
from petia.command_os.bus import CommandBus
from petia.command_os.build import build_catalog
from petia.command_os.envelope import Actor, ActorKind, Command
from petia.identity.bootstrap import seed_piloto
from petia.identity.tables import ensure_identity_tables


def _setup(db_session):
    ensure_identity_tables(db_session)
    ensure_cadastro_tables(db_session)
    ensure_agenda_tables(db_session)
    tenant_id, unidade_id, usuario_id = seed_piloto(db_session, nome="Agenda")
    ids = seed_servicos_piloto(db_session, tenant_id, unidade_id)
    bus = CommandBus(build_catalog())
    actor = Actor(ActorKind.USUARIO, str(usuario_id))
    pet = bus.execute(
        db_session,
        Command(
            name="RegistrarPet",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"nome": "Thor", "especie": "cao"},
            actor=actor,
        ),
    )
    return tenant_id, unidade_id, actor, bus, ids, pet.data["pet_id"]


def test_agendar_banho(db_session):
    tenant_id, unidade_id, actor, bus, ids, pet_id = _setup(db_session)
    result = bus.execute(
        db_session,
        Command(
            name="Agendar",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={
                "pet_id": pet_id,
                "servico_id": ids["banho_id"],
                "profissional_id": ids["profissional_id"],
                "inicio": "2026-09-01T10:00:00-03:00",
            },
            actor=actor,
        ),
    )
    assert result.ok
    assert any(e.name == "AgendamentoCriado" for e in result.events)


def test_slot_ocupado(db_session):
    tenant_id, unidade_id, actor, bus, ids, pet_id = _setup(db_session)
    payload = {
        "pet_id": pet_id,
        "servico_id": ids["banho_id"],
        "profissional_id": ids["profissional_id"],
        "inicio": "2026-09-01T10:00:00-03:00",
    }
    first = bus.execute(db_session, Command(name="Agendar", tenant_id=tenant_id, unidade_id=unidade_id, payload=payload, actor=actor))
    assert first.ok
    second = bus.execute(db_session, Command(name="Agendar", tenant_id=tenant_id, unidade_id=unidade_id, payload=payload, actor=actor))
    assert second.ok is False
    assert second.error.code == "SLOT_OCUPADO"


def test_servico_inexistente(db_session):
    from uuid import uuid4
    tenant_id, unidade_id, actor, bus, ids, pet_id = _setup(db_session)
    result = bus.execute(
        db_session,
        Command(
            name="Agendar",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={
                "pet_id": pet_id,
                "servico_id": str(uuid4()),
                "profissional_id": ids["profissional_id"],
                "inicio": "2026-09-01T11:00:00-03:00",
            },
            actor=actor,
        ),
    )
    assert result.ok is False
    assert result.error.code == "SERVICO_INEXISTENTE"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_agenda.py -v`

Expected: FAIL missing `Agendar`.

- [ ] **Step 3: Write minimal implementation**

`src/petia/agenda/tables.py`:

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

DDL = """
CREATE TABLE IF NOT EXISTS servico (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  unidade_id UUID NOT NULL,
  nome TEXT NOT NULL,
  duracao_minutos INTEGER NOT NULL,
  preco_centavos INTEGER NOT NULL
);
CREATE TABLE IF NOT EXISTS profissional (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  unidade_id UUID NOT NULL,
  nome TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS agendamento (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  unidade_id UUID NOT NULL,
  pet_id UUID NOT NULL,
  servico_id UUID NOT NULL,
  profissional_id UUID NOT NULL,
  inicio TIMESTAMPTZ NOT NULL,
  fim TIMESTAMPTZ NOT NULL,
  status TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
"""


def ensure_agenda_tables(session: Session) -> None:
    for stmt in DDL.strip().split(";"):
        if stmt.strip():
            session.execute(text(stmt))
```

`src/petia/agenda/seed.py`:

```python
from uuid import uuid4

from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.agenda.tables import ensure_agenda_tables


def seed_servicos_piloto(session: Session, tenant_id, unidade_id) -> dict:
    ensure_agenda_tables(session)
    banho_id = uuid4()
    tosa_id = uuid4()
    profissional_id = uuid4()
    session.execute(
        text(
            """
            INSERT INTO servico (id, tenant_id, unidade_id, nome, duracao_minutos, preco_centavos)
            VALUES (:id, :tenant_id, :unidade_id, :nome, :duracao, :preco)
            """
        ),
        {"id": banho_id, "tenant_id": tenant_id, "unidade_id": unidade_id, "nome": "Banho", "duracao": 60, "preco": 8000},
    )
    session.execute(
        text(
            """
            INSERT INTO servico (id, tenant_id, unidade_id, nome, duracao_minutos, preco_centavos)
            VALUES (:id, :tenant_id, :unidade_id, :nome, :duracao, :preco)
            """
        ),
        {"id": tosa_id, "tenant_id": tenant_id, "unidade_id": unidade_id, "nome": "Tosa", "duracao": 90, "preco": 12000},
    )
    session.execute(
        text(
            "INSERT INTO profissional (id, tenant_id, unidade_id, nome) VALUES (:id, :tenant_id, :unidade_id, :nome)"
        ),
        {"id": profissional_id, "tenant_id": tenant_id, "unidade_id": unidade_id, "nome": "Carla"},
    )
    return {"banho_id": str(banho_id), "tosa_id": str(tosa_id), "profissional_id": str(profissional_id)}
```

`src/petia/agenda/handlers.py`:

```python
from datetime import datetime, timedelta
from uuid import UUID, uuid4

from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.command_os.envelope import Command, CommandError, CommandResult, DomainEvent
from petia.identity.handlers import assert_unidade_in_tenant


def _parse_inicio(value: str) -> datetime:
    return datetime.fromisoformat(value)


def handle_agendar(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    servico_sql = """
    SELECT duracao_minutos FROM servico
    WHERE id = :id AND tenant_id = :tenant_id AND unidade_id = :unidade_id
    """
    servico = session.execute(
        text(servico_sql),
        {
            "id": command.payload["servico_id"],
            "tenant_id": command.tenant_id,
            "unidade_id": command.unidade_id,
        },
    ).first()
    if servico is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="servico nao encontrado nesta unidade"),
            events=[],
            sql_statements=[servico_sql],
        )
    inicio = _parse_inicio(command.payload["inicio"])
    fim = inicio + timedelta(minutes=int(servico[0]))
    overlap_sql = """
    SELECT id FROM agendamento
    WHERE tenant_id = :tenant_id
      AND unidade_id = :unidade_id
      AND profissional_id = :profissional_id
      AND status IN ('agendado', 'confirmado', 'em_atendimento')
      AND inicio < :fim AND fim > :inicio
    """
    overlap = session.execute(
        text(overlap_sql),
        {
            "tenant_id": command.tenant_id,
            "unidade_id": command.unidade_id,
            "profissional_id": command.payload["profissional_id"],
            "inicio": inicio,
            "fim": fim,
        },
    ).first()
    if overlap:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SLOT_OCUPADO", message="profissional ocupado neste horario"),
            events=[],
            sql_statements=[servico_sql, overlap_sql],
        )
    agendamento_id = uuid4()
    insert_sql = """
    INSERT INTO agendamento (
      id, tenant_id, unidade_id, pet_id, servico_id, profissional_id, inicio, fim, status
    ) VALUES (
      :id, :tenant_id, :unidade_id, :pet_id, :servico_id, :profissional_id, :inicio, :fim, :status
    )
    """
    session.execute(
        text(insert_sql),
        {
            "id": agendamento_id,
            "tenant_id": command.tenant_id,
            "unidade_id": command.unidade_id,
            "pet_id": command.payload["pet_id"],
            "servico_id": command.payload["servico_id"],
            "profissional_id": command.payload["profissional_id"],
            "inicio": inicio,
            "fim": fim,
            "status": "agendado",
        },
    )
    return CommandResult(
        ok=True,
        data={"agendamento_id": str(agendamento_id), "inicio": inicio.isoformat(), "fim": fim.isoformat()},
        error=None,
        events=[
            DomainEvent(
                name="AgendamentoCriado",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=UUID(command.payload["pet_id"]),
                tutor_id=None,
                payload={"agendamento_id": str(agendamento_id)},
            )
        ],
        sql_statements=[servico_sql, overlap_sql, insert_sql],
    )
```

`src/petia/agenda/queries.py`:

```python
from datetime import date
from uuid import UUID

from sqlalchemy import text
from sqlalchemy.orm import Session


def agenda_do_dia(session: Session, tenant_id: UUID, unidade_id: UUID, dia: date) -> list[dict]:
    sql = """
    SELECT a.id, a.inicio, a.fim, a.status, p.nome AS pet_nome, s.nome AS servico_nome, pr.nome AS profissional_nome
    FROM agendamento a
    JOIN pet p ON p.id = a.pet_id AND p.tenant_id = a.tenant_id
    JOIN servico s ON s.id = a.servico_id AND s.tenant_id = a.tenant_id
    JOIN profissional pr ON pr.id = a.profissional_id AND pr.tenant_id = a.tenant_id
    WHERE a.tenant_id = :tenant_id AND a.unidade_id = :unidade_id
      AND a.inicio::date = :dia
    ORDER BY a.inicio
    """
    rows = session.execute(
        text(sql),
        {"tenant_id": tenant_id, "unidade_id": unidade_id, "dia": dia},
    ).mappings()
    return [dict(r) for r in rows]
```

Add to `build_catalog()`: `catalog.register("Agendar", handle_agendar)`.

Write the **full** `src/petia/command_os/build.py` after this task:

```python
from petia.agenda.handlers import handle_agendar
from petia.cadastro.handlers import handle_registrar_pet, handle_registrar_tutor, handle_vincular_pet
from petia.command_os.catalog import Catalog
from petia.identity.handlers import handle_ping_unidade


def build_catalog() -> Catalog:
    catalog = Catalog()
    catalog.register("PingUnidade", handle_ping_unidade)
    catalog.register("RegistrarTutor", handle_registrar_tutor)
    catalog.register("RegistrarPet", handle_registrar_pet)
    catalog.register("VincularPet", handle_vincular_pet)
    catalog.register("Agendar", handle_agendar)
    return catalog
```

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest tests/test_agenda.py tests/test_cadastro.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/petia/agenda src/petia/command_os/build.py tests/test_agenda.py
git commit -m "feat: add fixed-duration scheduling with slot conflict"
```

---

### Task 6: Reagendar, CancelarAgendamento, ConfirmarAgendamento

**Files:**
- Modify: `src/petia/agenda/handlers.py`
- Modify: `src/petia/command_os/build.py`
- Modify: `tests/test_agenda.py`

**Interfaces:**
- Consumes: `handle_agendar`, tables from Task 5
- Produces:
  - `handle_confirmar_agendamento` payload `{agendamento_id}` → status `confirmado`, event `AgendamentoConfirmado`
  - `handle_reagendar` payload `{agendamento_id, inicio?, profissional_id?}` → mesmas regras de overlap, event `Reagendar`
  - `handle_cancelar_agendamento` payload `{agendamento_id}` → status `cancelado`, event `CancelarAgendamento`

- [ ] **Step 1: Write the failing test**

Add to `tests/test_agenda.py`:

```python
def test_reagendar_cancelar_confirmar(db_session):
    tenant_id, unidade_id, actor, bus, ids, pet_id = _setup(db_session)
    created = bus.execute(
        db_session,
        Command(
            name="Agendar",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={
                "pet_id": pet_id,
                "servico_id": ids["banho_id"],
                "profissional_id": ids["profissional_id"],
                "inicio": "2026-09-01T14:00:00-03:00",
            },
            actor=actor,
        ),
    )
    aid = created.data["agendamento_id"]
    rsvp = bus.execute(
        db_session,
        Command(
            name="ConfirmarAgendamento",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"agendamento_id": aid},
            actor=actor,
        ),
    )
    assert rsvp.ok
    assert any(e.name == "AgendamentoConfirmado" for e in rsvp.events)
    moved = bus.execute(
        db_session,
        Command(
            name="Reagendar",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"agendamento_id": aid, "inicio": "2026-09-01T16:00:00-03:00"},
            actor=actor,
        ),
    )
    assert moved.ok
    canceled = bus.execute(
        db_session,
        Command(
            name="CancelarAgendamento",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"agendamento_id": aid},
            actor=actor,
        ),
    )
    assert canceled.ok
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_agenda.py::test_reagendar_cancelar_confirmar -v`

Expected: FAIL `FORA_DO_CATALOGO`

- [ ] **Step 3: Write minimal implementation**

Append to `src/petia/agenda/handlers.py`:

```python
def _load_agendamento(session, command, agendamento_id):
    sql = """
    SELECT id, pet_id, servico_id, profissional_id, inicio, fim, status
    FROM agendamento
    WHERE id = :id AND tenant_id = :tenant_id AND unidade_id = :unidade_id
    """
    row = session.execute(
        text(sql),
        {"id": agendamento_id, "tenant_id": command.tenant_id, "unidade_id": command.unidade_id},
    ).mappings().first()
    return row, sql


def handle_confirmar_agendamento(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    row, select_sql = _load_agendamento(session, command, command.payload["agendamento_id"])
    if row is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="agendamento nao encontrado"),
            events=[],
            sql_statements=[select_sql],
        )
    sql = """
    UPDATE agendamento SET status = 'confirmado'
    WHERE id = :id AND tenant_id = :tenant_id AND unidade_id = :unidade_id
    """
    session.execute(
        text(sql),
        {"id": command.payload["agendamento_id"], "tenant_id": command.tenant_id, "unidade_id": command.unidade_id},
    )
    return CommandResult(
        ok=True,
        data={"agendamento_id": command.payload["agendamento_id"]},
        error=None,
        events=[
            DomainEvent(
                name="AgendamentoConfirmado",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=row["pet_id"],
                tutor_id=None,
                payload={"agendamento_id": command.payload["agendamento_id"]},
            )
        ],
        sql_statements=[select_sql, sql],
    )


def handle_reagendar(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    row, select_sql = _load_agendamento(session, command, command.payload["agendamento_id"])
    if row is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="agendamento nao encontrado"),
            events=[],
            sql_statements=[select_sql],
        )
    profissional_id = command.payload.get("profissional_id", str(row["profissional_id"]))
    if "inicio" in command.payload:
        inicio = _parse_inicio(command.payload["inicio"])
        duracao = row["fim"] - row["inicio"]
        fim = inicio + duracao
    else:
        inicio, fim = row["inicio"], row["fim"]
    overlap_sql = """
    SELECT id FROM agendamento
    WHERE tenant_id = :tenant_id AND unidade_id = :unidade_id
      AND profissional_id = :profissional_id
      AND status IN ('agendado', 'confirmado', 'em_atendimento')
      AND id <> :id
      AND inicio < :fim AND fim > :inicio
    """
    overlap = session.execute(
        text(overlap_sql),
        {
            "tenant_id": command.tenant_id,
            "unidade_id": command.unidade_id,
            "profissional_id": profissional_id,
            "id": command.payload["agendamento_id"],
            "inicio": inicio,
            "fim": fim,
        },
    ).first()
    if overlap:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SLOT_OCUPADO", message="profissional ocupado neste horario"),
            events=[],
            sql_statements=[select_sql, overlap_sql],
        )
    sql = """
    UPDATE agendamento
    SET inicio = :inicio, fim = :fim, profissional_id = :profissional_id
    WHERE id = :id AND tenant_id = :tenant_id AND unidade_id = :unidade_id
    """
    session.execute(
        text(sql),
        {
            "inicio": inicio,
            "fim": fim,
            "profissional_id": profissional_id,
            "id": command.payload["agendamento_id"],
            "tenant_id": command.tenant_id,
            "unidade_id": command.unidade_id,
        },
    )
    return CommandResult(
        ok=True,
        data={"agendamento_id": command.payload["agendamento_id"], "inicio": inicio.isoformat()},
        error=None,
        events=[
            DomainEvent(
                name="Reagendar",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=row["pet_id"],
                tutor_id=None,
                payload={"agendamento_id": command.payload["agendamento_id"]},
            )
        ],
        sql_statements=[select_sql, overlap_sql, sql],
    )


def handle_cancelar_agendamento(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    row, select_sql = _load_agendamento(session, command, command.payload["agendamento_id"])
    if row is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="agendamento nao encontrado"),
            events=[],
            sql_statements=[select_sql],
        )
    sql = """
    UPDATE agendamento SET status = 'cancelado'
    WHERE id = :id AND tenant_id = :tenant_id AND unidade_id = :unidade_id
    """
    session.execute(
        text(sql),
        {"id": command.payload["agendamento_id"], "tenant_id": command.tenant_id, "unidade_id": command.unidade_id},
    )
    return CommandResult(
        ok=True,
        data={"agendamento_id": command.payload["agendamento_id"]},
        error=None,
        events=[
            DomainEvent(
                name="CancelarAgendamento",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=row["pet_id"],
                tutor_id=None,
                payload={"agendamento_id": command.payload["agendamento_id"]},
            )
        ],
        sql_statements=[select_sql, sql],
    )
```

Full `src/petia/command_os/build.py` after this task also registers `Reagendar`, `CancelarAgendamento`, `ConfirmarAgendamento`.

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest tests/test_agenda.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/petia/agenda/handlers.py src/petia/command_os/build.py tests/test_agenda.py
git commit -m "feat: add reschedule, cancel, and appointment RSVP commands"
```

---

### Task 7: Check-in, OS entrada → execução → saída

**Files:**
- Create: `src/petia/os_banho/tables.py`
- Create: `src/petia/os_banho/handlers.py`
- Create: `tests/test_os.py`
- Modify: `src/petia/command_os/build.py`

**Interfaces:**
- Consumes: agendamento, `assert_unidade_in_tenant`
- Produces:
  - `ConfirmarPresenca` `{agendamento_id}` status pet present; event `PresencaConfirmada`; may be followed by `AbrirOS`
  - `AbrirOS` `{agendamento_id}` creates OS `aberta`; event `OSAberta`; duplicate open on same agendamento is idempotent via unique `agendamento_id`
  - `RegistrarEntradaOS` `{os_id, pelagem?, nos?, parasitas?, lesoes?, comportamento?, objetos?, observacoes?, fotos: list[str]}` event `OSEntradaRegistrada`
  - `AtualizarExecucaoOS` `{os_id, iniciado_em?, profissional_id?, servicos?, produtos?, ocorrencias?}`
  - `RegistrarSaidaOS` `{os_id, observacoes?, fotos: list[str], proxima_visita_sugerida?}` event `OSConcluida`; agendamento status `concluido`

- [ ] **Step 1: Write the failing test**

`tests/test_os.py`:

```python
from petia.agenda.seed import seed_servicos_piloto
from petia.agenda.tables import ensure_agenda_tables
from petia.cadastro.tables import ensure_cadastro_tables
from petia.command_os.bus import CommandBus
from petia.command_os.build import build_catalog
from petia.command_os.envelope import Actor, ActorKind, Command
from petia.identity.bootstrap import seed_piloto
from petia.identity.tables import ensure_identity_tables
from petia.os_banho.tables import ensure_os_tables


def test_loop_os(db_session):
    ensure_identity_tables(db_session)
    ensure_cadastro_tables(db_session)
    ensure_agenda_tables(db_session)
    ensure_os_tables(db_session)
    tenant_id, unidade_id, usuario_id = seed_piloto(db_session, nome="OS")
    ids = seed_servicos_piloto(db_session, tenant_id, unidade_id)
    bus = CommandBus(build_catalog())
    actor = Actor(ActorKind.USUARIO, str(usuario_id))
    pet = bus.execute(
        db_session,
        Command(name="RegistrarPet", tenant_id=tenant_id, unidade_id=unidade_id, payload={"nome": "Thor", "especie": "cao"}, actor=actor),
    )
    ag = bus.execute(
        db_session,
        Command(
            name="Agendar",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={
                "pet_id": pet.data["pet_id"],
                "servico_id": ids["banho_id"],
                "profissional_id": ids["profissional_id"],
                "inicio": "2026-09-02T09:00:00-03:00",
            },
            actor=actor,
        ),
    )
    presenca = bus.execute(
        db_session,
        Command(name="ConfirmarPresenca", tenant_id=tenant_id, unidade_id=unidade_id, payload={"agendamento_id": ag.data["agendamento_id"]}, actor=actor),
    )
    assert presenca.ok
    opened = bus.execute(
        db_session,
        Command(name="AbrirOS", tenant_id=tenant_id, unidade_id=unidade_id, payload={"agendamento_id": ag.data["agendamento_id"]}, actor=actor),
    )
    assert opened.ok
    os_id = opened.data["os_id"]
    entrada = bus.execute(
        db_session,
        Command(
            name="RegistrarEntradaOS",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"os_id": os_id, "pelagem": "suja", "fotos": ["http://local/antes.jpg"]},
            actor=actor,
        ),
    )
    assert entrada.ok
    execu = bus.execute(
        db_session,
        Command(
            name="AtualizarExecucaoOS",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"os_id": os_id, "ocorrencias": "tranquilo", "produtos": "shampoo X"},
            actor=actor,
        ),
    )
    assert execu.ok
    saida = bus.execute(
        db_session,
        Command(
            name="RegistrarSaidaOS",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"os_id": os_id, "fotos": ["http://local/depois.jpg"], "proxima_visita_sugerida": "2026-09-23"},
            actor=actor,
        ),
    )
    assert saida.ok
    assert any(e.name == "OSConcluida" for e in saida.events)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_os.py -v`

Expected: FAIL missing commands.

- [ ] **Step 3: Write minimal implementation**

`src/petia/os_banho/tables.py`:

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

DDL = """
CREATE TABLE IF NOT EXISTS ordem_servico (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  unidade_id UUID NOT NULL,
  agendamento_id UUID NOT NULL UNIQUE,
  pet_id UUID NOT NULL,
  status TEXT NOT NULL,
  entrada JSONB,
  execucao JSONB,
  saida JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
"""


def ensure_os_tables(session: Session) -> None:
    for stmt in DDL.strip().split(";"):
        if stmt.strip():
            session.execute(text(stmt))
```

`src/petia/os_banho/handlers.py` — implement the five handlers with parameterized SQL, `assert_unidade_in_tenant`, events `PresencaConfirmada`, `OSAberta`, `OSEntradaRegistrada`, `OSConcluida`. `ConfirmarPresenca` updates `agendamento.status` to `em_atendimento`. `AbrirOS` inserts OS with pet_id from agendamento. `RegistrarSaidaOS` sets OS status `concluida` and agendamento `concluido`.

Full file:

```python
import json
from uuid import uuid4

from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.command_os.envelope import Command, CommandError, CommandResult, DomainEvent
from petia.identity.handlers import assert_unidade_in_tenant


def handle_confirmar_presenca(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    sql = """
    UPDATE agendamento SET status = 'em_atendimento'
    WHERE id = :id AND tenant_id = :tenant_id AND unidade_id = :unidade_id
      AND status IN ('agendado', 'confirmado')
    RETURNING pet_id
    """
    row = session.execute(
        text(sql),
        {"id": command.payload["agendamento_id"], "tenant_id": command.tenant_id, "unidade_id": command.unidade_id},
    ).first()
    if row is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="agendamento nao encontrado ou status invalido"),
            events=[],
            sql_statements=[sql],
        )
    return CommandResult(
        ok=True,
        data={"agendamento_id": command.payload["agendamento_id"]},
        error=None,
        events=[
            DomainEvent(
                name="PresencaConfirmada",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=row[0],
                tutor_id=None,
                payload={"agendamento_id": command.payload["agendamento_id"]},
            )
        ],
        sql_statements=[sql],
    )


def handle_abrir_os(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    existing_sql = """
    SELECT id FROM ordem_servico
    WHERE tenant_id = :tenant_id AND agendamento_id = :agendamento_id
    """
    existing = session.execute(
        text(existing_sql),
        {"tenant_id": command.tenant_id, "agendamento_id": command.payload["agendamento_id"]},
    ).first()
    if existing:
        return CommandResult(ok=True, data={"os_id": str(existing[0])}, error=None, events=[], sql_statements=[existing_sql])
    pet_sql = """
    SELECT pet_id FROM agendamento
    WHERE id = :id AND tenant_id = :tenant_id AND unidade_id = :unidade_id
    """
    pet = session.execute(
        text(pet_sql),
        {"id": command.payload["agendamento_id"], "tenant_id": command.tenant_id, "unidade_id": command.unidade_id},
    ).first()
    if pet is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="agendamento nao encontrado"),
            events=[],
            sql_statements=[existing_sql, pet_sql],
        )
    os_id = uuid4()
    sql = """
    INSERT INTO ordem_servico (id, tenant_id, unidade_id, agendamento_id, pet_id, status)
    VALUES (:id, :tenant_id, :unidade_id, :agendamento_id, :pet_id, 'aberta')
    """
    session.execute(
        text(sql),
        {
            "id": os_id,
            "tenant_id": command.tenant_id,
            "unidade_id": command.unidade_id,
            "agendamento_id": command.payload["agendamento_id"],
            "pet_id": pet[0],
        },
    )
    return CommandResult(
        ok=True,
        data={"os_id": str(os_id)},
        error=None,
        events=[
            DomainEvent(
                name="OSAberta",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=pet[0],
                tutor_id=None,
                payload={"os_id": str(os_id)},
            )
        ],
        sql_statements=[existing_sql, pet_sql, sql],
    )


def handle_registrar_entrada_os(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    sql = """
    UPDATE ordem_servico
    SET entrada = CAST(:entrada AS JSONB)
    WHERE id = :id AND tenant_id = :tenant_id AND unidade_id = :unidade_id
    RETURNING pet_id
    """
    entrada = {k: v for k, v in command.payload.items() if k != "os_id"}
    row = session.execute(
        text(sql),
        {
            "entrada": json.dumps(entrada),
            "id": command.payload["os_id"],
            "tenant_id": command.tenant_id,
            "unidade_id": command.unidade_id,
        },
    ).first()
    if row is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="OS nao encontrada"),
            events=[],
            sql_statements=[sql],
        )
    return CommandResult(
        ok=True,
        data={"os_id": command.payload["os_id"]},
        error=None,
        events=[
            DomainEvent(
                name="OSEntradaRegistrada",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=row[0],
                tutor_id=None,
                payload={"os_id": command.payload["os_id"]},
            )
        ],
        sql_statements=[sql],
    )


def handle_atualizar_execucao_os(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    sql = """
    UPDATE ordem_servico
    SET execucao = CAST(:execucao AS JSONB)
    WHERE id = :id AND tenant_id = :tenant_id AND unidade_id = :unidade_id
    RETURNING pet_id
    """
    execucao = {k: v for k, v in command.payload.items() if k != "os_id"}
    row = session.execute(
        text(sql),
        {
            "execucao": json.dumps(execucao),
            "id": command.payload["os_id"],
            "tenant_id": command.tenant_id,
            "unidade_id": command.unidade_id,
        },
    ).first()
    if row is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="OS nao encontrada"),
            events=[],
            sql_statements=[sql],
        )
    return CommandResult(ok=True, data={"os_id": command.payload["os_id"]}, error=None, events=[], sql_statements=[sql])


def handle_registrar_saida_os(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    sql = """
    UPDATE ordem_servico
    SET saida = CAST(:saida AS JSONB), status = 'concluida'
    WHERE id = :id AND tenant_id = :tenant_id AND unidade_id = :unidade_id
    RETURNING pet_id, agendamento_id
    """
    saida = {k: v for k, v in command.payload.items() if k != "os_id"}
    row = session.execute(
        text(sql),
        {
            "saida": json.dumps(saida),
            "id": command.payload["os_id"],
            "tenant_id": command.tenant_id,
            "unidade_id": command.unidade_id,
        },
    ).first()
    if row is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="OS nao encontrada"),
            events=[],
            sql_statements=[sql],
        )
    ag_sql = """
    UPDATE agendamento SET status = 'concluido'
    WHERE id = :id AND tenant_id = :tenant_id
    """
    session.execute(text(ag_sql), {"id": row[1], "tenant_id": command.tenant_id})
    return CommandResult(
        ok=True,
        data={"os_id": command.payload["os_id"]},
        error=None,
        events=[
            DomainEvent(
                name="OSConcluida",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=row[0],
                tutor_id=None,
                payload={"os_id": command.payload["os_id"]},
            )
        ],
        sql_statements=[sql, ag_sql],
    )
```

Register the five commands in `build_catalog()`.

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest tests/test_os.py tests/test_agenda.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/petia/os_banho src/petia/command_os/build.py tests/test_os.py
git commit -m "feat: add grooming work-order lifecycle commands"
```

---

### Task 8: Eventos e timeline do pet

**Files:**
- Create: `src/petia/timeline/tables.py`
- Create: `src/petia/timeline/projector.py`
- Create: `src/petia/timeline/queries.py`
- Create: `tests/test_timeline.py`
- Modify: `src/petia/command_os/bus.py`

**Interfaces:**
- Consumes: `CommandResult.events`
- Produces: `project_events(session, events) -> None`; `timeline_do_pet(session, tenant_id, pet_id) -> list[dict]` ordered by `created_at`
- `CommandBus.execute` after successful/any result with events calls `project_events`

- [ ] **Step 1: Write the failing test**

`tests/test_timeline.py`:

```python
from petia.agenda.seed import seed_servicos_piloto
from petia.agenda.tables import ensure_agenda_tables
from petia.cadastro.tables import ensure_cadastro_tables
from petia.command_os.bus import CommandBus
from petia.command_os.build import build_catalog
from petia.command_os.envelope import Actor, ActorKind, Command
from petia.identity.bootstrap import seed_piloto
from petia.identity.tables import ensure_identity_tables
from petia.timeline.queries import timeline_do_pet
from petia.timeline.tables import ensure_timeline_tables


def test_timeline_contem_pet_e_agendamento(db_session):
    ensure_identity_tables(db_session)
    ensure_cadastro_tables(db_session)
    ensure_agenda_tables(db_session)
    ensure_timeline_tables(db_session)
    tenant_id, unidade_id, usuario_id = seed_piloto(db_session, nome="TL")
    ids = seed_servicos_piloto(db_session, tenant_id, unidade_id)
    bus = CommandBus(build_catalog())
    actor = Actor(ActorKind.USUARIO, str(usuario_id))
    pet = bus.execute(
        db_session,
        Command(name="RegistrarPet", tenant_id=tenant_id, unidade_id=unidade_id, payload={"nome": "Thor", "especie": "cao"}, actor=actor),
    )
    bus.execute(
        db_session,
        Command(
            name="Agendar",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={
                "pet_id": pet.data["pet_id"],
                "servico_id": ids["banho_id"],
                "profissional_id": ids["profissional_id"],
                "inicio": "2026-09-03T10:00:00-03:00",
            },
            actor=actor,
        ),
    )
    items = timeline_do_pet(db_session, tenant_id, pet.data["pet_id"])
    names = [i["name"] for i in items]
    assert "PetRegistrado" in names
    assert "AgendamentoCriado" in names
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_timeline.py -v`

Expected: FAIL no `evento` table or empty timeline.

- [ ] **Step 3: Write minimal implementation**

`src/petia/timeline/tables.py`:

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

DDL = """
CREATE TABLE IF NOT EXISTS evento (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  unidade_id UUID,
  pet_id UUID,
  tutor_id UUID,
  name TEXT NOT NULL,
  payload JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
"""


def ensure_timeline_tables(session: Session) -> None:
    for stmt in DDL.strip().split(";"):
        if stmt.strip():
            session.execute(text(stmt))
```

`src/petia/timeline/projector.py`:

```python
import json
from uuid import uuid4

from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.command_os.envelope import DomainEvent
from petia.timeline.tables import ensure_timeline_tables


def project_events(session: Session, events: list[DomainEvent]) -> None:
    ensure_timeline_tables(session)
    sql = """
    INSERT INTO evento (id, tenant_id, unidade_id, pet_id, tutor_id, name, payload)
    VALUES (:id, :tenant_id, :unidade_id, :pet_id, :tutor_id, :name, CAST(:payload AS JSONB))
    """
    for event in events:
        session.execute(
            text(sql),
            {
                "id": uuid4(),
                "tenant_id": event.tenant_id,
                "unidade_id": event.unidade_id,
                "pet_id": event.pet_id,
                "tutor_id": event.tutor_id,
                "name": event.name,
                "payload": json.dumps(event.payload),
            },
        )
```

`src/petia/timeline/queries.py`:

```python
from uuid import UUID

from sqlalchemy import text
from sqlalchemy.orm import Session


def timeline_do_pet(session: Session, tenant_id: UUID, pet_id: str | UUID) -> list[dict]:
    sql = """
    SELECT name, payload, created_at
    FROM evento
    WHERE tenant_id = :tenant_id AND pet_id = :pet_id
    ORDER BY created_at
    """
    rows = session.execute(text(sql), {"tenant_id": tenant_id, "pet_id": pet_id}).mappings()
    return [dict(r) for r in rows]
```

In `CommandBus.execute`, after computing `result` and before/after log insert, call `project_events(session, result.events)`.

Import in `bus.py`: `from petia.timeline.projector import project_events` then `project_events(session, result.events)`.

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest tests/test_timeline.py tests/test_command_bus.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/petia/timeline src/petia/command_os/bus.py tests/test_timeline.py
git commit -m "feat: project domain events onto the pet timeline"
```

---

### Task 9: RecomendarProduto e NotificarPetPronto

**Files:**
- Create: `src/petia/whatsapp/gateway.py`
- Modify: `src/petia/os_banho/handlers.py`
- Create: `src/petia/os_banho/notify.py` — actually keep handlers in os_banho and a recommend handler
- Create: `src/petia/cadastro/recommend.py` no — `src/petia/os_banho/recommend.py` as functions in `handlers.py` to avoid extra hop... Create `src/petia/relacionamento/handlers.py` for these two commands
- Create: `tests/test_os.py` extra tests in `tests/test_relacionamento.py`
- Modify: `build.py`

**Interfaces:**
- Consumes: OS, tutor phone via vinculo
- Produces:
  - `WhatsAppGateway.send(to: str, body: str) -> None`
  - `FakeWhatsAppGateway.sent: list[tuple[str,str]]`
  - `handle_recomendar_produto` payload `{pet_id, produto_nome, motivo?}` event `ProdutoRecomendado`; does not touch stock
  - `handle_notificar_pet_pronto` payload `{os_id}` event `PetProntoNotificado`; looks up principal tutor telefone; calls gateway

CommandBus needs gateway injected. Change `CommandBus.__init__(self, catalog, whatsapp: WhatsAppGateway | None = None)` and pass to handlers via `session.info['whatsapp']` set in execute:

```python
session.info["whatsapp"] = self._whatsapp
```

`NullWhatsApp` records nothing.

- [ ] **Step 1: Write the failing test**

`tests/test_relacionamento.py`:

```python
from petia.agenda.seed import seed_servicos_piloto
from petia.agenda.tables import ensure_agenda_tables
from petia.cadastro.tables import ensure_cadastro_tables
from petia.command_os.bus import CommandBus
from petia.command_os.build import build_catalog
from petia.command_os.envelope import Actor, ActorKind, Command
from petia.identity.bootstrap import seed_piloto
from petia.identity.tables import ensure_identity_tables
from petia.os_banho.tables import ensure_os_tables
from petia.whatsapp.gateway import FakeWhatsAppGateway


def test_recomendar_e_notificar(db_session):
    ensure_identity_tables(db_session)
    ensure_cadastro_tables(db_session)
    ensure_agenda_tables(db_session)
    ensure_os_tables(db_session)
    tenant_id, unidade_id, usuario_id = seed_piloto(db_session, nome="Rel")
    ids = seed_servicos_piloto(db_session, tenant_id, unidade_id)
    wa = FakeWhatsAppGateway()
    bus = CommandBus(build_catalog(), whatsapp=wa)
    actor = Actor(ActorKind.USUARIO, str(usuario_id))
    tutor = bus.execute(
        db_session,
        Command(name="RegistrarTutor", tenant_id=tenant_id, unidade_id=unidade_id, payload={"nome": "Ana", "telefone": "11988887777"}, actor=actor),
    )
    pet = bus.execute(
        db_session,
        Command(name="RegistrarPet", tenant_id=tenant_id, unidade_id=unidade_id, payload={"nome": "Thor", "especie": "cao"}, actor=actor),
    )
    bus.execute(
        db_session,
        Command(
            name="VincularPet",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"tutor_id": tutor.data["tutor_id"], "pet_id": pet.data["pet_id"], "principal": True},
            actor=actor,
        ),
    )
    rec = bus.execute(
        db_session,
        Command(
            name="RecomendarProduto",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"pet_id": pet.data["pet_id"], "produto_nome": "Shampoo X"},
            actor=actor,
        ),
    )
    assert rec.ok
    assert any(e.name == "ProdutoRecomendado" for e in rec.events)
    ag = bus.execute(
        db_session,
        Command(
            name="Agendar",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={
                "pet_id": pet.data["pet_id"],
                "servico_id": ids["banho_id"],
                "profissional_id": ids["profissional_id"],
                "inicio": "2026-09-04T09:00:00-03:00",
            },
            actor=actor,
        ),
    )
    bus.execute(db_session, Command(name="ConfirmarPresenca", tenant_id=tenant_id, unidade_id=unidade_id, payload={"agendamento_id": ag.data["agendamento_id"]}, actor=actor))
    opened = bus.execute(db_session, Command(name="AbrirOS", tenant_id=tenant_id, unidade_id=unidade_id, payload={"agendamento_id": ag.data["agendamento_id"]}, actor=actor))
    ready = bus.execute(
        db_session,
        Command(name="NotificarPetPronto", tenant_id=tenant_id, unidade_id=unidade_id, payload={"os_id": opened.data["os_id"]}, actor=actor),
    )
    assert ready.ok
    assert any("pronto" in body.lower() or "Thor" in body for _, body in wa.sent)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_relacionamento.py -v`

Expected: FAIL missing commands or `CommandBus` signature.

- [ ] **Step 3: Write minimal implementation**

`src/petia/whatsapp/gateway.py`:

```python
class WhatsAppGateway:
    def send(self, to: str, body: str) -> None:
        raise NotImplementedError


class NullWhatsAppGateway(WhatsAppGateway):
    def send(self, to: str, body: str) -> None:
        return None


class FakeWhatsAppGateway(WhatsAppGateway):
    def __init__(self) -> None:
        self.sent: list[tuple[str, str]] = []

    def send(self, to: str, body: str) -> None:
        self.sent.append((to, body))
```

Modify `CommandBus.__init__` to accept `whatsapp: WhatsAppGateway | None = None` and default `NullWhatsAppGateway()`. At start of `execute`: `session.info["whatsapp"] = self._whatsapp`.

`src/petia/relacionamento/handlers.py`:

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.command_os.envelope import Command, CommandError, CommandResult, DomainEvent
from petia.identity.handlers import assert_unidade_in_tenant
from petia.whatsapp.gateway import WhatsAppGateway


def handle_recomendar_produto(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    sql = "SELECT id FROM pet WHERE id = :id AND tenant_id = :tenant_id"
    row = session.execute(text(sql), {"id": command.payload["pet_id"], "tenant_id": command.tenant_id}).first()
    if row is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="pet nao encontrado"),
            events=[],
            sql_statements=[sql],
        )
    return CommandResult(
        ok=True,
        data={"produto_nome": command.payload["produto_nome"]},
        error=None,
        events=[
            DomainEvent(
                name="ProdutoRecomendado",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=row[0],
                tutor_id=None,
                payload={"produto_nome": command.payload["produto_nome"]},
            )
        ],
        sql_statements=[sql],
    )


def handle_notificar_pet_pronto(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    sql = """
    SELECT os.pet_id, t.telefone, p.nome
    FROM ordem_servico os
    JOIN pet p ON p.id = os.pet_id AND p.tenant_id = os.tenant_id
    JOIN vinculo_tutor_pet v ON v.pet_id = os.pet_id AND v.tenant_id = os.tenant_id AND v.principal
    JOIN tutor t ON t.id = v.tutor_id AND t.tenant_id = os.tenant_id
    WHERE os.id = :os_id AND os.tenant_id = :tenant_id AND os.unidade_id = :unidade_id
    """
    row = session.execute(
        text(sql),
        {"os_id": command.payload["os_id"], "tenant_id": command.tenant_id, "unidade_id": command.unidade_id},
    ).first()
    if row is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="OS ou tutor principal nao encontrado"),
            events=[],
            sql_statements=[sql],
        )
    gateway: WhatsAppGateway = session.info["whatsapp"]
    body = f"{row[2]} ficou pronto. Pode retirar."
    gateway.send(str(row[1]), body)
    return CommandResult(
        ok=True,
        data={"os_id": command.payload["os_id"]},
        error=None,
        events=[
            DomainEvent(
                name="PetProntoNotificado",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=row[0],
                tutor_id=None,
                payload={"os_id": command.payload["os_id"]},
            )
        ],
        sql_statements=[sql],
    )
```

Register both in `build_catalog()`. Update `tests/test_command_bus.py` still constructs `CommandBus(catalog)` — default whatsapp keeps it working.

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest tests/test_relacionamento.py tests/test_command_bus.py tests/test_os.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/petia/whatsapp src/petia/relacionamento src/petia/command_os/bus.py src/petia/command_os/build.py tests/test_relacionamento.py
git commit -m "feat: add product suggestion and pet-ready WhatsApp notify commands"
```

---

### Task 10: EmitirCobranca, FakePsp, BaixarPagamento idempotente

**Files:**
- Create: `src/petia/cobranca/psp.py`
- Create: `src/petia/cobranca/tables.py`
- Create: `src/petia/cobranca/handlers.py`
- Create: `tests/test_cobranca.py`
- Modify: `CommandBus` to put psp on `session.info["psp"]`
- Modify: `build.py`

**Interfaces:**
- Consumes: OS id
- Produces:
  - `Psp.create_charge(amount_centavos: int, idempotency_key: str) -> PspCharge(txid: str, copia_cola: str, link: str)`
  - `FakePsp` / `DownPsp` raises `PspUnavailable`
  - `handle_emitir_cobranca` payload `{os_id}` — amount from `servico.preco_centavos` via agendamento; idempotency_key `f"{os_id}:pix"`; event `CobrancaEmitida`; error `PSP_INDISPONIVEL` without closing OS
  - `handle_baixar_pagamento` payload `{txid}` actor `SISTEMA`/`psp`; event `PagamentoBaixado`; duplicate txid no second event effect (unique txid)

- [ ] **Step 1: Write the failing test**

`tests/test_cobranca.py`:

```python
from petia.agenda.seed import seed_servicos_piloto
from petia.agenda.tables import ensure_agenda_tables
from petia.cadastro.tables import ensure_cadastro_tables
from petia.cobranca.psp import DownPsp, FakePsp
from petia.cobranca.tables import ensure_cobranca_tables
from petia.command_os.bus import CommandBus
from petia.command_os.build import build_catalog
from petia.command_os.envelope import Actor, ActorKind, Command
from petia.identity.bootstrap import seed_piloto
from petia.identity.tables import ensure_identity_tables
from petia.os_banho.tables import ensure_os_tables


def _os(db_session):
    ensure_identity_tables(db_session)
    ensure_cadastro_tables(db_session)
    ensure_agenda_tables(db_session)
    ensure_os_tables(db_session)
    ensure_cobranca_tables(db_session)
    tenant_id, unidade_id, usuario_id = seed_piloto(db_session, nome="Pix")
    ids = seed_servicos_piloto(db_session, tenant_id, unidade_id)
    psp = FakePsp()
    bus = CommandBus(build_catalog(), psp=psp)
    actor = Actor(ActorKind.USUARIO, str(usuario_id))
    pet = bus.execute(
        db_session,
        Command(name="RegistrarPet", tenant_id=tenant_id, unidade_id=unidade_id, payload={"nome": "Thor", "especie": "cao"}, actor=actor),
    )
    ag = bus.execute(
        db_session,
        Command(
            name="Agendar",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={
                "pet_id": pet.data["pet_id"],
                "servico_id": ids["banho_id"],
                "profissional_id": ids["profissional_id"],
                "inicio": "2026-09-05T09:00:00-03:00",
            },
            actor=actor,
        ),
    )
    bus.execute(db_session, Command(name="ConfirmarPresenca", tenant_id=tenant_id, unidade_id=unidade_id, payload={"agendamento_id": ag.data["agendamento_id"]}, actor=actor))
    opened = bus.execute(db_session, Command(name="AbrirOS", tenant_id=tenant_id, unidade_id=unidade_id, payload={"agendamento_id": ag.data["agendamento_id"]}, actor=actor))
    return tenant_id, unidade_id, actor, bus, opened.data["os_id"], psp


def test_emitir_e_baixar_idempotente(db_session):
    tenant_id, unidade_id, actor, bus, os_id, psp = _os(db_session)
    cob = bus.execute(
        db_session,
        Command(
            name="EmitirCobranca",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"os_id": os_id},
            actor=actor,
            idempotency_key=f"{os_id}:pix",
        ),
    )
    assert cob.ok
    txid = cob.data["txid"]
    sistema = Actor(ActorKind.SISTEMA, "psp")
    first = bus.execute(
        db_session,
        Command(name="BaixarPagamento", tenant_id=tenant_id, unidade_id=unidade_id, payload={"txid": txid}, actor=sistema, idempotency_key=txid),
    )
    second = bus.execute(
        db_session,
        Command(name="BaixarPagamento", tenant_id=tenant_id, unidade_id=unidade_id, payload={"txid": txid}, actor=sistema, idempotency_key=txid),
    )
    assert first.ok and second.ok
    assert first.data == second.data


def test_psp_fora_nao_trava_os(db_session):
    ensure_identity_tables(db_session)
    ensure_cadastro_tables(db_session)
    ensure_agenda_tables(db_session)
    ensure_os_tables(db_session)
    ensure_cobranca_tables(db_session)
    tenant_id, unidade_id, usuario_id = seed_piloto(db_session, nome="PixDown")
    ids = seed_servicos_piloto(db_session, tenant_id, unidade_id)
    bus = CommandBus(build_catalog(), psp=DownPsp())
    actor = Actor(ActorKind.USUARIO, str(usuario_id))
    pet = bus.execute(
        db_session,
        Command(name="RegistrarPet", tenant_id=tenant_id, unidade_id=unidade_id, payload={"nome": "Thor", "especie": "cao"}, actor=actor),
    )
    ag = bus.execute(
        db_session,
        Command(
            name="Agendar",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={
                "pet_id": pet.data["pet_id"],
                "servico_id": ids["banho_id"],
                "profissional_id": ids["profissional_id"],
                "inicio": "2026-09-05T11:00:00-03:00",
            },
            actor=actor,
        ),
    )
    bus.execute(db_session, Command(name="ConfirmarPresenca", tenant_id=tenant_id, unidade_id=unidade_id, payload={"agendamento_id": ag.data["agendamento_id"]}, actor=actor))
    opened = bus.execute(db_session, Command(name="AbrirOS", tenant_id=tenant_id, unidade_id=unidade_id, payload={"agendamento_id": ag.data["agendamento_id"]}, actor=actor))
    cob = bus.execute(
        db_session,
        Command(name="EmitirCobranca", tenant_id=tenant_id, unidade_id=unidade_id, payload={"os_id": opened.data["os_id"]}, actor=actor),
    )
    assert cob.ok is False
    assert cob.error.code == "PSP_INDISPONIVEL"
    from sqlalchemy import text
    status = db_session.execute(text("SELECT status FROM ordem_servico WHERE id = :id"), {"id": opened.data["os_id"]}).scalar()
    assert status == "aberta"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_cobranca.py -v`

Expected: FAIL missing cobranca.

- [ ] **Step 3: Write minimal implementation**

`src/petia/cobranca/psp.py`:

```python
from dataclasses import dataclass
from uuid import uuid4


class PspUnavailable(Exception):
    pass


@dataclass(frozen=True)
class PspCharge:
    txid: str
    copia_cola: str
    link: str


class Psp:
    def create_charge(self, amount_centavos: int, idempotency_key: str) -> PspCharge:
        raise NotImplementedError


class FakePsp(Psp):
    def __init__(self) -> None:
        self._by_key: dict[str, PspCharge] = {}

    def create_charge(self, amount_centavos: int, idempotency_key: str) -> PspCharge:
        if idempotency_key in self._by_key:
            return self._by_key[idempotency_key]
        charge = PspCharge(txid=f"tx_{uuid4().hex[:12]}", copia_cola="0002012658", link="https://pix.local/pay")
        self._by_key[idempotency_key] = charge
        return charge


class DownPsp(Psp):
    def create_charge(self, amount_centavos: int, idempotency_key: str) -> PspCharge:
        raise PspUnavailable("psp down")
```

`src/petia/cobranca/tables.py`:

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

DDL = """
CREATE TABLE IF NOT EXISTS cobranca (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  unidade_id UUID NOT NULL,
  os_id UUID NOT NULL,
  txid TEXT UNIQUE,
  copia_cola TEXT,
  link TEXT,
  amount_centavos INTEGER NOT NULL,
  status TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE IF NOT EXISTS pagamento (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  cobranca_id UUID NOT NULL,
  txid TEXT NOT NULL UNIQUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
"""


def ensure_cobranca_tables(session: Session) -> None:
    for stmt in DDL.strip().split(";"):
        if stmt.strip():
            session.execute(text(stmt))
```

`src/petia/cobranca/handlers.py`:

```python
from uuid import uuid4

from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.cobranca.psp import Psp, PspUnavailable
from petia.command_os.envelope import Command, CommandError, CommandResult, DomainEvent
from petia.identity.handlers import assert_unidade_in_tenant


def handle_emitir_cobranca(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    price_sql = """
    SELECT s.preco_centavos, os.pet_id
    FROM ordem_servico os
    JOIN agendamento a ON a.id = os.agendamento_id AND a.tenant_id = os.tenant_id
    JOIN servico s ON s.id = a.servico_id AND s.tenant_id = os.tenant_id
    WHERE os.id = :os_id AND os.tenant_id = :tenant_id AND os.unidade_id = :unidade_id
    """
    priced = session.execute(
        text(price_sql),
        {"os_id": command.payload["os_id"], "tenant_id": command.tenant_id, "unidade_id": command.unidade_id},
    ).first()
    if priced is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="OS nao encontrada"),
            events=[],
            sql_statements=[price_sql],
        )
    psp: Psp = session.info["psp"]
    key = command.idempotency_key or f"{command.payload['os_id']}:pix"
    try:
        charge = psp.create_charge(int(priced[0]), key)
    except PspUnavailable:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="PSP_INDISPONIVEL", message="nao foi possivel gerar PIX"),
            events=[],
            sql_statements=[price_sql],
        )
    cobranca_id = uuid4()
    sql = """
    INSERT INTO cobranca (id, tenant_id, unidade_id, os_id, txid, copia_cola, link, amount_centavos, status)
    VALUES (:id, :tenant_id, :unidade_id, :os_id, :txid, :copia_cola, :link, :amount, 'emitida')
    """
    session.execute(
        text(sql),
        {
            "id": cobranca_id,
            "tenant_id": command.tenant_id,
            "unidade_id": command.unidade_id,
            "os_id": command.payload["os_id"],
            "txid": charge.txid,
            "copia_cola": charge.copia_cola,
            "link": charge.link,
            "amount": int(priced[0]),
        },
    )
    return CommandResult(
        ok=True,
        data={"cobranca_id": str(cobranca_id), "txid": charge.txid, "copia_cola": charge.copia_cola, "link": charge.link},
        error=None,
        events=[
            DomainEvent(
                name="CobrancaEmitida",
                tenant_id=command.tenant_id,
                unidade_id=command.unidade_id,
                pet_id=priced[1],
                tutor_id=None,
                payload={"txid": charge.txid},
            )
        ],
        sql_statements=[price_sql, sql],
    )


def handle_baixar_pagamento(session: Session, command: Command) -> CommandResult:
    lookup = """
    SELECT id, tenant_id, unidade_id, os_id FROM cobranca WHERE txid = :txid AND tenant_id = :tenant_id
    """
    row = session.execute(text(lookup), {"txid": command.payload["txid"], "tenant_id": command.tenant_id}).mappings().first()
    if row is None:
        return CommandResult(
            ok=False,
            data={},
            error=CommandError(code="SERVICO_INEXISTENTE", message="cobranca nao encontrada"),
            events=[],
            sql_statements=[lookup],
        )
    existing = session.execute(
        text("SELECT id FROM pagamento WHERE txid = :txid"),
        {"txid": command.payload["txid"]},
    ).first()
    if existing:
        return CommandResult(
            ok=True,
            data={"pagamento_id": str(existing[0]), "txid": command.payload["txid"]},
            error=None,
            events=[],
            sql_statements=[lookup],
        )
    pagamento_id = uuid4()
    ins = """
    INSERT INTO pagamento (id, tenant_id, cobranca_id, txid)
    VALUES (:id, :tenant_id, :cobranca_id, :txid)
    """
    upd = "UPDATE cobranca SET status = 'paga' WHERE id = :id AND tenant_id = :tenant_id"
    session.execute(
        text(ins),
        {"id": pagamento_id, "tenant_id": command.tenant_id, "cobranca_id": row["id"], "txid": command.payload["txid"]},
    )
    session.execute(text(upd), {"id": row["id"], "tenant_id": command.tenant_id})
    pet = session.execute(
        text("SELECT pet_id FROM ordem_servico WHERE id = :id AND tenant_id = :tenant_id"),
        {"id": row["os_id"], "tenant_id": command.tenant_id},
    ).first()
    return CommandResult(
        ok=True,
        data={"pagamento_id": str(pagamento_id), "txid": command.payload["txid"]},
        error=None,
        events=[
            DomainEvent(
                name="PagamentoBaixado",
                tenant_id=command.tenant_id,
                unidade_id=row["unidade_id"],
                pet_id=None if pet is None else pet[0],
                tutor_id=None,
                payload={"txid": command.payload["txid"]},
            )
        ],
        sql_statements=[lookup, ins, upd],
    )
```

`CommandBus.__init__(self, catalog, whatsapp=None, psp=None)` store `_psp` default `FakePsp()` for tests? Default a `FakePsp` would accidentally create PIX in prod. Default a `Psp` base that raises. For unit tests pass FakePsp. For shop later inject FakePsp until a real adapter exists.

Default: `self._psp = psp` required in tests; in `execute` `session.info["psp"] = self._psp`. Tests that don't pass psp: `EmitirCobranca` unused. `CommandBus(catalog)` sets `psp=None`; `handle_emitir_cobranca` would fail. Keep `psp: Psp | None = None` and only set session.info if not None.

Register commands in `build_catalog()`.

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest tests/test_cobranca.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/petia/cobranca src/petia/command_os/bus.py src/petia/command_os/build.py tests/test_cobranca.py
git commit -m "feat: add PIX charge command and idempotent payment webhook handler"
```

---

### Task 11: Protocolo de confirmação no chat

**Files:**
- Create: `src/petia/copilot/confirmation.py`
- Create: `src/petia/copilot/tables.py`
- Create: `tests/test_copilot.py`
- Modify: `build.py` only if `ConfigurarNivelAcao` lands in Task 13 — here just confirmation store

**Interfaces:**
- Consumes: `Command`, `CommandBus`
- Produces:
  - `ConfirmationStore.propose(session, conversation_id, command, now, ttl_seconds=300) -> PendingConfirmation(id, prompt_text)`
  - `ConfirmationStore.confirm(session, conversation_id, pending_id, now) -> Command | CommandError(code=CONFIRMATION_EXPIRED)`
  - `confirm` twice with same pending_id: second returns expired/used error `CONFIRMATION_EXPIRED`
  - Default v1: every chat-originated command goes through propose except `BaixarPagamento`

- [ ] **Step 1: Write the failing test**

```python
from datetime import datetime, timedelta, timezone
from uuid import uuid4

from petia.cadastro.tables import ensure_cadastro_tables
from petia.command_os.bus import CommandBus
from petia.command_os.build import build_catalog
from petia.command_os.envelope import Actor, ActorKind, Command
from petia.copilot.confirmation import ConfirmationStore
from petia.copilot.tables import ensure_copilot_tables
from petia.identity.bootstrap import seed_piloto
from petia.identity.tables import ensure_identity_tables


def test_confirmacao_expirada_nao_grava(db_session):
    ensure_identity_tables(db_session)
    ensure_cadastro_tables(db_session)
    ensure_copilot_tables(db_session)
    tenant_id, unidade_id, usuario_id = seed_piloto(db_session, nome="Conf")
    bus = CommandBus(build_catalog())
    store = ConfirmationStore()
    now = datetime(2026, 9, 1, 12, 0, tzinfo=timezone.utc)
    command = Command(
        name="RegistrarPet",
        tenant_id=tenant_id,
        unidade_id=unidade_id,
        payload={"nome": "Thor", "especie": "cao"},
        actor=Actor(ActorKind.TUTOR_WHATSAPP, "5511999"),
    )
    pending = store.propose(db_session, conversation_id="wa:5511999", command=command, now=now, ttl_seconds=60)
    late = now + timedelta(seconds=120)
    outcome = store.confirm(db_session, conversation_id="wa:5511999", pending_id=pending.id, now=late)
    assert outcome.error is not None
    assert outcome.error.code == "CONFIRMATION_EXPIRED"
    result = bus.execute(db_session, command)
    # expired path must NOT auto-execute; this line only shows bus still works from UI
    assert result.ok


def test_sim_duplicado_nao_duplica(db_session):
    ensure_identity_tables(db_session)
    ensure_cadastro_tables(db_session)
    ensure_copilot_tables(db_session)
    tenant_id, unidade_id, _ = seed_piloto(db_session, nome="Dup")
    bus = CommandBus(build_catalog())
    store = ConfirmationStore()
    now = datetime(2026, 9, 1, 12, 0, tzinfo=timezone.utc)
    command = Command(
        name="RegistrarPet",
        tenant_id=tenant_id,
        unidade_id=unidade_id,
        payload={"nome": "Thor", "especie": "cao"},
        actor=Actor(ActorKind.TUTOR_WHATSAPP, "5511888"),
        idempotency_key="wa:pet:thor",
    )
    pending = store.propose(db_session, conversation_id="wa:5511888", command=command, now=now, ttl_seconds=300)
    first_cmd = store.confirm(db_session, conversation_id="wa:5511888", pending_id=pending.id, now=now)
    assert first_cmd.command is not None
    r1 = bus.execute(db_session, first_cmd.command)
    second = store.confirm(db_session, conversation_id="wa:5511888", pending_id=pending.id, now=now)
    assert second.error is not None
    assert second.error.code == "CONFIRMATION_EXPIRED"
    assert r1.ok
```

Define `ConfirmOutcome` with `.command` and `.error`.

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_copilot.py -v`

Expected: FAIL missing confirmation module.

- [ ] **Step 3: Write minimal implementation**

`src/petia/copilot/tables.py`:

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

DDL = """
CREATE TABLE IF NOT EXISTS pending_confirmation (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  conversation_id TEXT NOT NULL,
  command_json JSONB NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  used BOOLEAN NOT NULL DEFAULT false
);
"""


def ensure_copilot_tables(session: Session) -> None:
    for stmt in DDL.strip().split(";"):
        if stmt.strip():
            session.execute(text(stmt))
```

`src/petia/copilot/confirmation.py`:

```python
import json
from dataclasses import dataclass
from datetime import datetime, timedelta
from uuid import UUID, uuid4

from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.command_os.envelope import Actor, ActorKind, Command, CommandError
from petia.copilot.tables import ensure_copilot_tables


@dataclass
class PendingConfirmation:
    id: UUID
    prompt_text: str


@dataclass
class ConfirmOutcome:
    command: Command | None
    error: CommandError | None


def _command_to_json(command: Command) -> dict:
    return {
        "name": command.name,
        "tenant_id": str(command.tenant_id),
        "unidade_id": None if command.unidade_id is None else str(command.unidade_id),
        "payload": command.payload,
        "actor_kind": command.actor.kind.value,
        "actor_id": command.actor.id,
        "idempotency_key": command.idempotency_key,
        "command_id": str(command.command_id),
    }


def _command_from_json(data: dict) -> Command:
    return Command(
        name=data["name"],
        tenant_id=UUID(data["tenant_id"]),
        unidade_id=None if data["unidade_id"] is None else UUID(data["unidade_id"]),
        payload=data["payload"],
        actor=Actor(ActorKind(data["actor_kind"]), data["actor_id"]),
        idempotency_key=data["idempotency_key"],
        command_id=UUID(data["command_id"]),
    )


class ConfirmationStore:
    def propose(
        self,
        session: Session,
        conversation_id: str,
        command: Command,
        now: datetime,
        ttl_seconds: int = 300,
    ) -> PendingConfirmation:
        ensure_copilot_tables(session)
        pending_id = uuid4()
        expires = now + timedelta(seconds=ttl_seconds)
        session.execute(
            text(
                """
                INSERT INTO pending_confirmation (id, tenant_id, conversation_id, command_json, expires_at, used)
                VALUES (:id, :tenant_id, :conversation_id, CAST(:command_json AS JSONB), :expires_at, false)
                """
            ),
            {
                "id": pending_id,
                "tenant_id": command.tenant_id,
                "conversation_id": conversation_id,
                "command_json": json.dumps(_command_to_json(command)),
                "expires_at": expires,
            },
        )
        prompt = f"Confirmo {command.name}?"
        if command.name == "Agendar":
            prompt = f"Confirmo o banho/serviço em {command.payload.get('inicio')}?"
        return PendingConfirmation(id=pending_id, prompt_text=prompt)

    def confirm(self, session: Session, conversation_id: str, pending_id: UUID, now: datetime) -> ConfirmOutcome:
        ensure_copilot_tables(session)
        row = session.execute(
            text(
                """
                SELECT command_json, expires_at, used, conversation_id
                FROM pending_confirmation
                WHERE id = :id
                """
            ),
            {"id": pending_id},
        ).mappings().first()
        if row is None or row["used"] or row["conversation_id"] != conversation_id or row["expires_at"] < now:
            return ConfirmOutcome(
                command=None,
                error=CommandError(code="CONFIRMATION_EXPIRED", message="confirmacao expirada ou ja usada"),
            )
        session.execute(text("UPDATE pending_confirmation SET used = true WHERE id = :id"), {"id": pending_id})
        return ConfirmOutcome(command=_command_from_json(row["command_json"]), error=None)
```

For Agendar prompt, Task 12 can specialize “Confirmo o banho do Thor sábado 11:30?” using pet name lookup. In this task, if payload has pet name missing, use inicio. Improve propose() to accept optional `prompt_text` argument defaulting as above.

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest tests/test_copilot.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/petia/copilot tests/test_copilot.py
git commit -m "feat: add chat confirmation protocol with expiry and single use"
```

---

### Task 12: Orquestrador + FakeAgent (mesmo efeito que a tela)

**Files:**
- Create: `src/petia/copilot/agent.py`
- Create: `src/petia/copilot/orchestrator.py`
- Modify: `tests/test_copilot.py`
- Modify: `tests/test_isolamento.py` is not the agent-down test — add `test_agente_caido_tela_funciona` here

**Interfaces:**
- Consumes: `Catalog`, `ConfirmationStore`, `CommandBus`, queries for pets of phone
- Produces:
  - `AgentIntent(command_name: str | None, arguments: dict, clarification_question: str | None, refusal_reason: str | None)`
  - `Agent.interpret(message: str, context: dict) -> AgentIntent`
  - `FakeAgent` scripted; `DownAgent` raises `AgentUnavailable`
  - `Orchestrator.handle_incoming(session, *, tenant_id, unidade_id, conversation_id, wa_id, text, now) -> OrchestratorReply(body: str, executed: bool)`
  - Flow: if text is `sim`/`não` and pending exists → confirm/cancel. Else agent. If refusal, body = reason. If clarification, ask. If intent, `propose` then ask prompt. Never execute without confirm.
  - Unknown tutor (no tutor with telefone = wa_id): do not `RegistrarTutor` without confirmation; reply asking name or that recepção will help (`DADOS_INSUFICIENTES` as clarification, not silent create)
  - Fora do catálogo: agent returns `refusal_reason`; orchestrator sends it; no SQL write

- [ ] **Step 1: Write the failing test**

Append to `tests/test_copilot.py`:

```python
from petia.agenda.seed import seed_servicos_piloto
from petia.agenda.tables import ensure_agenda_tables
from petia.cadastro.tables import ensure_cadastro_tables
from petia.copilot.agent import DownAgent, FakeAgent, AgentIntent
from petia.copilot.confirmation import ConfirmationStore
from petia.copilot.orchestrator import Orchestrator
from petia.copilot.tables import ensure_copilot_tables
from petia.timeline.tables import ensure_timeline_tables


def test_chat_e_tela_mesmo_efeito(db_session):
    ensure_identity_tables(db_session)
    ensure_cadastro_tables(db_session)
    ensure_agenda_tables(db_session)
    ensure_timeline_tables(db_session)
    ensure_copilot_tables(db_session)
    tenant_id, unidade_id, usuario_id = seed_piloto(db_session, nome="Eq")
    ids = seed_servicos_piloto(db_session, tenant_id, unidade_id)
    bus = CommandBus(build_catalog())
    actor_ui = Actor(ActorKind.USUARIO, str(usuario_id))
    pet_ui = bus.execute(
        db_session,
        Command(name="RegistrarPet", tenant_id=tenant_id, unidade_id=unidade_id, payload={"nome": "Luna", "especie": "cao"}, actor=actor_ui),
    )
    agent = FakeAgent(
        AgentIntent(
            command_name="RegistrarPet",
            arguments={"nome": "Luna2", "especie": "cao"},
            clarification_question=None,
            refusal_reason=None,
        )
    )
    orch = Orchestrator(bus=bus, store=ConfirmationStore(), agent=agent)
    now = datetime(2026, 9, 1, 12, 0, tzinfo=timezone.utc)
    reply = orch.handle_incoming(
        db_session,
        tenant_id=tenant_id,
        unidade_id=unidade_id,
        conversation_id="wa:1",
        wa_id="5511",
        text="cadastrar pet Luna2",
        now=now,
    )
    assert "Confirmo" in reply.body
    yes = orch.handle_incoming(
        db_session,
        tenant_id=tenant_id,
        unidade_id=unidade_id,
        conversation_id="wa:1",
        wa_id="5511",
        text="sim",
        now=now,
    )
    assert yes.executed is True
    from sqlalchemy import text as sqltext
    count = db_session.execute(
        sqltext("SELECT count(*) FROM pet WHERE tenant_id = :t"),
        {"t": tenant_id},
    ).scalar()
    assert count == 2
    assert pet_ui.ok


def test_fora_do_catalogo_recusa(db_session):
    ensure_identity_tables(db_session)
    ensure_copilot_tables(db_session)
    tenant_id, unidade_id, _ = seed_piloto(db_session, nome="NFe")
    bus = CommandBus(build_catalog())
    agent = FakeAgent(AgentIntent(command_name=None, arguments={}, clarification_question=None, refusal_reason="nao emito NFe"))
    orch = Orchestrator(bus=bus, store=ConfirmationStore(), agent=agent)
    now = datetime(2026, 9, 1, 12, 0, tzinfo=timezone.utc)
    reply = orch.handle_incoming(
        db_session,
        tenant_id=tenant_id,
        unidade_id=unidade_id,
        conversation_id="wa:2",
        wa_id="5511",
        text="emite NFe",
        now=now,
    )
    assert reply.executed is False
    assert "NFe" in reply.body


def test_agente_caido_tela_funciona(db_session):
    ensure_identity_tables(db_session)
    ensure_cadastro_tables(db_session)
    ensure_copilot_tables(db_session)
    tenant_id, unidade_id, usuario_id = seed_piloto(db_session, nome="Down")
    bus = CommandBus(build_catalog())
    orch = Orchestrator(bus=bus, store=ConfirmationStore(), agent=DownAgent())
    now = datetime(2026, 9, 1, 12, 0, tzinfo=timezone.utc)
    reply = orch.handle_incoming(
        db_session,
        tenant_id=tenant_id,
        unidade_id=unidade_id,
        conversation_id="wa:3",
        wa_id="5511",
        text="oi",
        now=now,
    )
    assert reply.executed is False
    ui = bus.execute(
        db_session,
        Command(
            name="RegistrarPet",
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload={"nome": "Thor", "especie": "cao"},
            actor=Actor(ActorKind.USUARIO, str(usuario_id)),
        ),
    )
    assert ui.ok
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_copilot.py -v`

Expected: FAIL missing Orchestrator.

- [ ] **Step 3: Write minimal implementation**

`src/petia/copilot/agent.py`:

```python
from dataclasses import dataclass


@dataclass
class AgentIntent:
    command_name: str | None
    arguments: dict
    clarification_question: str | None
    refusal_reason: str | None


class AgentUnavailable(Exception):
    pass


class Agent:
    def interpret(self, message: str, context: dict) -> AgentIntent:
        raise NotImplementedError


class FakeAgent(Agent):
    def __init__(self, intent: AgentIntent) -> None:
        self.intent = intent

    def interpret(self, message: str, context: dict) -> AgentIntent:
        return self.intent


class DownAgent(Agent):
    def interpret(self, message: str, context: dict) -> AgentIntent:
        raise AgentUnavailable("agent down")
```

`src/petia/copilot/orchestrator.py`:

```python
from dataclasses import dataclass
from datetime import datetime
from uuid import UUID

from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.command_os.bus import CommandBus
from petia.command_os.build import build_catalog
from petia.command_os.envelope import Actor, ActorKind, Command
from petia.copilot.agent import Agent, AgentUnavailable
from petia.copilot.confirmation import ConfirmationStore


@dataclass
class OrchestratorReply:
    body: str
    executed: bool


class Orchestrator:
    def __init__(self, bus: CommandBus, store: ConfirmationStore, agent: Agent) -> None:
        self._bus = bus
        self._store = store
        self._agent = agent

    def handle_incoming(
        self,
        session: Session,
        *,
        tenant_id: UUID,
        unidade_id: UUID,
        conversation_id: str,
        wa_id: str,
        text: str,
        now: datetime,
    ) -> OrchestratorReply:
        normalized = text.strip().lower()
        pending = session.execute(
            text(
                """
                SELECT id FROM pending_confirmation
                WHERE conversation_id = :cid AND used = false
                ORDER BY expires_at DESC
                LIMIT 1
                """
            ),
            {"cid": conversation_id},
        ).first()
        if pending and normalized in {"sim", "s", "confirmo", "yes"}:
            outcome = self._store.confirm(session, conversation_id, pending[0], now)
            if outcome.error:
                return OrchestratorReply(body="Confirmacao expirada. Pode repetir o pedido.", executed=False)
            result = self._bus.execute(session, outcome.command)
            if result.ok:
                return OrchestratorReply(body="Pronto, registrado.", executed=True)
            return OrchestratorReply(body=result.error.message, executed=False)
        if pending and normalized in {"nao", "não", "n"}:
            session.execute(text("UPDATE pending_confirmation SET used = true WHERE id = :id"), {"id": pending[0]})
            return OrchestratorReply(body="Ok, nao gravei.", executed=False)

        catalog = build_catalog()
        try:
            intent = self._agent.interpret(
                text,
                {
                    "tenant_id": str(tenant_id),
                    "unidade_id": str(unidade_id),
                    "commands": list(catalog._handlers.keys()),
                },
            )
        except AgentUnavailable:
            return OrchestratorReply(body="Estou offline no WhatsApp. A loja consegue te atender na recepcao.", executed=False)

        if intent.refusal_reason:
            return OrchestratorReply(body=intent.refusal_reason, executed=False)
        if intent.clarification_question or not intent.command_name:
            return OrchestratorReply(body=intent.clarification_question or "Pode detalhar?", executed=False)
        if catalog.get(intent.command_name) is None:
            return OrchestratorReply(body="Nao consigo fazer isso por aqui.", executed=False)

        command = Command(
            name=intent.command_name,
            tenant_id=tenant_id,
            unidade_id=unidade_id,
            payload=intent.arguments,
            actor=Actor(ActorKind.TUTOR_WHATSAPP, wa_id),
        )
        pending_obj = self._store.propose(session, conversation_id, command, now)
        return OrchestratorReply(body=pending_obj.prompt_text, executed=False)
```

Exposing `catalog._handlers` is ugly. Add `Catalog.names() -> list[str]` in `catalog.py`:

```python
def names(self) -> list[str]:
    return list(self._handlers.keys())
```

Use `catalog.names()` in the orchestrator instead of `_handlers`.

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest tests/test_copilot.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/petia/copilot src/petia/command_os/catalog.py tests/test_copilot.py
git commit -m "feat: add agent orchestrator with confirm-then-execute write path"
```

---

### Task 13: ConfigurarNivelAcao, HTTP API, webhook PSP, telas da loja

**Files:**
- Create: `src/petia/copilot/niveis.py` handlers for `ConfigurarNivelAcao`
- Create: `src/petia/runtime/app.py`
- Create: `src/petia/web/routes_commands.py`
- Create: `src/petia/web/routes_queries.py`
- Create: `src/petia/web/routes_webhooks.py`
- Create: `src/petia/web/routes_shop.py`
- Create: `src/petia/web/templates/agenda.html`
- Create: `src/petia/web/templates/os.html`
- Create: `src/petia/web/templates/pet.html`
- Create: `tests/test_shop_loop.py`
- Modify: `build.py`

**Interfaces:**
- Consumes: all commands, queries, Orchestrator, FakePsp
- Produces:
  - `POST /comandos` body `{name, tenant_id, unidade_id, payload, actor_id}` → bus.execute (UI, no confirmation)
  - `POST /whatsapp/inbound` → orchestrator (with confirmation)
  - `POST /webhooks/psp` `{txid, tenant_id, unidade_id}` → `BaixarPagamento`
  - `GET /unidades/{unidade_id}/agenda?dia=YYYY-MM-DD&tenant_id=`
  - `GET /pets/{pet_id}/timeline?tenant_id=`
  - HTML `GET /loja/agenda`, `GET /loja/os/{id}`, `GET /loja/pets/{id}` using the same query functions
  - `ConfigurarNivelAcao` payload `{comando, exige_confirmacao: bool}` stored in `acao_nivel`; Orchestrator consults it — v1 default true; if false, execute immediately (still no LLM SQL)

- [ ] **Step 1: Write the failing test**

`tests/test_shop_loop.py`:

```python
from fastapi.testclient import TestClient

from petia.agenda.seed import seed_servicos_piloto
from petia.agenda.tables import ensure_agenda_tables
from petia.cadastro.tables import ensure_cadastro_tables
from petia.cobranca.psp import FakePsp
from petia.cobranca.tables import ensure_cobranca_tables
from petia.command_os.tables import ensure_comando_log
from petia.copilot.tables import ensure_copilot_tables
from petia.identity.bootstrap import seed_piloto
from petia.identity.tables import ensure_identity_tables
from petia.os_banho.tables import ensure_os_tables
from petia.runtime.app import create_app
from petia.timeline.tables import ensure_timeline_tables


def test_http_ui_command_and_webhook(db_session):
    ensure_identity_tables(db_session)
    ensure_cadastro_tables(db_session)
    ensure_agenda_tables(db_session)
    ensure_os_tables(db_session)
    ensure_cobranca_tables(db_session)
    ensure_timeline_tables(db_session)
    ensure_copilot_tables(db_session)
    ensure_comando_log(db_session)
    tenant_id, unidade_id, usuario_id = seed_piloto(db_session, nome="Http")
    ids = seed_servicos_piloto(db_session, tenant_id, unidade_id)
    app = create_app(session_factory=lambda: db_session, psp=FakePsp())
    client = TestClient(app)
    pet = client.post(
        "/comandos",
        json={
            "name": "RegistrarPet",
            "tenant_id": str(tenant_id),
            "unidade_id": str(unidade_id),
            "actor_id": str(usuario_id),
            "payload": {"nome": "Thor", "especie": "cao"},
        },
    )
    assert pet.status_code == 200
    assert pet.json()["ok"] is True
    agenda = client.get(f"/unidades/{unidade_id}/agenda", params={"tenant_id": str(tenant_id), "dia": "2026-09-06"})
    assert agenda.status_code == 200
    html = client.get("/loja/agenda", params={"tenant_id": str(tenant_id), "unidade_id": str(unidade_id), "dia": "2026-09-06"})
    assert html.status_code == 200
    assert "Agenda" in html.text
```

Note: `create_app(session_factory=lambda: db_session)` must not close the pytest session. Implement `create_app` to yield the injected session without calling `session.close()` when a factory is provided for tests.

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_shop_loop.py -v`

Expected: FAIL missing FastAPI app.

- [ ] **Step 3: Write minimal implementation**

`src/petia/copilot/niveis.py`:

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

from petia.command_os.envelope import Command, CommandResult
from petia.identity.handlers import assert_unidade_in_tenant


def ensure_niveis(session: Session) -> None:
    session.execute(
        text(
            """
            CREATE TABLE IF NOT EXISTS acao_nivel (
              tenant_id UUID NOT NULL,
              unidade_id UUID NOT NULL,
              comando TEXT NOT NULL,
              exige_confirmacao BOOLEAN NOT NULL,
              PRIMARY KEY (tenant_id, unidade_id, comando)
            )
            """
        )
    )


def handle_configurar_nivel_acao(session: Session, command: Command) -> CommandResult:
    err = assert_unidade_in_tenant(session, command)
    if err:
        return CommandResult(ok=False, data={}, error=err, events=[], sql_statements=[])
    ensure_niveis(session)
    sql = """
    INSERT INTO acao_nivel (tenant_id, unidade_id, comando, exige_confirmacao)
    VALUES (:tenant_id, :unidade_id, :comando, :exige)
    ON CONFLICT (tenant_id, unidade_id, comando)
    DO UPDATE SET exige_confirmacao = EXCLUDED.exige_confirmacao
    """
    session.execute(
        text(sql),
        {
            "tenant_id": command.tenant_id,
            "unidade_id": command.unidade_id,
            "comando": command.payload["comando"],
            "exige": command.payload["exige_confirmacao"],
        },
    )
    return CommandResult(ok=True, data={}, error=None, events=[], sql_statements=[sql])


def exige_confirmacao(session: Session, tenant_id, unidade_id, comando: str) -> bool:
    ensure_niveis(session)
    row = session.execute(
        text(
            """
            SELECT exige_confirmacao FROM acao_nivel
            WHERE tenant_id = :t AND unidade_id = :u AND comando = :c
            """
        ),
        {"t": tenant_id, "u": unidade_id, "c": comando},
    ).first()
    if row is None:
        return True
    return bool(row[0])
```

In `Orchestrator.handle_incoming`, after building `command` from intent, if `not exige_confirmacao(...)` then `self._bus.execute` immediately; else propose. Default remains True.

`src/petia/runtime/app.py` creates FastAPI, registers routers, injects bus built from `build_catalog()`, FakePsp, FakeWhatsApp, ConfirmationStore, FakeAgent until env `PETIA_AGENT=live` (do not implement live LLM in this fatia beyond the port).

`src/petia/web/routes_commands.py`:

```python
from uuid import UUID

from fastapi import APIRouter, Request
from pydantic import BaseModel

from petia.command_os.envelope import Actor, ActorKind, Command

router = APIRouter()


class CommandIn(BaseModel):
    name: str
    tenant_id: UUID
    unidade_id: UUID | None = None
    actor_id: str
    payload: dict
    idempotency_key: str | None = None


@router.post("/comandos")
def post_comando(body: CommandIn, request: Request):
    bus = request.app.state.bus
    session = request.app.state.session_factory()
    result = bus.execute(
        session,
        Command(
            name=body.name,
            tenant_id=body.tenant_id,
            unidade_id=body.unidade_id,
            payload=body.payload,
            actor=Actor(ActorKind.USUARIO, body.actor_id),
            idempotency_key=body.idempotency_key,
        ),
    )
    return {
        "ok": result.ok,
        "data": result.data,
        "error": None if result.error is None else {"code": result.error.code, "message": result.error.message},
        "sql_statements": result.sql_statements,
    }
```

`routes_webhooks.py`: `POST /webhooks/psp` executes `BaixarPagamento` with `Actor(SISTEMA, "psp")` and `idempotency_key=txid`.

`routes_queries.py`: wrap `agenda_do_dia` and `timeline_do_pet`.

`templates/agenda.html`:

```html
<!doctype html>
<title>Agenda</title>
<h1>Agenda</h1>
<ul>
{% for item in items %}
  <li>{{ item.inicio }} — {{ item.pet_nome }} — {{ item.servico_nome }}</li>
{% endfor %}
</ul>
```

`create_app` mounts Jinja `GET /loja/agenda`.

Register `ConfigurarNivelAcao` in `build_catalog()`.

Write **full** `src/petia/command_os/build.py` listing every command registered in Tasks 2–13:

`PingUnidade`, `RegistrarTutor`, `RegistrarPet`, `VincularPet`, `Agendar`, `Reagendar`, `CancelarAgendamento`, `ConfirmarAgendamento`, `ConfirmarPresenca`, `AbrirOS`, `RegistrarEntradaOS`, `AtualizarExecucaoOS`, `RegistrarSaidaOS`, `RecomendarProduto`, `NotificarPetPronto`, `EmitirCobranca`, `BaixarPagamento`, `ConfigurarNivelAcao`.

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest tests/test_shop_loop.py tests/test_copilot.py tests/test_cobranca.py tests/test_isolamento.py -v`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/petia/runtime/app.py src/petia/web src/petia/copilot/niveis.py src/petia/command_os/build.py tests/test_shop_loop.py
git commit -m "feat: expose shop screens, command HTTP API, and PSP webhook"
```

---

### Task 14: Alembic, seed piloto, DECISIONS e verificação do spec

**Files:**
- Create: `alembic.ini`
- Create: `alembic/env.py`
- Create: `alembic/versions/0001_command_os.py`
- Modify: `DECISIONS.md`
- Modify: `ARCHITECTURE.md` — only a pointer to this plan and the spec, no extra subsystems

**Interfaces:**
- Consumes: all `ensure_*` DDL
- Produces: `alembic upgrade head` creates the same schema; document stack decision

- [ ] **Step 1: Write the failing test**

`tests/test_schema_alembic.py`:

```python
from pathlib import Path


def test_alembic_revision_exists():
    path = Path("alembic/versions/0001_command_os.py")
    assert path.exists()
    text = path.read_text(encoding="utf-8")
    assert "comando_log" in text
    assert "ordem_servico" in text
    assert "pending_confirmation" in text
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_schema_alembic.py -v`

Expected: FAIL file missing.

- [ ] **Step 3: Write minimal implementation**

Create Alembic revision whose `upgrade()` concatenates the DDL strings already defined in `ensure_*` functions (copy the SQL into the revision so migrations do not import app side effects). `downgrade()` drops the tables in reverse order.

Append to `DECISIONS.md`:

```markdown
## 2026-08-29 — Stack da primeira fatia (Command OS)

- Linguagem: Python 3.12
- API: FastAPI
- Persistência: PostgreSQL 16 + SQLAlchemy 2 com SQL parametrizado nos handlers
- UI da loja: Jinja2 (WhatsApp é a UI do tutor)
- Agente: porta `Agent`; testes com `FakeAgent` / `DownAgent`
- Pagamento: porta `Psp`; testes com `FakePsp` / `DownPsp`
- Hipótese restante: PSP e WhatsApp Cloud API reais na loja-piloto
```

Append to `ARCHITECTURE.md` two sentences: Command OS as unique write path; link to `.cursor/docs/superpowers/specs/2026-08-29-petia-command-os-design.md` and this plan.

- [ ] **Step 4: Run the tests and make sure they pass**

Run: `pytest -v`

Expected: all tests PASS, including `test_schema_alembic`.

- [ ] **Step 5: Commit**

```bash
git add alembic.ini alembic DECISIONS.md ARCHITECTURE.md tests/test_schema_alembic.py
git commit -m "chore: add command-os schema migration and record stack decision"
```

---

## Self-review (plan vs spec)

| Spec | Task |
|---|---|
| Cadastro 360 operacional | 4 |
| Agenda duração fixa, slot, RSVP, cancel, reagendar | 5–6 |
| OS entrada/execução/saída + fotos como URLs | 7 |
| Timeline / eventos | 8 |
| Recomendar sem estoque, pet pronto | 9 |
| PIX/link + webhook idempotente, PSP down | 10 |
| Confirmar no chat, expirado, sim duplicado | 11–12 |
| Mesmo comando tela e chat | 12, 13 |
| LLM não grava SQL; orquestrador chama agente | 12 |
| Tenant → Unidade, isolamento | 3 |
| Agente caído, tela segue | 12 |
| Fora do catálogo | 12 |
| Configurar nível por ação | 13 |
| Telas recepção/tosador | 13 |
| Fora de escopo (holding, estoque, IA preditiva) | nenhum task — proposital |

Não há passo “TBD” ou “similar to Task N” sem código. Stack que o spec deixou aberta está travada na Task 14 / `DECISIONS.md`.
