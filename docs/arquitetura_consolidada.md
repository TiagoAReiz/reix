# Arquitetura da Plataforma — Documento Consolidado

> Visão completa da arquitetura pretendida: o control-plane (o que construímos),
> os apps gerados (o que a IA produz), as tools, os ambientes e os fluxos.
> Documento de contexto — guarda o raciocínio inteiro num lugar.
> Companion: `ressalvas_decisoes.md` (decisões e dívidas) e `tools_refinadas.md`
> (assinaturas detalhadas das tools).

---

## 1. O QUE É A PLATAFORMA

Uma plataforma que gera apps completos via IA — fugindo do molde fixo do Lovable
(arquitetura simples e travada). Aqui a IA **compõe a arquitetura** que o app
precisa (ecommerce escalável, múltiplos serviços, etc.) escolhendo de um catálogo
curado de serviços, em vez de encaixar tudo num esqueleto único.

**Princípios fundadores:**
- A IA escolhe de um **menu curado**, não inventa infraestrutura do zero.
- **Isolamento por app**: cada app tem repo, banco e container próprios.
- **As tools expressam intenção, não comandos de runtime** (a IA não sabe se
  está rodando em Compose ou k8s).
- **Comprar robustez, construir só o diferencial** (operável solo).
- **Self-host como requisito de design** (empresa roda na cloud dela, no
  compliance dela).
- **BYOK**: cliente traz a própria chave de IA (custo não passa por nós).

---

## 2. AS DUAS CAMADAS

A confusão mais comum é misturar duas arquiteturas distintas:

```
CAMADA 1 — A FÁBRICA (build-time)        CAMADA 2 — O PRODUTO (run-time)
o control-plane + o agente que gera      o app que a IA gerou e que roda
```

- **Camada 1 (control-plane)**: o que NÓS construímos. O agente de IA, a
  orquestração, o versionamento, o provisionamento. É o diferencial.
- **Camada 2 (apps gerados)**: o que a IA produz. Cada app tem sua própria
  arquitetura (serviços, banco, código), roda isolado, e é publicado em prod.

---

## 3. O CONTROL-PLANE POR DENTRO

O control-plane é o **maestro**: não toca instrumento (não roda app, não é banco,
não é Git), coordena todos. Fala com tudo que é externo via interfaces trocáveis.

