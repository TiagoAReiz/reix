# Tool Contract Vertical Slice — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Provar o contrato agente↔executor ponta a ponta: as tools `status` e `read` do control-plane, finas, delegando via HTTP a um agent runtime mínimo, dentro do loop `create_agent` com checkpointer.

**Architecture:** Control-plane em Python. Cada tool é uma função `@tool` fina que valida e delega para um `SandboxAdapter` (cliente HTTP). O adapter fala com um agent runtime FastAPI que roda "no container" e executa o efeito real sobre um workspace em disco. O envelope (sucesso/recuperável) volta como retorno; erro de transporte vira `FatalToolError`. O loop é o `create_agent` do LangChain, com estado persistido por um checkpointer do LangGraph, `thread_id` = id da Session. Contexto (tenant/user/app/sandbox) é injetado via `config`, nunca vem do LLM.

**Tech Stack:** Python 3.12, LangChain (`@tool`, `create_agent`), LangGraph (checkpointer), FastAPI, httpx (`ASGITransport` para testes in-process), pytest + pytest-asyncio, respx.

## Global Constraints

- Python 3.11+ (ambiente tem 3.12).
- Loop agêntico via LangChain `create_agent`; toda tool declarada com `@tool`.
- Estado persistido via checkpointer LangGraph; `thread_id` = id da Session.
- `tenant_id`/`user_id`/`app_id`/`sandbox_ref` são INJETADOS via `config["configurable"]`, nunca parâmetros preenchidos pelo LLM.
- Envelope: sucesso `{"ok": True, "data": {...}}`; recuperável `{"ok": False, "error": ..., "recoverable": True, "hint": ...}` é RETORNADO; fatal LEVANTA `FatalToolError`.
- Erro de transporte (runtime caiu/timeout) → `FatalToolError`. Erro de domínio do runtime (`not_found`, `ambiguous`, `name_exists`) → envelope recuperável com hint. O runtime responde HTTP 200 com `{"ok": False, "reason": ...}` para erro de domínio; não-2xx é sempre fatal.
- Container tools (`status`/`read`/`write`/`change`/`gen`/`push`) passam pelo agent runtime HTTP. `promote`/`scale` NÃO — fora do escopo deste plano.
- `internal_url` de serviço é sempre nome estável, nunca IP (não exercido nesta fatia, mas vale como regra).
- Usar caminhos explícitos `.venv/bin/python` e `.venv/bin/pytest` (shell é fish; não depender de `activate`).

---

### Task 1: Scaffold do projeto + toolchain

**Files:**
- Create: `pyproject.toml`
- Create: `control_plane/__init__.py`
- Create: `agent_runtime/__init__.py`
- Create: `tests/__init__.py`
- Create: `tests/test_smoke.py`
- Create: `.gitignore`

**Interfaces:**
- Consumes: nada.
- Produces: pacotes importáveis `control_plane`, `agent_runtime`; ambiente `.venv` com LangChain/LangGraph/FastAPI/httpx/pytest.

- [ ] **Step 1: Escrever o `.gitignore`**

Create `.gitignore`:

```gitignore
.venv/
__pycache__/
*.pyc
.pytest_cache/
```

- [ ] **Step 2: Escrever o `pyproject.toml`**

Create `pyproject.toml`:

```toml
[project]
name = "reix-control-plane"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "langchain",
    "langgraph",
    "fastapi",
    "httpx",
    "pydantic",
]

[project.optional-dependencies]
dev = [
    "pytest",
    "pytest-asyncio",
    "respx",
]

[tool.setuptools.packages.find]
include = ["control_plane*", "agent_runtime*"]

[tool.pytest.ini_options]
asyncio_mode = "auto"

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"
```

- [ ] **Step 3: Criar os pacotes vazios**

Create `control_plane/__init__.py`, `agent_runtime/__init__.py`, `tests/__init__.py` (todos vazios).

- [ ] **Step 4: Criar venv e instalar**

Run:
```bash
python3 -m venv .venv && .venv/bin/pip install -U pip && .venv/bin/pip install -e ".[dev]"
```
Expected: instala sem erro; termina com "Successfully installed ...".

- [ ] **Step 5: Escrever o teste de smoke (prova que os imports-chave existem)**

Create `tests/test_smoke.py`:

```python
def test_key_imports():
    from langchain.tools import tool
    from langchain.agents import create_agent
    from langgraph.checkpoint.memory import InMemorySaver

    assert callable(tool)
    assert callable(create_agent)
    assert InMemorySaver is not None
```

