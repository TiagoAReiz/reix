# Tools do Agente de Infraestrutura — Especificação Refinada

Princípio central: **as tools expressam INTENÇÃO sobre o app, nunca comandos de runtime.** A IA diz "quero um postgres"; o control-plane decide se isso vira serviço no Compose (local), no Swarm, ou Deployment no k8s. A IA nunca sabe qual runtime está embaixo. É isso que mantém Compose→Swarm→k8s trocável.

---

## O MODELO DE ESTADO (o que existe antes das tools)

Toda tool opera sobre um estado central por app. Sem isso o agente age cego.

APP STATE (o "manifesto vivo" do app, guardado pelo control-plane)

├─ app\_id, tenant\_id, tier

├─ services\[\]          ← o service registry (quem existe e como se liga)

│   └─ { name, type, internal\_url, port, env\_exposed\[\], status }

├─ files\[\]             ← árvore de arquivos do código

├─ git                 ← { branch, last\_commit, dirty? }

├─ environments

│   ├─ dev  → { running\_commit, resources, service\_instances }

│   └─ prod → { running\_commit, resources, service\_instances }

└─ runtime\_target      ← compose | swarm | k8s (trocável, a IA não vê)

**Service registry** é a peça que faltava na tua ideia original. É o catálogo interno de "o que este app tem e como os serviços se descobrem". Quando a IA gera um postgres e depois um node, é o registry que faz a connection string do postgres chegar no node automaticamente, sem hardcode — e faz isso diferente em dev e prod (cada ambiente tem sua instância).

---

## CATÁLOGO DE SERVIÇOS (menu curado — a IA escolhe, não inventa)

STATELESS (escala horizontal fácil \= multiplica cópias)

├─ node

├─ python

└─ java

STATEFUL (escala difícil \= operação especializada)

├─ postgres

├─ redis

└─ rabbitmq

Regra de materialização do stateful (a síntese do trade-off):

ambiente local/dev      → container nosso (simples, dados descartáveis)

self-host do cliente    → container, operação é responsabilidade do host

prod gerenciada (k8s)   → aponta pra gerenciado (Neon/RDS) — não viramos DBA

---

## AS TOOLS

### 1\. `status(scope?)` — LER ESTADO (a tool que faltava)

**Por que existe:** o agente precisa ver antes de agir. No agentic loop, ler o estado é metade das chamadas. Sem isso a IA decide errado.

status(scope?: "services" | "files" | "git" | "environments" | "all")

→ {

    services: \[

      { name: "db", type: "postgres", internal\_url: "db.internal",

        port: 5432, status: "running" },

      { name: "api", type: "node", internal\_url: "api.internal",

        port: 3000, status: "running" }

    \],

    git: { branch: "dev", last\_commit: "a1b2c3", dirty: true },

    environments: {

      dev:  { running\_commit: "a1b2c3", cpu: "0.5", mem: "512m" },

      prod: { running\_commit: "f9e8d7", cpu: "2", mem: "2g" }

    }

  }

---

### 2\. `gen(service_type, name)` — CRIAR SERVIÇO

**Mudanças vs tua versão:** recebe `name` (pode ter 2 bancos no mesmo app), registra no service registry, e expõe as variáveis de conexão pros outros serviços automaticamente.

gen(service\_type: "postgres"|"redis"|"rabbitmq"|"node"|"python"|"java",

    name: string)

→ {

    result: "success",

    service: {

      name: "db",

      type: "postgres",

      internal\_url: "db.internal",     ← nome estável, não IP

      port: 5432,

      env\_exposed: \["DB\_HOST", "DB\_PORT", "DB\_USER", "DB\_PASSWORD", "DB\_NAME"\]

                                        ← injetadas automático nos outros serviços

    }

  }

**Situação pra validar:**

IA chama `gen("postgres", "db")` e depois `gen("node", "api")`. O registry faz com que o container "api" já nasça com `DB_HOST=db.internal` etc no ambiente. A IA escreve `process.env.DB_HOST` no código — nunca um IP ou URL hardcoded. Em dev aponta pro postgres de dev; em prod, pro de prod. Mesmo código.

❓ *O `internal_url` ser um nome estável (não IP) é o que faz funcionar igual em compose/swarm/k8s — todos resolvem nome de serviço. Faz sentido?*

---

### 3\. `change(path, old_str, new_str)` — EDITAR ARQUIVO

**Igual à tua, com guarda:** valida que `old_str` existe e é único antes de trocar. Se não for único ou não existir, falha em vez de corromper.

change(path: string, old\_str: string, new\_str: string)

→ { result: "success", changes: "3 linhas alteradas em src/api.js" }

  | { result: "error", reason: "old\_str não encontrado" | "old\_str ambíguo" }

Complementos provavelmente necessários: `write(path, content)` pra criar arquivo novo, e `read(path)` pra IA ver o conteúdo antes de editar.

---

### 4\. `push(commit_name, files?)` — COMMITAR NO GIT

**Igual à tua.** Sobe alterações. Sem `files` \= tudo; com `files` \= só aqueles.

push(commit\_name: string, files?: string\[\])

→ { result: "success", commit: "a1b2c3" }

---

### 5\. `promote(commit, to_env)` — PROMOVER ENTRE AMBIENTES

**Esta é a fusão que sugeri:** substitui `publish` \+ `back_version` por uma operação só. Promover é sempre "aponta este ambiente pro commit X". A diferença é só a validação.

promote(commit: string, to\_env: "dev" | "prod")