```
┌─────────────────────────────────────────────────────────────┐
│  CONTROL-PLANE                                              │
│                                                            │
│  ┌──────────────────────────────────────────────────┐     │
│  │  API / ORQUESTRADOR (porta de entrada)            │     │
│  │  recebe requisições do editor, coordena os módulos │     │
│  └───────────────────────┬──────────────────────────┘     │
│                          │                                 │
│   ┌──────────┬───────────┼───────────┬──────────────┐      │
│   ▼          ▼           ▼           ▼              ▼      │
│ ┌──────┐ ┌────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐ │
│ │AGENTE│ │TENANT  │ │PROVISIO-│ │EXECUTOR  │ │RESOLVER  │ │
│ │ DE IA│ │CATALOG │ │NAMENTO  │ │DE TOOLS  │ │DE SEGREDOS│ │
│ └──────┘ └────────┘ └─────────┘ └──────────┘ └──────────┘ │
│                                                            │
│  ┌──────────────────────────────────────────────────┐     │
│  │  ADAPTADORES (interfaces trocáveis pro externo)    │     │
│  │  Git │ Runtime │ Secrets │ LLM │ CI/CD             │     │
│  └──────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### Módulos

**API / Orquestrador** — porta de entrada. Recebe "usuário pediu X", decide qual
módulo aciona. Sem lógica de negócio própria; coordena. Mantém a sessão de
criação (qual app, qual usuário, qual estado).

**Agente de IA** — o agentic loop: o LLM no laço com as tools. Recebe prompt +
contexto, **decide** quais tools chamar. Não executa nada — só raciocina e emite
intenções. Puro raciocínio.

**Executor de tools** — materializa as decisões do agente. Recebe a intenção
("chama gen com X"), valida, autoriza, e traduz pro mundo real via adaptadores.
É o ÚNICO ponto onde intenção vira efeito. (Detalhe na seção 6.)

**Tenant catalog** — registro central "qual app é de quem e mora onde":
`app_id → {repo, database, container, tier, região, modelo de IA, dono}`. Fonte
de verdade sobre os apps. Todo módulo consulta.

**Provisionamento** — roda quando um app nasce: cria o repo (via adaptador Git),
prepara o ambiente de criação, registra no catalog. Setup, uma vez por app.

**Resolver de segredos** — busca as chaves (API key BYOK do cliente) no secret
manager, server-side, no momento de usar. Nunca deixa vazar pro agente ou browser.

### Adaptadores (interfaces trocáveis)

Cada um esconde "qual implementação concreta" do resto do control-plane:

| Adaptador | Criação / hoje | Prod / depois |
|-----------|----------------|---------------|
| Git | GitHub (repo privado) | + Gitea (self-host puro) |
| Runtime | docker compose | k8s (AKS) |
| Secrets | .env protegido | Infisical / Key Vault |
| LLM | LiteLLM (BYOK) | LiteLLM (BYOK) |
| CI/CD | trigger → build → deploy | idem em k8s |

É o que permite trocar GitHub→Gitea ou Compose→k8s sem tocar nos módulos de cima.

---

## 4. OS APPS GERADOS (camada 2)

Cada app é composto pela IA via tools, com isolamento total.

### Anatomia de um app

```
APP "loja-acme"
├─ repo próprio (GitHub privado)
│   ├─ branch dev    (IA trabalha, cada push = commit)
│   └─ branch master (prod; promote leva pra cá → CI/CD)
├─ database próprio (isolamento real, não RLS compartilhado)
├─ serviços (escolhidos do catálogo curado)
│   ├─ stateless: node / python / java
│   └─ stateful:  postgres / redis / rabbitmq
├─ service registry (serviços se descobrem, conn string flui sem hardcode)
└─ descrição interna (a fonte que gera Compose e k8s)
```

### Catálogo de serviços (menu curado)

```
STATELESS (escala horizontal fácil = multiplica cópias)
├─ node · python · java

STATEFUL (escala difícil = operação especializada)
├─ postgres · redis · rabbitmq
   └─ materialização varia por ambiente (ver seção 5)
```

### Service registry

A peça que faz serviços se descobrirem sem hardcode. Quando a IA gera um postgres
e depois um node, o registry injeta `DB_HOST`, `DB_PORT` etc no node
automaticamente. O código lê `process.env.DB_HOST` — nunca um IP/URL fixo. Em
criação aponta pro container; em prod, pro gerenciado Azure. Mesmo código.

---

## 5. OS DOIS AMBIENTES

NÃO são três. "Local" e "preview ao vivo" são a mesma coisa: o ambiente de
CRIAÇÃO. O preview em tempo real é uma PROPRIEDADE dele, não um ambiente.

```
AMBIENTE DE CRIAÇÃO (Compose + hot-reload + preview ao vivo)
  ├─ dev rodando local e criando com a IA
  ├─ usuário criando na instância self-hosted privada
  └─ mesmo runtime, mesma experiência, mesma mecânica
        │  "publicar" → trigger CI/CD
        ▼
PROD (k8s / Azure)
  └─ app publicado, escalado, estável; stateful → gerenciado Azure