- [ ] **Step 6: Rodar o smoke**

Run: `.venv/bin/pytest tests/test_smoke.py -v`
Expected: PASS. Se algum import falhar, ajustar o caminho conforme a versão instalada (ex.: `from langchain_core.tools import tool`) e corrigir aqui antes de seguir — todas as tasks seguintes assumem estes três imports.

- [ ] **Step 7: Commit**

```bash
git add pyproject.toml control_plane/__init__.py agent_runtime/__init__.py tests/__init__.py tests/test_smoke.py .gitignore
git commit -m "chore: scaffold control-plane + agent-runtime packages"
```

---

### Task 2: Envelope de retorno + FatalToolError

**Files:**
- Create: `control_plane/tools/__init__.py`
- Create: `control_plane/tools/envelope.py`
- Test: `tests/test_envelope.py`

**Interfaces:**
- Consumes: nada.
- Produces:
  - `ok(data: dict) -> dict` → `{"ok": True, "data": data}`
  - `recoverable(error: str, hint: str) -> dict` → `{"ok": False, "error": error, "recoverable": True, "hint": hint}`
  - `class FatalToolError(Exception)`

- [ ] **Step 1: Escrever o teste que falha**

Create `tests/test_envelope.py`:

```python
import pytest
from control_plane.tools.envelope import ok, recoverable, FatalToolError


def test_ok_wraps_data():
    assert ok({"content": "hi"}) == {"ok": True, "data": {"content": "hi"}}


def test_recoverable_shape():
    env = recoverable("nome 'db' já existe", "escolha outro name")
    assert env == {
        "ok": False,
        "error": "nome 'db' já existe",
        "recoverable": True,
        "hint": "escolha outro name",
    }


def test_fatal_is_exception():
    with pytest.raises(FatalToolError):
        raise FatalToolError("adapter caiu")
```

- [ ] **Step 2: Rodar pra ver falhar**

Run: `.venv/bin/pytest tests/test_envelope.py -v`
Expected: FAIL com `ModuleNotFoundError: No module named 'control_plane.tools.envelope'`.

- [ ] **Step 3: Implementar**

Create `control_plane/tools/__init__.py` (vazio).

Create `control_plane/tools/envelope.py`:

```python
"""O envelope de retorno das tools (contrato agente↔executor)."""
from typing import Any


class FatalToolError(Exception):
    """Erro que o agente não resolve sozinho. Para o loop; o checkpoint guarda o ponto."""


def ok(data: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "data": data}


def recoverable(error: str, hint: str) -> dict[str, Any]:
    return {"ok": False, "error": error, "recoverable": True, "hint": hint}
```

- [ ] **Step 4: Rodar pra ver passar**

Run: `.venv/bin/pytest tests/test_envelope.py -v`
Expected: PASS (3 passed).

- [ ] **Step 5: Commit**

```bash
git add control_plane/tools/__init__.py control_plane/tools/envelope.py tests/test_envelope.py
git commit -m "feat: envelope de retorno e FatalToolError"
```

---

### Task 3: Contexto injetável + get_current_context

**Files:**
- Create: `control_plane/domain/__init__.py`
- Create: `control_plane/domain/context.py`
- Test: `tests/test_context.py`

**Interfaces:**
- Consumes: nada.
- Produces:
  - `@dataclass(frozen=True) class Context` com campos `tenant_id: str`, `user_id: str`, `app_id: str`, `sandbox_ref: str`.
  - `get_current_context(app_id: str, sandbox_ref: str) -> Context` (tenant/user fixos hoje).
  - `FIXED_TENANT_ID: str`, `FIXED_USER_ID: str`.

- [ ] **Step 1: Escrever o teste que falha**

Create `tests/test_context.py`:

```python
from control_plane.domain.context import (
    Context,
    get_current_context,
    FIXED_TENANT_ID,
    FIXED_USER_ID,
)


def test_context_is_frozen():
    ctx = Context(tenant_id="t", user_id="u", app_id="a", sandbox_ref="s")
    try:
        ctx.tenant_id = "x"
        assert False, "Context deveria ser imutável"
    except AttributeError:
        pass


def test_get_current_context_uses_fixed_identity():
    ctx = get_current_context(app_id="app_1", sandbox_ref="box_1")
    assert ctx.tenant_id == FIXED_TENANT_ID
    assert ctx.user_id == FIXED_USER_ID
    assert ctx.app_id == "app_1"
    assert ctx.sandbox_ref == "box_1"
```

