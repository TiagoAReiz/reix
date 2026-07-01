# Modelo de Domínio & Conexões — Control-Plane

> Consolida as decisões fundadoras de implementação: como o control-plane se
> conecta aos sandboxes, onde o agente roda, a stack/estrutura do repo, e o
> modelo de entidades (o vocabulário do sistema).
> Companion: arquitetura_consolidada.md, viabilidade_economica.md,
> ressalvas_decisoes.md, tools_refinadas.md

---

## 1. STACK DO CONTROL-PLANE

**Linguagem: Python** (FastAPI). Razão: o control-plane é majoritariamente
orquestração de IA (loop de agente, tool calling, integração LLM), onde o
ecossistema Python é o mais maduro e onde o dev já pratica. FastAPI dá a API
async que o orquestrador precisa. A linguagem do control-plane NÃO amarra os
apps gerados (que podem ser JS,Java,etc) — ele fala com tudo via adaptadores.

🔴 A revisitar: se "uma linguagem só no projeto" for prioridade, Node/TS é
defensável. Pra orquestração de agente, recomendação pende a Python.

---

## 2. CONEXÃO CONTROL-PLANE ↔ SANDBOX (como alcançar os arquivos)

Dois mecanismos, um por ambiente — cada um onde brilha.

### Criação → Opção A: agent runtime no sandbox

Cada container de sandbox roda, além do app, um **agent runtime**: um servidor
HTTP pequeno que expõe operações (edita arquivo, lê, roda comando). O
control-plane chama essa API. As tools viram requisições HTTP pro container.

```
LLM decide change(arquivo, old, new)
  → agent/loop emite a intenção
  → tools/executor valida + autoriza
  → adapters/sandbox traduz em HTTP
  → AGENT RUNTIME no container edita o arquivo
  → dev server vê a mudança → hot-reload → preview atualiza
```

**Distinção crucial:**
- **Agente** = o LLM que raciocina (cérebro, no control-plane).
- **Agent runtime** = as "mãos" no container (sem inteligência, só executa
  ordens simples no filesystem/shell local dele).

**Por que A:** funciona igual em Compose e k8s (só muda a URL), mantém
isolamento, escala (cada sandbox é autônomo). Padrão da indústria (E2B, Replit).

### Prod → Opção C: via Git + CI/CD

Publicar não usa agent runtime. O Git É a conexão:
```
publish → promote(commit, prod) → merge na master
  → CI/CD lê o repo → builda → deploy k8s
```
Tudo bem ser "lento" (assíncrono, ninguém assistindo). A criação nunca toca Git
pra refletir no preview (seria lento); a prod nunca usa agent runtime
(acoplamento desnecessário). Os dois se encontram no `push`: a criação trabalha
via agent runtime E commita no Git em paralelo, então no publish o repo já está
pronto pro caminho C assumir.

---

## 3. ONDE O AGENTE RODA (assincronismo)

**Decisão: módulo dentro do control-plane AGORA, desenhado pra virar serviço
separado DEPOIS.** Isolado atrás de interface limpa (inicia sessão / manda
mensagem / recebe eventos). No MVP roda junto; em escala extrai sem reescrever.

### Como fica assíncrono sem bloquear

O loop do agente é longo (segundos/minutos, várias chamadas LLM). Não pode rodar
"dentro" da requisição HTTP. Solução: **tarefa em background + eventos**.

```
usuário manda prompt
  → control-plane INICIA tarefa de agente em background
  → responde na hora: "comecei" (+ id de sessão)
  → usuário abre canal de eventos (SSE/WebSocket) pra essa sessão
  → tarefa roda o loop, EMITE evento a cada passo:
    "li arquivo X" / "criei serviço Y" / "terminei"
  → usuário vê progresso em tempo real
```

**Por que funciona sem serviço separado:** o modelo async do Python
(asyncio/FastAPI) deixa a tarefa "esperar" o LLM sem bloquear o control-plane,
que segue atendendo outras requisições. Ganha concorrência sem paralelismo de
processos.

**Limite (onde vira serviço depois):** async resolve esperar I/O (esperar o
LLM); não resolve processar CPU pesada. Como o loop é quase todo espera de I/O,
aguenta bastante antes de precisar separar.

**Robustez — estado persistido:** o estado da sessão (qual app, em que passo,
histórico da conversa com o LLM) NÃO vive só na memória da tarefa. É persistido
(Redis pra estado vivo + banco pra histórico), pra sobreviver a restart e poder
retomar o loop. Conecta com a fila: sessões de agente podem ser enfileiradas
como os builds (máx de loops concorrentes, resto espera, feedback por tier).

---

## 4. ESTRUTURA DO REPO DO CONTROL-PLANE

