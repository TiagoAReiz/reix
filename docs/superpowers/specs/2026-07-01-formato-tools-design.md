# Formato das Tools — Design (contrato agente↔executor)

> Fecha os dois 🔴 do `estado_atual_contexto.md` §4: o **formato do contrato
> agente↔executor** (tool call + envelope de resultado) e as **assinaturas
> finais das tools**. É o que destrava escrever a primeira tool de verdade.
>
> Decisão de stack tomada aqui: **LangChain** (`@tool` + `create_agent`) para o
> loop agêntico, e **LangGraph** (checkpointer) para persistir estado de execução
> e não perder progresso em restart. Substitui a ideia anterior de um protocolo
> JSON próprio no texto.
>
> Companion: `arquitetura_consolidada.md`, `modelo_dominio_controlplane.md`,
> `tools_refinadas.md`, `ressalvas_decisoes.md`, `viabilidade_economica.md`.

---

## 1. O FRAME — stack e os dois contratos

As tools vivem em `tools/definitions/`, **uma função `@tool` por arquivo**. O
loop agêntico é o `create_agent` do LangChain (não escrevemos o loop à mão). O
estado de execução persiste via checkpointer do LangGraph (`PostgresSaver`), com
`thread_id` = id da Session.

Cada tool é **fina**: valida no control-plane e delega o efeito real. Cai num de
dois caminhos (a fronteira decidida — o fio condutor é *onde a coisa mora*):

```
LLM ──tool call──► @tool (control-plane: valida schema/auth/estado)
                     │
      ┌──────────────┴───────────────┐
      ▼ (workspace vivo)             ▼ (promover pra prod)
  agent runtime HTTP            adapter control-plane
  status / read / write /       promote (Git dev→master + CI/CD)
  change / gen / push           scale  (runtime/k8s, prod)
      │                             │
      ▼                             ▼
  executa no container          opera no GitHub / pipeline
```

**Por que essa fronteira:** o código (working tree **e** `.git`) mora dentro do
sandbox, clonado na preparação do container (provisionamento em 5 passos). Então
tudo sobre o *workspace vivo* acontece onde o código está → agent runtime. Já
`promote`/`scale` são sobre *promover pra prod* → control-plane puro, sem agent
runtime (bate com "prod nunca usa agent runtime", `modelo_dominio §2`).

Dois contratos, ambos documentados neste doc:

- **(A) LLM ↔ tool** — assinatura `@tool` + envelope de retorno (§2, §3).
- **(B) tool ↔ agent runtime** — o HTTP no sandbox (§4).

```python
from langchain.tools import tool
from langchain.agents import create_agent
from langgraph.checkpoint.postgres import PostgresSaver

agent = create_agent(
    model=llm,                    # LiteLLM / BYOK plugado aqui
    tools=[status, read, write, change, gen, push, promote, scale],
    system_prompt=SYSTEM_PROMPT,
    checkpointer=PostgresSaver(...),   # estado por thread_id (= Session)
)
```

> Nota de mapeamento com os docs: `agent/loop.py` vira o `create_agent`
> pré-pronto; `tools/definitions/` vira uma função `@tool` por arquivo; o
> "executor" da arquitetura é o corpo fino da tool + o agent runtime que executa
> o efeito de verdade.

---

## 2. O ENVELOPE DE RETORNO

Toda tool respeita uma forma única. **Sucesso** e **erro recuperável** *retornam*
(viram `ToolMessage` que o LLM lê e reage). **Erro fatal** *levanta exceção*
(para o grafo; o checkpoint guarda o ponto pra escalar/retomar).

```python
# sucesso
{ "ok": True, "data": { ... } }

# erro recuperável → LLM lê o hint e tenta de novo (dentro da trava de iteração)
{ "ok": False, "error": "nome 'db' já existe", "recoverable": True,
  "hint": "escolha outro name ou use status para ver os serviços existentes" }

# erro fatal → NÃO retorna, levanta:
raise FatalToolError("adapter Git indisponível")   # para o loop, escala pro humano
```