- [ ] **Step 2: Rodar pra ver falhar**

Run: `.venv/bin/pytest tests/test_context.py -v`
Expected: FAIL com `ModuleNotFoundError: No module named 'control_plane.domain.context'`.

- [ ] **Step 3: Implementar**

Create `control_plane/domain/__init__.py` (vazio).

Create `control_plane/domain/context.py`:

```python
"""O contexto de identidade/execução. Ponto único trocável: hoje fixo, amanhã lê o JWT."""
from dataclasses import dataclass

FIXED_TENANT_ID = "t_dev"
FIXED_USER_ID = "u_dev"


@dataclass(frozen=True)
class Context:
    tenant_id: str
    user_id: str
    app_id: str
    sandbox_ref: str


def get_current_context(app_id: str, sandbox_ref: str) -> Context:
    # HOJE: tenant/user fixos. AMANHÃ: lê o JWT. Troca só aqui.
    return Context(
        tenant_id=FIXED_TENANT_ID,
        user_id=FIXED_USER_ID,
        app_id=app_id,
        sandbox_ref=sandbox_ref,
    )
```

- [ ] **Step 4: Rodar pra ver passar**

Run: `.venv/bin/pytest tests/test_context.py -v`
Expected: PASS (2 passed).

- [ ] **Step 5: Commit**

```bash
git add control_plane/domain/__init__.py control_plane/domain/context.py tests/test_context.py
git commit -m "feat: Context injetável e get_current_context (identidade fixa)"
```

---

### Task 4: SandboxAdapter (cliente HTTP) + tradução de erro

**Files:**
- Create: `control_plane/adapters/__init__.py`
- Create: `control_plane/adapters/sandbox/__init__.py`
- Create: `control_plane/adapters/sandbox/base.py`
- Create: `control_plane/adapters/sandbox/http.py`
- Test: `tests/test_sandbox_adapter.py`

**Interfaces:**
- Consumes: `ok`, `recoverable`, `FatalToolError` de `control_plane.tools.envelope`.
- Produces:
  - `class SandboxAdapter(Protocol)` com `async def status(self, scope: str) -> dict` e `async def read(self, path: str) -> dict`.
  - `class HttpSandboxAdapter` com `__init__(self, base_url: str, token: str, client: httpx.AsyncClient | None = None)` e os métodos `status`/`read` async retornando o envelope.

- [ ] **Step 1: Escrever o teste que falha**

Create `tests/test_sandbox_adapter.py`:

```python
import httpx
import pytest
import respx

from control_plane.adapters.sandbox.http import HttpSandboxAdapter
from control_plane.tools.envelope import FatalToolError

BASE = "http://runtime.test"


def _adapter() -> HttpSandboxAdapter:
    return HttpSandboxAdapter(base_url=BASE, token="tok")


@respx.mock
async def test_read_success_returns_ok_envelope():
    respx.post(f"{BASE}/fs/read").mock(
        return_value=httpx.Response(200, json={"ok": True, "content": "hello"})
    )
    env = await _adapter().read("README.md")
    assert env == {"ok": True, "data": {"content": "hello"}}


@respx.mock
async def test_read_domain_error_is_recoverable_with_hint():
    respx.post(f"{BASE}/fs/read").mock(
        return_value=httpx.Response(200, json={"ok": False, "reason": "not_found"})
    )
    env = await _adapter().read("nope.txt")
    assert env["ok"] is False
    assert env["recoverable"] is True
    assert "status" in env["hint"]


@respx.mock
async def test_transport_error_is_fatal():
    respx.post(f"{BASE}/fs/read").mock(side_effect=httpx.ConnectError("boom"))
    with pytest.raises(FatalToolError):
        await _adapter().read("README.md")


@respx.mock
async def test_non_2xx_is_fatal():
    respx.post(f"{BASE}/fs/read").mock(return_value=httpx.Response(500))
    with pytest.raises(FatalToolError):
        await _adapter().read("README.md")


@respx.mock
async def test_status_success_returns_ok_envelope():
    respx.get(f"{BASE}/status").mock(
        return_value=httpx.Response(
            200,
            json={
                "ok": True,
                "services": [],
                "git": {"branch": "dev", "last_commit": None, "dirty": False},
                "environments": {"dev": {}, "prod": {}},
                "files": ["README.md"],
            },
        )
    )
    env = await _adapter().status("all")
    assert env["ok"] is True
    assert env["data"]["files"] == ["README.md"]
    assert env["data"]["git"]["branch"] == "dev"


@respx.mock
async def test_adapter_sends_bearer_token():
    route = respx.post(f"{BASE}/fs/read").mock(
        return_value=httpx.Response(200, json={"ok": True, "content": "x"})
    )
    await _adapter().read("README.md")
    assert route.calls.last.request.headers["Authorization"] == "Bearer tok"
```