→ {

    result: "success",

    env: "prod",

    now\_running: "a1b2c3",

    previous: "f9e8d7"      ← guardado, permite voltar

  }

**Por que fundir:**

- **commit é a única fonte de verdade** (imutável, ordenável). "Versão" vira só uma tag amigável que aponta pra um commit, se você quiser.  
- `publish` (subir versão nova) \= `promote(commit_mais_novo, "prod")`  
- `back_version` (voltar) \= `promote(commit_antigo, "prod")`  
- Uma operação, dois usos. Menos conceito, menos bug.

**A trava "só publica mais recente" vira política, não tool separada:**

promote pode rodar em 2 modos:

  \- forward-only (default em prod): rejeita commit mais antigo que o rodando

  \- rollback (explícito): permite voltar, exige flag intencional

**Situação pra validar:**

Deu bug em prod. Você não precisa de uma tool diferente — chama `promote(commit_anterior, "prod")` com a flag de rollback. Mesmo mecanismo de sempre, só apontando pra trás. O artefato do commit antigo ainda existe (imutável), então é instantâneo, não rebuilda.

❓ *Trocar publish+back\_version por um promote só te parece mais limpo, ou você prefere manter as duas tools separadas por clareza de intenção?*

---

### 6\. `scale(service, env, replicas)` — ESCALAR (entra com Swarm/k8s)

**Não existe na tua lista, mas é onde a escala horizontal aparece.** No Compose puro não faz nada útil (um host só); ganha sentido quando o runtime vira Swarm ou k8s.

scale(service: string, env: "prod", replicas: number)

→ { result: "success", service: "api", replicas: 4 }

**Situação pra validar:**

App de ecommerce em Black Friday. IA (ou regra automática) chama `scale("api", "prod", 8)`. No runtime Compose isso é no-op/limitado; no Swarm ou k8s, sobe 8 cópias do "api" distribuídas em várias máquinas. A IA chama igual — o control-plane sabe traduzir conforme o runtime\_target.

❓ *Só faz sentido escalar stateless assim. Postgres não escala por replicas simples. A tool deve recusar `scale` em serviço stateful e orientar pro caminho gerenciado?*

---

## O FLUXO DEV → PROD (resolvendo teu Problema 1\)

1\. IA trabalha no ambiente DEV

   ├─ gen() cria serviços (instâncias de dev, recursos baixos)

   ├─ change()/write() edita código

   └─ status() verifica

2\. push("feat: carrinho")  → commit a1b2c3 no branch dev

3\. promote(a1b2c3, "prod")

   ├─ valida: a1b2c3 é mais novo que o prod atual? (forward-only)

   ├─ dispara CI/CD: merge na master

   └─ CI/CD materializa o MESMO artefato no ambiente prod

       (mesma topologia, recursos de prod, instâncias de prod)

4\. Se quebrar: promote(commit\_anterior, "prod", rollback=true)

**O princípio que mata o "funciona em dev, quebra em prod":** dev e prod são **o mesmo compose/topologia**, parametrizados por ambiente. O que muda é só: qual commit roda, quanto recurso recebe, e quais instâncias de dados usa. A **estrutura é idêntica** porque é o mesmo artefato imutável promovido, não um build novo.

---

## A CAMADA DE TRADUÇÃO (por que a IA nunca vê o runtime)

        IA chama tool (intenção)

              │

              ▼

   ┌──────────────────────────┐

   │  CONTROL-PLANE           │

   │  \- interpreta a intenção │

   │  \- consulta runtime\_target│

   │  \- traduz pro runtime ativo│

   └──────────┬───────────────┘

              │

    ┌─────────┼─────────┐

    ▼         ▼         ▼

 COMPOSE    SWARM      K8S

 (local)  (intermed.) (escala)

`gen("postgres","db")` vira:

- **Compose**: um bloco `services: db:` no docker-compose.yml  
- **Swarm**: um service no stack Swarm, distribuível  
- **k8s**: um Deployment \+ Service (ou aponta pra Neon, se prod gerenciada)

A IA chamou a mesma coisa nos três casos. Essa indireção é o que te deixa **começar no Compose (local, grátis, MVP) e migrar pra Swarm e depois k8s** sem reescrever o agente nem as tools.

---

## RESUMO DAS MUDANÇAS vs TUA VERSÃO ORIGINAL

| Tua versão | Refinamento | Por quê |
| :---- | :---- | :---- |
| `gen(nome)` | `gen(type, name)` \+ registry \+ env auto | serviços se descobrem sem hardcode |
| `change` | \+ validação de unicidade | não corromper arquivo silenciosamente |
| (faltava) | `status()`, `read()`, `write()` | agente não pode agir cego |
| `publish` \+ `back_version` | `promote(commit, env)` único | commit como fonte de verdade; menos conceito |
| versão E commit | commit é verdade, versão é tag | elimina competição de fonte de verdade |
| (faltava) | `scale()` | onde a escala horizontal entra (Swarm/k8s) |
| compose direto | camada de tradução intenção→runtime | Compose→Swarm→k8s sem reescrever |

---

## DECISÕES EM ABERTO

1. `promote` único vs manter `publish`\+`back_version` separados?  
2. `scale` deve recusar serviço stateful e orientar pro gerenciado?  
3. `internal_url` por nome estável (resolve igual nos 3 runtimes) — confirma?  
4. Precisa de `read`/`write` além de `change`, ou `change` cobre? (acho que precisa)  
5. "Versão" sobrevive como tag amigável, ou commit basta?