```
control-plane/
├─ api/                  # porta de entrada (FastAPI): rotas, sessões
│   ├─ routes/           # endpoints (criar app, mandar prompt, status)
│   └─ sessions/         # estado da sessão de criação ativa
│
├─ agent/                # o loop do LLM (módulo agora, serviço depois)
│   ├─ loop.py           # o agentic loop
│   ├─ prompts/          # system prompts, instruções
│   └─ context.py        # monta o contexto que vai pro LLM
│
├─ tools/                # as tools (o contrato)
│   ├─ registry.py       # catálogo de tools + schemas
│   ├─ definitions/      # cada tool: gen, change, push, promote, status...
│   └─ executor.py       # O EXECUTOR: valida, autoriza, materializa
│
├─ catalog/              # tenant catalog
│   ├─ models.py         # entidades de gestão
│   └─ repository.py     # acesso ao registro (Postgres do control-plane)
│
├─ provisioning/         # cria app: repo, scaffold, branches, registro
│
├─ adapters/             # INTERFACES TROCÁVEIS (coração da portabilidade)
│   ├─ git/              # interface + impl GitHub (Gitea depois)
│   ├─ runtime/          # interface + impl Compose (k8s depois)
│   ├─ secrets/          # interface + impl .env (Infisical depois)
│   ├─ llm/              # interface + impl LiteLLM
│   └─ sandbox/          # interface pro agent runtime no container
│
├─ domain/               # conceitos centrais puros (app, serviço, sessão)
│
└─ infra/                # config, conexões, bootstrap, docker
```

**Lógica:** cada módulo da arquitetura vira uma pasta. Os `adapters` ficam
isolados (onde mora toda troca GitHub→Gitea, Compose→k8s). O `domain/` no centro
guarda conceitos puros sem dependência externa (testável). Aplicação direta de
separação de camadas + interface-as-contract + DI: `agent` e `executor` dependem
de INTERFACES nos adapters, não de implementações.

---

## 5. MODELO DE ENTIDADES (o vocabulário do sistema)

### Mapa de relações

```
Tenant ──< User
  │
  └──< App ──< Service
        │
        ├──< Session     (criação ao vivo)
        │
        └──< Deployment  (publicações pra prod)
```

### Tenant — o cliente/empresa
```
id, nome
tier (free/pro/enterprise)   → define quotas e comportamento de sleep
quotas                        → quantos apps, quanto recurso
relação: tem muitos Apps, muitos Users
```

### User — a pessoa
```
id, email, nome
tenant_id                     → a que cliente pertence
role                          → admin do tenant? membro?
```
✅ Decidido: User e Tenant SEPARADOS desde o MVP (não retrabalhar auth depois).

### App — a entidade central
```
IDENTIDADE E PROPRIEDADE
├─ id (UUID), nome, slug (URL-safe pro subdomínio)
├─ tenant_id    → o app pertence ao TENANT (não à pessoa)
└─ created_by   → qual user criou (pode diferir de quem usa)

REFERÊNCIAS DE INFRA (referências, NÃO credenciais)
├─ repo_url        → o repositório no GitHub
├─ database_ref    → identificador do banco ("db_acme"), NÃO connection string
├─ sandbox_ref     → qual container serve a criação (se sessão ativa)
└─ prod_ref        → onde roda em prod (namespace k8s)
   ⚠️ credenciais reais vivem no secret manager, resolvidas em runtime.
      Se o registro vazar, não vaza segredo nenhum — só nomes.

ESTADO (dois estados distintos, NÃO misturar num "status" só)
├─ lifecycle_status → criando / ativo / arquivado / deletado
│                     (onde o app está na vida)
└─ runtime_status   → dormindo / acordando / no-ar / sem-deploy
                      (o que a infra está fazendo agora)

CONFIGURAÇÃO
├─ tier          → herda do Tenant, com possível override no App
├─ ai_config     → qual modelo de IA esse app usa (conecta com BYOK)
└─ sleep_policy  → quão agressivo o sleep (deriva do tier)

MANIFESTO (espelho — ver seção 6)
└─ manifest_atual → <json> da topologia atual (cache de leitura)

relações: tem muitos Service, muitas Session, muitos Deployment
```

### Service — um componente do app (o que `gen` cria)
```
IDENTIDADE
├─ id, app_id
├─ name    → LOCAL ao app ("db","api"). Dois apps podem ter "db", não colidem.
│            É como a IA e o código se referem ("conecta no db").
├─ type    → do catálogo curado (postgres/redis/rabbitmq/node/python/java)
└─ nature  → stateless | stateful (derivado do type, mas EXPLÍCITO)
             dispara comportamentos: scale, sleep, tradução prod.

REGISTRY (descoberta entre serviços)
├─ internal_url → nome estável ("db.internal"), NÃO um IP
│                 (resolve igual em Compose/k8s/gerenciado)
├─ port
└─ env_exposed[] → variáveis que oferece (DB_HOST, DB_PORT, DB_PASSWORD...)
                   ⚠️ os NOMES. Os VALORES (senha) vêm do secret manager.

ESTADO POR AMBIENTE
└─ instances → { criação: {...}, prod: {...} }
   ✅ Decidido: UMA declaração única serve dev E prod (não um service por
      ambiente). A declaração é única; a materialização é por ambiente
      (container em dev, gerenciado Azure em prod). Guarda o estado do app.
```