- [ ] **Step 2: Rodar pra ver falhar**

Run: `.venv/bin/pytest tests/test_sandbox_adapter.py -v`
Expected: FAIL com `ModuleNotFoundError: No module named 'control_plane.adapters.sandbox.http'`.

- [ ] **Step 3: Implementar a interface**

Create `control_plane/adapters/__init__.py` (vazio) e `control_plane/adapters/sandbox/__init__.py` (vazio).

Create `control_plane/adapters/sandbox/base.py`:

```python
"""Interface do agent runtime (as 'mãos' no container). Trocável."""
from typing import Protocol


class SandboxAdapter(Protocol):
    async def status(self, scope: str) -> dict: ...
    async def read(self, path: str) -> dict: ...
```

- [ ] **Step 4: Implementar o cliente HTTP**

Create `control_plane/adapters/sandbox/http.py`:

```python
"""Impl HTTP do SandboxAdapter: traduz intenção em request pro agent runtime."""
import httpx

from control_plane.tools.envelope import FatalToolError, ok, recoverable

_HINTS = {
    "not_found": "verifique o caminho com status(scope='files')",
    "ambiguous": "o old_str aparece mais de uma vez; inclua mais contexto pra torná-lo único",
    "name_exists": "escolha outro name ou use status para ver os serviços existentes",
}


def _hint_for(reason: str | None) -> str:
    return _HINTS.get(reason or "", "revise os argumentos e tente novamente")


class HttpSandboxAdapter:
    def __init__(self, base_url: str, token: str, client: httpx.AsyncClient | None = None):
        self._base_url = base_url.rstrip("/")
        self._headers = {"Authorization": f"Bearer {token}"}
        self._client = client or httpx.AsyncClient()

    async def _call(self, method: str, path: str, **kwargs) -> dict:
        try:
            resp = await self._client.request(
                method, f"{self._base_url}{path}", headers=self._headers, **kwargs
            )
            resp.raise_for_status()
        except httpx.HTTPError as exc:
            raise FatalToolError(f"agent runtime indisponível: {exc}") from exc
        return resp.json()

    async def read(self, path: str) -> dict:
        body = await self._call("POST", "/fs/read", json={"path": path})
        if body.get("ok"):
            return ok({"content": body["content"]})
        reason = body.get("reason")
        return recoverable(f"não foi possível ler '{path}': {reason}", _hint_for(reason))

    async def status(self, scope: str) -> dict:
        body = await self._call("GET", "/status", params={"scope": scope})
        return ok({k: body[k] for k in ("services", "git", "environments", "files")})
```

- [ ] **Step 5: Rodar pra ver passar**

Run: `.venv/bin/pytest tests/test_sandbox_adapter.py -v`
Expected: PASS (6 passed).

- [ ] **Step 6: Commit**

```bash
git add control_plane/adapters tests/test_sandbox_adapter.py
git commit -m "feat: SandboxAdapter HTTP com tradução recuperável/fatal"
```

---

### Task 5: Agent runtime mínimo (FastAPI)

**Files:**
- Create: `agent_runtime/server.py`
- Test: `tests/test_agent_runtime.py`

**Interfaces:**
- Consumes: nada do control-plane (é o outro lado do contrato).
- Produces: `create_app(workspace: pathlib.Path) -> fastapi.FastAPI`, servindo:
  - `POST /fs/read {path}` → `{"ok": True, "content": str}` ou `{"ok": False, "reason": "not_found"}`.
  - `GET /status?scope=all` → `{"ok": True, "services": [], "git": {...}, "environments": {...}, "files": [...]}`.

- [ ] **Step 1: Escrever o teste que falha**

Create `tests/test_agent_runtime.py`:

```python
import pathlib

import httpx

from agent_runtime.server import create_app


async def _client(workspace: pathlib.Path) -> httpx.AsyncClient:
    app = create_app(workspace)
    transport = httpx.ASGITransport(app=app)
    return httpx.AsyncClient(transport=transport, base_url="http://runtime")


async def test_read_existing_file(tmp_path):
    (tmp_path / "README.md").write_text("hello")
    client = await _client(tmp_path)
    resp = await client.post("/fs/read", json={"path": "README.md"})
    assert resp.json() == {"ok": True, "content": "hello"}


async def test_read_missing_file_is_not_found(tmp_path):
    client = await _client(tmp_path)
    resp = await client.post("/fs/read", json={"path": "nope.txt"})
    assert resp.json() == {"ok": False, "reason": "not_found"}


async def test_status_lists_files(tmp_path):
    (tmp_path / "a.txt").write_text("x")
    client = await _client(tmp_path)
    resp = await client.get("/status", params={"scope": "all"})
    body = resp.json()
    assert body["ok"] is True
    assert "a.txt" in body["files"]
    assert body["git"]["branch"] == "dev"
    assert body["environments"] == {"dev": {}, "prod": {}}
```

- [ ] **Step 2: Rodar pra ver falhar**

Run: `.venv/bin/pytest tests/test_agent_runtime.py -v`
Expected: FAIL com `ModuleNotFoundError: No module named 'agent_runtime.server'`.

- [ ] **Step 3: Implementar**

Create `agent_runtime/server.py`:

```python
"""Agent runtime mínimo: as 'mãos' no container. Executa efeito sobre um workspace em disco.

MVP: sem verificação de token (dívida aceita — ressalva #3). O adapter já envia o Bearer.
"""
import pathlib

from fastapi import FastAPI
from pydantic import BaseModel


class ReadReq(BaseModel):
    path: str


def create_app(workspace: pathlib.Path) -> FastAPI:
    app = FastAPI()

    @app.post("/fs/read")
    async def fs_read(req: ReadReq) -> dict:
        target = workspace / req.path
        if not target.is_file():
            return {"ok": False, "reason": "not_found"}
        return {"ok": True, "content": target.read_text()}

    @app.get("/status")
    async def status(scope: str = "all") -> dict:
        files = [
            str(p.relative_to(workspace))
            for p in workspace.rglob("*")
            if p.is_file()
        ]
        return {
            "ok": True,
            "services": [],
            "git": {"branch": "dev", "last_commit": None, "dirty": False},
            "environments": {"dev": {}, "prod": {}},
            "files": files,
        }

    return app
```

- [ ] **Step 4: Rodar pra ver passar**

Run: `.venv/bin/pytest tests/test_agent_runtime.py -v`
Expected: PASS (3 passed).

- [ ] **Step 5: Commit**

```bash
git add agent_runtime/server.py tests/test_agent_runtime.py
git commit -m "feat: agent runtime mínimo (POST /fs/read, GET /status)"
```

---

### Task 6: As tools `read` e `status` + acesso ao config

**Files:**
- Create: `control_plane/tools/context_access.py`
- Create: `control_plane/tools/definitions/__init__.py`
- Create: `control_plane/tools/definitions/read.py`
- Create: `control_plane/tools/definitions/status.py`
- Create: `control_plane/tools/registry.py`
- Test: `tests/test_tools.py`

**Interfaces:**
- Consumes: `SandboxAdapter` (via config), `FatalToolError`, `Context`.
- Produces:
  - `sandbox_from_config(config) -> SandboxAdapter` e `context_from_config(config) -> Context` em `context_access.py` (levantam `FatalToolError` se ausente).
  - `read` e `status` (objetos `@tool`) em `definitions/`.
  - `ALL_TOOLS: list` em `registry.py`.
- Nota: as tools declaram `config: RunnableConfig` — o LangChain injeta o config e esconde esse parâmetro do schema visto pelo LLM. A chave `configurable` carrega `context` (`Context`) e `sandbox` (`SandboxAdapter`).

- [ ] **Step 1: Escrever o teste que falha**

Create `tests/test_tools.py`:

```python
import pytest

from control_plane.domain.context import get_current_context
from control_plane.tools.context_access import (
    context_from_config,
    sandbox_from_config,
)
from control_plane.tools.definitions.read import read
from control_plane.tools.definitions.status import status
from control_plane.tools.envelope import FatalToolError
from control_plane.tools.registry import ALL_TOOLS


class FakeSandbox:
    async def read(self, path: str) -> dict:
        return {"ok": True, "data": {"content": f"conteudo de {path}"}}

    async def status(self, scope: str) -> dict:
        return {"ok": True, "data": {"scope": scope, "files": ["README.md"]}}


def _config(sandbox=None, context=None):
    return {
        "configurable": {
            "thread_id": "sess_1",
            "sandbox": sandbox,
            "context": context,
        }
    }


def test_config_accessors_return_injected_values():
    ctx = get_current_context("app_1", "box_1")
    sb = FakeSandbox()
    cfg = _config(sandbox=sb, context=ctx)
    assert sandbox_from_config(cfg) is sb
    assert context_from_config(cfg) is ctx


def test_config_accessor_raises_fatal_when_missing():
    with pytest.raises(FatalToolError):
        sandbox_from_config({"configurable": {}})


async def test_read_tool_delegates_to_sandbox():
    cfg = _config(sandbox=FakeSandbox(), context=get_current_context("a", "b"))
    result = await read.ainvoke({"path": "README.md"}, config=cfg)
    assert result == {"ok": True, "data": {"content": "conteudo de README.md"}}


async def test_status_tool_delegates_to_sandbox():
    cfg = _config(sandbox=FakeSandbox(), context=get_current_context("a", "b"))
    result = await status.ainvoke({"scope": "all"}, config=cfg)
    assert result["ok"] is True
    assert result["data"]["scope"] == "all"


def test_llm_visible_args_exclude_injected_config():
    # o schema que o LLM vê não deve conter 'config'
    assert "config" not in read.args
    assert "path" in read.args
    assert "config" not in status.args


def test_registry_lists_both_tools():
    names = {t.name for t in ALL_TOOLS}
    assert names == {"read", "status"}
```

- [ ] **Step 2: Rodar pra ver falhar**

Run: `.venv/bin/pytest tests/test_tools.py -v`
Expected: FAIL com `ModuleNotFoundError: No module named 'control_plane.tools.context_access'`.

- [ ] **Step 3: Implementar os acessores de config**

Create `control_plane/tools/context_access.py`:

```python
"""Lê o contexto injetado no config do LangChain. O LLM nunca preenche isso."""
from typing import Any

from control_plane.adapters.sandbox.base import SandboxAdapter
from control_plane.domain.context import Context
from control_plane.tools.envelope import FatalToolError


def _require(config: dict[str, Any], key: str):
    try:
        value = config["configurable"][key]
    except (KeyError, TypeError) as exc:
        raise FatalToolError(f"'{key}' ausente no contexto da execução") from exc
    if value is None:
        raise FatalToolError(f"'{key}' ausente no contexto da execução")
    return value


def sandbox_from_config(config: dict[str, Any]) -> SandboxAdapter:
    return _require(config, "sandbox")


def context_from_config(config: dict[str, Any]) -> Context:
    return _require(config, "context")
```

- [ ] **Step 4: Implementar a tool `read`**

Create `control_plane/tools/definitions/__init__.py` (vazio).

Create `control_plane/tools/definitions/read.py`:

```python
"""read: lê o conteúdo de um arquivo do app. Tool fina — delega ao agent runtime."""
from langchain_core.runnables import RunnableConfig
from langchain.tools import tool

from control_plane.tools.context_access import sandbox_from_config


@tool
async def read(path: str, config: RunnableConfig) -> dict:
    """Lê o conteúdo de um arquivo do app pelo caminho relativo.

    Use antes de editar um arquivo, pra ver o conteúdo atual.
    """
    sandbox = sandbox_from_config(config)
    return await sandbox.read(path)
```

- [ ] **Step 5: Implementar a tool `status`**

Create `control_plane/tools/definitions/status.py`:

```python
"""status: lê o estado atual do app (serviços, arquivos, git, ambientes)."""
from typing import Literal

from langchain_core.runnables import RunnableConfig
from langchain.tools import tool

from control_plane.tools.context_access import sandbox_from_config


@tool
async def status(
    scope: Literal["services", "files", "git", "environments", "all"] = "all",
    config: RunnableConfig = None,
) -> dict:
    """Lê o estado atual do app: serviços, arquivos, git e ambientes.

    Use antes de agir, pra decidir com base no que já existe.
    """
    sandbox = sandbox_from_config(config)
    return await sandbox.status(scope)
```

- [ ] **Step 6: Implementar o registry**

Create `control_plane/tools/registry.py`:

```python
"""Catálogo de tools registradas no agente."""
from control_plane.tools.definitions.read import read
from control_plane.tools.definitions.status import status

ALL_TOOLS = [status, read]
```

- [ ] **Step 7: Rodar pra ver passar**

Run: `.venv/bin/pytest tests/test_tools.py -v`
Expected: PASS (6 passed). Se `read.args` incluir `config`, confirmar que o parâmetro está tipado como `RunnableConfig` (o LangChain só o esconde quando reconhece esse tipo).