```

### O que muda entre criar e produzir (só isso a tradução resolve)

1. **Modo de execução**: criação = dev (hot-reload, código não-otimizado);
   prod = artefato compilado (bundle otimizado).
2. **Destino do stateful**: criação = container; prod = gerenciado Azure
   (Postgres → Azure Database for PostgreSQL, Redis → Azure Cache, etc).
3. **Recursos/escala**: criação = 1 container, 1 host; prod = k8s com réplicas.

→ A **topologia NÃO muda** (mesmos serviços, ligações, env vars). Só os
parâmetros de execução. É isso que evita "funciona ao criar, quebra ao publicar".

### Mecânica do preview ao vivo

- Container de dev sempre-on por sessão de criação, com hot-reload (Vite).
- A IA edita arquivos via tools → dev server detecta → recarrega.
- Preview = iframe apontando pro container de dev + opcional WebSocket de eventos.
- Em escala: cada usuário criando = um container de dev vivo (frota efêmera).

---

## 6. EXECUÇÃO DE TOOLS (o coração)

**Princípio: o agente decide, o executor faz.** O agente nunca toca o mundo real
— só produz intenções. O executor pega a intenção e a transforma em efeito.

```
┌─────────────┐   intenção    ┌──────────────┐   efeito    ┌──────────┐
│   AGENTE    │ ────────────► │   EXECUTOR   │ ──────────► │  MUNDO   │
│ (raciocínio)│  "chama gen"  │ (valida +    │  cria       │ (runtime,│
│             │ ◄──────────── │  materializa)│ ◄────────── │  git...) │
└─────────────┘   resultado   └──────────────┘   retorno   └──────────┘
```

### Por que separar
- **Testabilidade**: testa o agente sem criar recursos; testa o executor sem LLM.
- **Segurança**: o executor é o ÚNICO ponto onde efeitos acontecem → uma porta
  pra validar/autorizar/limitar.
- **Controle**: dá pra pausar entre "decidiu" e "fez" (confirmação, log, permissão).

### Ciclo de vida de uma tool call

```
1. Agente emite intenção (nome da tool + args) — só dados, sem efeito
2. Executor VALIDA antes de tudo:
   ├─ schema (args batem? type está no menu curado?)
   ├─ autorização (tenant/tier pode fazer isso?)
   └─ estado (faz sentido agora? nome livre?)
3. Executor consulta tenant catalog (qual app/repo/banco)
4. Executor MATERIALIZA via adaptador (gen → runtime → bloco compose/k8s)
5. Executor atualiza service registry (se aplicável)
6. Executor captura resultado + erros
7. Agente incorpora resultado → decide próximo passo (com trava de iteração)
```

### Formato do resultado (o que dá inteligência ao loop)

```
SUCESSO:           { ok: true, data: {...} }
ERRO RECUPERÁVEL:  { ok: false, error, recoverable: true, hint: "..." }
ERRO FATAL:        { ok: false, error, recoverable: false }
```

- **Recuperável**: o agente tenta de novo diferente (nome duplicado, validação).
  Devolve dica, deixa ele se corrigir — mas com TRAVA DE ITERAÇÃO (teto de
  passos), senão vira loop infinito gastando tokens.
- **Fatal**: agente não resolve sozinho (adaptador caiu, secret não achado, tier
  estourado) → para o loop, escala pro humano.

### Naturezas de tool

| Natureza | Tools | Tratamento |
|----------|-------|------------|
| Leitura (idempotente, segura) | `status`, `read` | executa direto, pode paralelizar |
| Escrita (efeito real) | `gen`, `change`, `write`, `push`, `scale` | validação rigorosa, log, sequencial |
| Escrita em prod (irreversível) | `promote` (prod) | + confirmação/gate |

**Paralelo vs sequencial:** leitura paraleliza à vontade; escrita sempre
sequencial no início (evita condição de corrida), otimiza depois se precisar.

---

## 7. AS TOOLS (resumo — detalhe em tools_refinadas.md)

```
status(scope?)              → lê estado (services, files, git, environments)
gen(service_type, name)     → cria serviço do catálogo + registra no registry
read(path) / write(path)    → lê / cria arquivo
change(path, old, new)      → edita arquivo (valida unicidade)
push(commit_name, files?)   → commita no branch dev
promote(commit, env)        → aponta ambiente pro commit (fwd-only ou rollback)
scale(service, env, n)      → escala stateless (só faz sentido em k8s/prod)
```

Mudanças-chave vs ideia original:
- **service registry** (faltava): serviços se descobrem, conn string sem hardcode.
- **`promote` unificado**: substitui publish + back_version. Commit = fonte de
  verdade; "versão" vira tag amigável.
- **`status`/`read`/`write` adicionados**: agente não pode agir cego.

---

## 8. VERSIONAMENTO E REPOSITÓRIOS

- **GitHub gerenciado, repo PRIVADO por app**, criado via API pela plataforma.
- **Repo por app** (não monorepo): coerente com isolamento total. `promote` por
  commit limpo, "exportar app" = dar o repo, blast radius mínimo. Apps de IA são
  independentes (não compartilham código) → monorepo resolveria problema inexistente.
- **Branches**: `dev` (IA trabalha) + `master` (prod).
- **Criação**: provisionamento cria repo via API → scaffold base → branches →
  registra no catalog. Tudo programático, antes da IA gerar código.
- **GitHub atrás de interface trocável** (adaptador Git): self-host puro exige
  não mandar código pro GitHub.com → aí entra adaptador Gitea, sem mexer no resto.

---

## 9. FLUXOS PRINCIPAIS

### Criar um app
```
usuário cria app → provisionamento:
  cria repo (GitHub API) → scaffold → branches dev/master
  → registra no tenant catalog → prepara ambiente de criação (Compose)