### Session — uma sessão de criação com a IA
```
id, app_id, user_id
status        → ativa / processando / ociosa / encerrada
sandbox_ref   → qual container de sandbox está servindo
histórico     → a conversa com o LLM (o contexto) — PERSISTIDO no banco
estado_loop   → em que passo está — pra retomar após restart
✅ Decidido: persistir histórico no banco pra continuar o loop.
   Sandbox é descartável; a sessão sobrevive. Sleep do sandbox conecta aqui.
```

### Deployment — uma publicação pra prod
```
id, app_id
commit        → a fonte de verdade (commit, não "versão")
status        → na fila / buildando / no ar / falhou / rollback
environment   → prod
É o que promote cria e o CI/CD materializa.
```

---

## 6. O MANIFESTO (AppManifest) — armazenamento duplo

**O que é:** a "planta" do app — a topologia COMPLETA e traduzível (serviços +
ligações + ordem + env vars + recursos por ambiente + ponto da história). Mais
que a lista de Services (inventário): é a planta executável (como tudo se monta).

**Papel:** fonte ÚNICA da qual os dois ambientes derivam.
```
              AppManifest (uma planta)
              /                      \
     tradução criação          tradução prod
            /                          \
   docker-compose.yml             manifestos k8s
   (casa de dev)                  (casa de prod)
```
Como ambos saem do MESMO manifesto, não divergem na estrutura — só nos
parâmetros que mudam por ambiente. Mata "funciona ao criar, quebra ao publicar".

### Armazenamento duplo (decidido: ambos)

```
NO REPO (app.manifest.json)        NO BANCO (campo manifest_atual no App)
═══════════════════════════        ══════════════════════════════════════
papel: FONTE DE VERDADE            papel: ESPELHO / CACHE DE LEITURA
versionado pelo Git (todo commit)  só estado atual (sobrescrito)
lê: CI/CD no deploy, rollback      lê: UI, roteamento, control-plane dia a dia
escreve: IA via push (c/ código)   escreve: sistema, a cada push (copia do repo)
se sumir: problema sério           se sumir: re-espelha lendo do repo
resolve: versionamento + deploy    resolve: consulta rápida (ms vs rede externa)
```

**Regra de ouro: o REPO sempre manda.** Se banco e repo discordarem, a verdade é
o repo; o banco é corrigido lendo de lá. Nunca o contrário.

**Sincronia (no push):**
```
1. atualiza app.manifest.json com a topologia atual
2. commita TUDO junto no repo (código + manifesto)   ← grava a VERDADE
3. lê o manifesto recém-commitado
4. copia pro campo manifest_atual no banco            ← atualiza o ESPELHO
```
Ordem importa: grava a verdade primeiro, espelha depois. Se 4 falhar, o espelho
fica velho um momento, mas a verdade está salva (re-sincroniza depois).

**Relação com Services:** Services são as entidades VIVAS (estado atual,
manipuladas por `gen`). O manifesto versionado é o HISTÓRICO de plantas. No push,
os Services são "fotografados" no manifest.json (verdade) e espelhados no banco
(cache). Service = presente vivo; manifestos versionados = linha do tempo.
Rollback via commit restaura a topologia daquele ponto (planta guardada).

---

## 7. CADEIA COMPLETA DE UMA TOOL (ex: gen + change)

```
LLM decide gen("postgres","db")
  → agent/loop emite intenção
  → tools/executor:
     1. valida schema (postgres no catálogo?)
     2. valida autorização (tier permite mais um serviço?)
     3. valida estado (nome "db" livre nesse app?)
     4. consulta catalog (qual app/repo)
  → adapters/runtime materializa (bloco no Compose de criação)
  → cria entidade Service no banco + atualiza registry
  → resultado estruturado volta { ok, data/error, hint }
  → agente incorpora → próximo passo (com trava de iteração)

[mais tarde] push:
  → atualiza app.manifest.json → commita (código+manifesto) → espelha no banco
```

---

## 8. ESTADO ATUAL: DECIDIDO vs ABERTO

**Decidido e estável:**
- Stack Python/FastAPI no control-plane
- Conexão: agent runtime na criação (A), Git/CI-CD na prod (C)
- Agente como módulo isolado (extraível pra serviço depois)
- Async via tarefa background + eventos (SSE/WebSocket), estado persistido
- Estrutura de repo por módulo, adapters isolados
- User e Tenant separados
- Entidades: Tenant, User, App, Service, Session, Deployment
- Service: declaração única (dev+prod), guarda estado do app
- Manifesto: armazenamento duplo (repo = verdade, banco = espelho)
- Session: histórico persistido no banco pra retomar loop

**Aberto (próximos):**
- 🔴 Liberações de acesso seguro (auth: user/tenant → permissões, acesso
  mediado aos apps) — RESERVADO como próximo tema
- 🔴 Assinaturas finais das tools (promote unificado, schemas exatos)
- 🔴 Formato exato do contrato agente↔executor (JSON de tool call e resultado)
- ⚠️ Quotas: números reais por tier
```