- [ ] **Step 8: Commit**

```bash
git add control_plane/tools/context_access.py control_plane/tools/definitions control_plane/tools/registry.py tests/test_tools.py
git commit -m "feat: tools read e status (finas, delegando via config)"
```

---

### Task 7: Wiring do agente (create_agent + checkpointer)

**Files:**
- Create: `control_plane/agent/__init__.py`
- Create: `control_plane/agent/loop.py`
- Test: `tests/test_agent_wiring.py`

**Interfaces:**
- Consumes: `ALL_TOOLS`, `InMemorySaver`.
- Produces:
  - `SYSTEM_PROMPT: str`.
  - `build_agent(model, checkpointer=None)` → agente `create_agent` com `ALL_TOOLS` e checkpointer (default `InMemorySaver`).
  - `run_config(session_id: str, context, sandbox, recursion_limit: int = 50) -> dict` — monta o config com `thread_id` e o contexto injetado.

- [ ] **Step 1: Escrever o teste que falha**

Create `tests/test_agent_wiring.py`:

```python
from langgraph.checkpoint.memory import InMemorySaver

from control_plane.agent.loop import build_agent, run_config, SYSTEM_PROMPT
from control_plane.domain.context import get_current_context


class DummyModel:
    pass


def test_run_config_carries_thread_id_and_context():
    ctx = get_current_context("app_1", "box_1")
    cfg = run_config("sess_9", context=ctx, sandbox="SB")
    assert cfg["configurable"]["thread_id"] == "sess_9"
    assert cfg["configurable"]["context"] is ctx
    assert cfg["configurable"]["sandbox"] == "SB"
    assert cfg["recursion_limit"] == 50


def test_system_prompt_is_non_empty():
    assert isinstance(SYSTEM_PROMPT, str) and SYSTEM_PROMPT.strip()


def test_build_agent_defaults_to_in_memory_saver():
    # não deve levantar ao construir com checkpointer default
    agent = build_agent(model=_scripted([]))
    assert agent is not None


def _scripted(responses):
    from tests.helpers.scripted_model import ScriptedModel
    return ScriptedModel(responses=responses)
```

- [ ] **Step 2: Rodar pra ver falhar**

Run: `.venv/bin/pytest tests/test_agent_wiring.py -v`
Expected: FAIL com `ModuleNotFoundError: No module named 'control_plane.agent.loop'`.

- [ ] **Step 3: Implementar o wiring**

Create `control_plane/agent/__init__.py` (vazio).

Create `control_plane/agent/loop.py`:

```python
"""O loop agêntico: create_agent do LangChain + checkpointer do LangGraph."""
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver

from control_plane.tools.registry import ALL_TOOLS

SYSTEM_PROMPT = (
    "Você é o agente que compõe apps. Use status para ver o estado antes de agir "
    "e read para ver arquivos antes de editar. As tools expressam intenção sobre o "
    "app; você nunca toca runtime diretamente."
)


def build_agent(model, checkpointer=None):
    return create_agent(
        model=model,
        tools=ALL_TOOLS,
        system_prompt=SYSTEM_PROMPT,
        checkpointer=checkpointer or InMemorySaver(),
    )


def run_config(session_id: str, context, sandbox, recursion_limit: int = 50) -> dict:
    return {
        "configurable": {
            "thread_id": session_id,
            "context": context,
            "sandbox": sandbox,
        },
        "recursion_limit": recursion_limit,
    }
```

- [ ] **Step 4: Criar o modelo de teste scriptado (reusado no e2e)**

Create `tests/helpers/__init__.py` (vazio).

Create `tests/helpers/scripted_model.py`:

```python
"""Chat model de teste que devolve mensagens pré-scriptadas (emite tool calls)."""
from langchain_core.language_models import BaseChatModel
from langchain_core.outputs import ChatGeneration, ChatResult


class ScriptedModel(BaseChatModel):
    responses: list

    def bind_tools(self, tools, **kwargs):
        return self

    def _generate(self, messages, stop=None, run_manager=None, **kwargs) -> ChatResult:
        message = self.responses.pop(0)
        return ChatResult(generations=[ChatGeneration(message=message)])

    async def _agenerate(self, messages, stop=None, run_manager=None, **kwargs) -> ChatResult:
        message = self.responses.pop(0)
        return ChatResult(generations=[ChatGeneration(message=message)])

    @property
    def _llm_type(self) -> str:
        return "scripted"
```