```

### Gerar/editar com IA (o loop em criação)
```
usuário: "adiciona login"
  → orquestrador monta contexto (consulta catalog)
  → agente raciocina, emite intenções (tools)
  → executor valida + materializa via adaptadores
     (change arquivo, gen serviço, push commit)
  → preview ao vivo reflete em tempo real (hot-reload)
  → agente avalia resultados → próximo passo ou termina (trava de iteração)
```

### Publicar pra prod
```
usuário clica "publicar"
  → promote(commit, prod) → merge na master
  → trigger CI/CD:
     1. roda build de produção DE VERDADE + valida (pega bug modo-dev→build)
     2. camada de tradução converte descrição → manifesto k8s
     3. stateful aponta pra gerenciado Azure
     4. deploy no AKS
  → app rodando em prod, escalado
```

### Chamar IA dentro de um app gerado (BYOK)
```
app pede algo à IA (server-side)
  → resolver de segredos busca a key do tenant (Infisical)
  → LiteLLM (escolhe modelo conforme config do app)
  → provedor (OpenAI/Anthropic/Google) — paga o cliente
  → key NUNCA toca o browser nem o código do app
```

---

## 10. STACK (criação local, docker-compose)

```yaml
services:
  traefik:        # borda: TLS, roteamento, egress allowlist
  control-plane:  # NOSSO código: orquestrador + agente + catalog +
                  #   provisionamento + executor + resolver
  litellm:        # LLM gateway: multi-modelo, fallback, metering
  postgres:       # database-per-app
  # secrets: .env protegido → Infisical depois
  # apps gerados: containers dinâmicos por app (criados via tools)
  # fase 2: grafana + prometheus + loki (observabilidade, tenant_id em tudo)
```

Mesmo arquivo é a base do self-host. Prod traduz pra k8s via CI/CD.

---

## 11. CAMINHO DE EVOLUÇÃO

```
FASE 0 (agora)     → docker-compose de criação, local, grátis, MVP/demo
FASE 1 (publicar)  → CI/CD + k8s pra prod (AKS quando alguém custear)
FASE 2 (crescer)   → observabilidade, metering por tenant, quotas
FASE 3 (self-host) → empacota (Helm); empresa roda na cloud dela
```

---

## 12. MAPA MENTAL DO TODO

```
                    ┌──────────────────────────────────┐
                    │  CONTROL-PLANE (nós construímos)  │
                    │  maestro: orquestra tudo          │
                    └────────────────┬──────────────────┘
                                     │ via adaptadores
        ┌────────────────────────────┼────────────────────────────┐
        ▼                            ▼                            ▼
  ┌───────────┐              ┌──────────────┐            ┌──────────────┐
  │ Git       │              │ Runtime      │            │ Secrets/LLM  │
  │ (GitHub)  │              │ Compose→k8s  │            │ Infisical/   │
  │ repo/app  │              │ criação→prod │            │ LiteLLM(BYOK)│
  └───────────┘              └──────┬───────┘            └──────────────┘
                                    │
                                    ▼
                          ┌──────────────────┐
                          │  APPS GERADOS    │
                          │  (camada 2)      │
                          │  isolados:       │
                          │  repo + banco +  │
                          │  container/app   │
                          └──────────────────┘

Fio condutor: a IA COMPÕE a arquitetura via tools (intenção); o control-plane
TRADUZ pro runtime real; cada app vive ISOLADO; criação é leve (Compose) e prod
é escalável (k8s); o cliente pode rodar TUDO na infra dele (self-host).
```