**Comportamento no loop (LangChain/LangGraph):** por padrão o `create_agent` /
`ToolNode` engole exceção de tool e a devolve como `ToolMessage`. Para
`FatalToolError` **interromper de verdade**, configuramos `handle_tool_errors`
para *re-levantar* a nossa exceção tipada. Erro recuperável nunca é exceção —
então nunca vaza pra esse caminho. A **trava de iteração** é o `recursion_limit`
do LangGraph (teto de passos → evita loop infinito gastando tokens).

| Situação | Retorno | Efeito no loop |
|----------|---------|----------------|
| Sucesso | `{ ok: true, data }` | LLM incorpora e segue |
| Erro recuperável | `{ ok: false, recoverable: true, hint }` | LLM se corrige e tenta de novo |
| Erro fatal | `raise FatalToolError(...)` | grafo para; checkpoint guarda o ponto |

---

## 3. AS 8 ASSINATURAS

**Contexto é injetado, não vem do LLM.** O modelo só passa os args semânticos.
`tenant_id`/`user_id` vêm do `get_current_context()`; `app_id`/`sandbox_ref` vêm
da Session. Viajam como `InjectedToolArg` (via `config`), nunca como parâmetro que
o modelo preenche — o LLM não forja identidade nem escolhe container.

```python
# ── container (via agent runtime) ────────────────────────────────────────────
status(scope: Literal["services","files","git","environments","all"] = "all")
  → data: {
      services: [{ name, type, internal_url, port, status }],
      git: { branch, last_commit, dirty },
      environments: { dev: {...}, prod: {...} },
      files: [ ... ]
    }

read(path: str)
  → data: { content: str }

write(path: str, content: str)                  # cria/sobrescreve arquivo novo
  → data: { path, bytes }

change(path: str, old_str: str, new_str: str)   # valida old_str existe e é único
  → data: { changes: "3 linhas em src/api.js" }
  → recuperável se old_str ausente ou ambíguo

gen(service_type: Literal["postgres","redis","rabbitmq","node","python","java"],
    name: str)
  → data: { service: { name, type, internal_url, port, env_exposed: [...] } }
  → recuperável se name já existe no app

push(commit_name: str, files: list[str] | None = None)   # git commit && push na branch dev, dentro do container
  → data: { commit: "a1b2c3", branch: "dev" }

# ── control-plane (adapters, sem agent runtime) ──────────────────────────────
promote(commit: str, to_env: Literal["dev","prod"], rollback: bool = False)
  → data: { env, now_running, previous }
  → forward-only por padrão; commit mais antigo que o rodando exige rollback=True
    (recuperável se violar sem a flag)

scale(service: str, env: Literal["prod"], replicas: int)
  → data: { service, replicas }
  → recuperável se service for stateful → hint aponta pro caminho gerenciado
```

**Decisões em aberto dos docs, resolvidas aqui:**

- `promote` **unificado** substitui `publish` + `back_version`. Commit é a fonte
  de verdade; "versão" fica como tag amigável opcional (não entra na assinatura).
- `read` / `write` existem **além** de `change` (agente não age cego; cria
  arquivo novo sem `change`).
- `scale` **recusa** serviço stateful com erro recuperável (não silencioso),
  orientando pro caminho gerenciado.
- `internal_url` é **nome estável** (`db.internal`), não IP — resolve igual em
  Compose/k8s/gerenciado.

---

## 4. O CONTRATO HTTP (tool ↔ agent runtime)

As tools de container traduzem a intenção em HTTP no runtime do sandbox.
`sandbox_ref` resolve a base URL; autenticação por **token por-sandbox** injetado
na preparação do sandbox (credencial escopada a 1 container — dívida aceita na
criação, `ressalva #3`).

```
GET  /status?scope=all              → { ok, services, git, environments, files }
POST /fs/read     { path }          → { ok, content }
POST /fs/write    { path, content } → { ok, bytes }
POST /fs/change   { path, old, new }→ { ok, changes } | { ok:false, reason:"ambiguous"|"not_found" }
POST /services/gen { service_type, name }
                                    → { ok, service, manifest }   # edita manifest + compose, sobe serviço
POST /git/push    { commit_name, files? }
                                    → { ok, commit, branch }
```