- [ ] **Step 5: Rodar pra ver passar**

Run: `.venv/bin/pytest tests/test_agent_wiring.py -v`
Expected: PASS (3 passed). Se `create_agent` não aceitar `system_prompt` na versão instalada, checar o nome do parâmetro no smoke da Task 1 e ajustar aqui.

- [ ] **Step 6: Commit**

```bash
git add control_plane/agent tests/test_agent_wiring.py tests/helpers
git commit -m "feat: build_agent (create_agent + InMemorySaver) e run_config"
```

---

### Task 8: Teste end-to-end (loop → tool → adapter → runtime)

**Files:**
- Test: `tests/test_e2e_read.py`

**Interfaces:**
- Consumes: `build_agent`, `run_config`, `HttpSandboxAdapter`, `create_app` (agent runtime), `ScriptedModel`, `get_current_context`.
- Produces: nada (é o teste que prova a fatia inteira).

- [ ] **Step 1: Escrever o teste e2e**

Create `tests/test_e2e_read.py`:

```python
"""Prova a fatia inteira: LLM (scriptado) → tool read → adapter HTTP → agent runtime → volta."""
import httpx
from langchain_core.messages import AIMessage, HumanMessage

from agent_runtime.server import create_app
from control_plane.adapters.sandbox.http import HttpSandboxAdapter
from control_plane.agent.loop import build_agent, run_config
from control_plane.domain.context import get_current_context
from tests.helpers.scripted_model import ScriptedModel


def _adapter_wired_to(workspace) -> HttpSandboxAdapter:
    app = create_app(workspace)
    transport = httpx.ASGITransport(app=app)
    client = httpx.AsyncClient(transport=transport, base_url="http://runtime")
    return HttpSandboxAdapter(base_url="http://runtime", token="tok", client=client)


async def test_agent_reads_file_end_to_end(tmp_path):
    (tmp_path / "README.md").write_text("olá mundo")

    # o modelo scriptado: 1) chama read; 2) responde final citando o conteúdo
    scripted = ScriptedModel(
        responses=[
            AIMessage(
                content="",
                tool_calls=[{"name": "read", "args": {"path": "README.md"}, "id": "call_1"}],
            ),
            AIMessage(content="O arquivo contém: olá mundo"),
        ]
    )

    agent = build_agent(model=scripted)
    ctx = get_current_context(app_id="app_1", sandbox_ref="box_1")
    sandbox = _adapter_wired_to(tmp_path)
    cfg = run_config("sess_e2e", context=ctx, sandbox=sandbox)

    result = await agent.ainvoke(
        {"messages": [HumanMessage(content="leia o README")]},
        config=cfg,
    )

    messages = result["messages"]
    # a tool foi executada e devolveu o envelope de sucesso com o conteúdo
    tool_messages = [m for m in messages if m.type == "tool"]
    assert tool_messages, "a tool read deveria ter sido executada"
    assert "olá mundo" in tool_messages[-1].content
    # a resposta final do agente incorporou o resultado
    assert "olá mundo" in messages[-1].content
```

- [ ] **Step 2: Rodar o e2e**

Run: `.venv/bin/pytest tests/test_e2e_read.py -v`
Expected: PASS (1 passed). Se falhar na forma da tool call (`tool_calls` schema), conferir a versão do `langchain_core` e ajustar o dict do tool call (campos `name`/`args`/`id`).

- [ ] **Step 3: Rodar a suíte inteira**

Run: `.venv/bin/pytest -v`
Expected: PASS (todos os testes das Tasks 1–8 verdes).

- [ ] **Step 4: Commit**

```bash
git add tests/test_e2e_read.py
git commit -m "test: e2e read (loop → tool → adapter → agent runtime)"
```

---

## Follow-up (planos seguintes, fora deste)

1. **Tools de container restantes:** `write`, `change`, `gen` (+ espelho do manifest no banco), `push` (git dentro do container) — mesmos padrões desta fatia, mais endpoints no agent runtime.
2. **Tools de control-plane:** `promote` (adapter Git dev→master + trigger CI/CD) e `scale` (adapter runtime/k8s) — caminho sem agent runtime.
3. **Persistência real:** trocar `InMemorySaver` por `PostgresSaver` (mudança de config no `build_agent`) + docker-compose de criação.
4. **Streaming de eventos:** `astream_events` → SSE pro editor.
5. **LLM real / BYOK:** plugar LiteLLM como `model` no `build_agent`.