**Tradução de erro:** o runtime devolve seu próprio `{ ok, ... }`. A tool o
converte pro envelope do LLM (§2):

- Erro de **transporte** (runtime caiu, timeout, conexão) → `FatalToolError`.
- Erro de **domínio** do runtime (`ambiguous`, `name exists`, `not_found`) →
  envelope **recuperável** com hint.

**`gen` e o espelho (repo manda, banco espelha):** o `app.manifest.json` mora no
repo = no container. O runtime edita o manifest (verdade) + o compose, sobe o
serviço, e devolve o `manifest`. A tool então **espelha** a topologia no campo
`manifest_atual` do banco (cache de leitura). Nenhuma lógica de materialização
pesada no control-plane — ele valida na entrada e espelha na saída.

---

## 5. CONTEXTO, PERSISTÊNCIA E EVENTOS

**Contexto — o ponto único trocável.** Toda requisição resolve o contexto num só
lugar e injeta nas tools; o LLM nunca vê:

```python
def get_current_context() -> Context:
    # HOJE: tenant/user fixos.  AMANHÃ: lê o JWT. Troca só aqui.
    return Context(tenant_id=FIXED, user_id=FIXED, app_id=..., sandbox_ref=...)
```

`tenant_id`/`user_id`/`app_id`/`sandbox_ref` viajam como `InjectedToolArg` (via
`config`), não como parâmetros do modelo. Mantém multi-tenant desde já; a
autorização vira "toda tool filtra pelo `tenant_id` do contexto".

**Thread = Session (retomar sem perder estado).** O `thread_id` do checkpointer
**é o `id` da Session**. O `PostgresSaver` grava snapshot a cada superstep. Se o
control-plane reinicia, retomar é invocar com o mesmo `thread_id` — continua do
último checkpoint. O `histórico` e o `estado_loop` da entidade Session passam a
ser **materializados pelo checkpointer**, não campos geridos à mão.

```python
config = {"configurable": {"thread_id": session.id, **injected_context}}
agent.invoke({"messages": [{"role": "user", "content": prompt}]}, config)
```

**Async + eventos por passo (o SSE dos docs).** O loop roda em task background;
o `astream_events` do LangGraph emite evento a cada passo → stream SSE/WebSocket
pro editor:

```
usuário manda prompt → responde "comecei" + session.id
  → task roda agent.astream_events(...)
  → cada evento (on_tool_start / on_tool_end) → SSE → preview / editor
```

**Sandbox descartável, Session sobrevive.** O checkpoint (Postgres) guarda a
conversa/estado; o `sandbox_ref` no contexto aponta pro container atual. Sandbox
morreu ou dormiu? Sobe outro, atualiza `sandbox_ref`, retoma pelo `thread_id`.
Casa com os "níveis de sono" da `viabilidade_economica.md`.

---

## 6. RESUMO DAS DECISÕES

| Tema | Decisão |
|------|---------|
| Loop agêntico | `create_agent` do LangChain (não escrever à mão) |
| Declaração de tool | `@tool` + type hints + docstring (schema automático) |
| Persistência de estado | LangGraph `PostgresSaver`, `thread_id` = Session |
| Executor | tool fina no control-plane; efeito real no agent runtime (HTTP) |
| Fronteira | container: status/read/write/change/gen/push · control-plane: promote/scale |
| Envelope | sucesso/recuperável retornam; fatal levanta `FatalToolError` |
| Contexto | `get_current_context()` (fixo→JWT), injetado via `InjectedToolArg` |
| Trava de iteração | `recursion_limit` do LangGraph |
| `promote` | unificado (substitui publish + back_version); commit é a verdade |
| `scale` | recusa stateful (recuperável) |
| `gen` | runtime edita manifest (verdade); control-plane espelha no banco |

## 7. FORA DE ESCOPO (próximos)

- Auth real (JWT no `get_current_context()`) — gate 🔴 antes de expor publicamente.
- Scaffold: template genérico vs composto pela IA.
- Quotas: números reais por tier.
- Schema exato do secret manager / resolver de segredos.
- Materialização dual de stateful (container local vs gerenciado Azure) no `promote`.
