# RESSALVAS & DECISÕES — Plataforma AI App Builder

Arquivo vivo. Registra decisões tomadas, o porquê, e as dívidas/riscos conscientes que aceitamos pra revisitar depois. Não é spec — é memória. Última atualização: revisão inicial.

---

## DECISÃO PRINCIPAL: dois ambientes — Criação (Compose) e Prod (k8s)

**Esclarecimento estrutural:** não são três ambientes. "Local" e "preview ao vivo" são A MESMA COISA — o ambiente de CRIAÇÃO. O preview em tempo real não é um ambiente separado; é uma PROPRIEDADE do ambiente de criação (hot-reload \+ janela mostrando o app rodar). Quem cria — seja o dev local, seja um usuário num servidor self-hosted — usa o mesmo runtime e vê o app nascer em tempo real.

AMBIENTE DE CRIAÇÃO (Compose \+ hot-reload \+ preview ao vivo)

  ├─ dev rodando local e criando com a IA

  ├─ usuário criando na instância self-hosted privada

  └─ mesmo runtime, mesma experiência, mesma mecânica

        │  "publicar" → trigger CI/CD

        ▼

PROD (k8s / Azure)

  └─ app publicado, escalado, estável; stateful → gerenciado Azure

**O que decidimos:**

- **Criação** (local \+ self-host): docker compose \+ hot-reload. A IA gera no Compose, o app recarrega sozinho, o preview reflete em tempo real. Leve, iteração instantânea, idêntico em qualquer máquina.  
- **Prod**: trigger CI/CD materializa em k8s (AKS na Azure).  
- **Stateful**: container no Compose na criação; gerenciado Azure em prod.

**Por quê:**

- A experiência de criação é IDÊNTICA em qualquer lugar (dev local \= self-host do cliente). Reforça o requisito self-host: o que usamos pra dev é o que entregamos.  
- Não paga a complexidade do k8s onde ela não rende. k8s resolve escala horizontal — a criação não precisa disso.  
- Uma única fronteira de runtime (Compose→k8s), num único momento (publicar), escondida na camada de tradução. A IA e o código do app nunca veem.

**O que de fato muda entre criar e produzir (só isso a tradução resolve):**

1. **Modo de execução**: criação \= modo dev (hot-reload, código não-otimizado); prod \= artefato compilado (bundle otimizado, sem hot-reload).  
2. **Destino do stateful**: criação \= container; prod \= gerenciado Azure.  
3. **Recursos/escala**: criação \= 1 container, 1 host; prod \= k8s com réplicas. → A TOPOLOGIA não muda (mesmos serviços, ligações, env vars). Só os parâmetros de execução. É isso que evita "funciona ao criar, quebra ao publicar".

**Custo/risco aceito:**

- ⚠️ Salto modo-dev → build-de-produção é a ÚNICA divergência real. Mitigação: rodar o build de produção de verdade e validar ANTES de promover, nunca publicar direto do modo dev às cegas (ver ressalva \#8).

---

## RESSALVA \#8 — Paridade entre Criação (Compose) e Prod (k8s)

Uma única fronteira de runtime (Compose↔k8s), no momento de publicar. Disciplina pra não reintroduzir "funciona ao criar, quebra ao publicar":

1. **Paridade de topologia.** O Compose de criação e o manifesto k8s de prod devem descrever os MESMOS serviços, ligações e env vars — só mudando recursos e pra onde o stateful aponta. 🔴 Ideal: ambos GERADOS da mesma fonte interna (a descrição do app), não escritos à mão. Assim não divergem por construção.  
     
2. **Diferença stateful entre ambientes.** Criação fala com postgres-container; prod fala com Azure Database for PostgreSQL. Código lê sempre de `process.env.DB_*` — o ambiente troca o valor, o código nunca sabe. 🔴 Vigiar: fixar no container de criação a MESMA versão/extensões de Postgres que a Azure usa, pra não pegar diferença de feature só em prod.  
     
3. **Salto modo-dev → build-de-produção (a divergência real).** Cria-se vendo o modo dev (hot-reload, código tolerante); publica-se o build estrito otimizado. Bugs que só aparecem no build de produção existem. 🔴 Mitigação: no pipeline de publicação, rodar o build de produção DE VERDADE e validar antes de promover — é o passo de scan/verificação, agora com propósito claro.

**Mecânica do preview ao vivo (o "tempo real" da criação):**

- Container de dev sempre-on por sessão de criação, com hot-reload (Vite).  
- A IA edita arquivos via tools → o dev server detecta → recarrega.  
- Preview \= iframe apontando pro container de dev (reflete sozinho) \+ opcionalmente WebSocket de eventos pro editor.  
- ⚠️ Em escala: cada usuário criando \= um container de dev vivo. Frota de containers efêmeros (liga/desliga por sessão). Pro MVP/local é trivial; em escala é orquestração de previews — anotado, não urgente.

---

## RESSALVA \#1 — Serviços stateful (postgres/redis/rabbit)

**A regra que adotamos:** nível de gestão de estado é função de QUEM OPERA.

local / dev (k3s)        → container no cluster (dados descartáveis, simples)

self-host do cliente     → container no cluster DELE (operação é responsa deles)

prod gerenciada (AKS)    → aponta pra gerenciado da Azure, NÃO StatefulSet nosso

                            ├─ postgres → Azure Database for PostgreSQL

                            ├─ redis    → Azure Cache for Redis

                            └─ rabbit   → Azure Service Bus ou container gerenciado

**Dívida consciente:**

- 🔴 `gen("postgres")` terá DOIS modos de materialização (container local vs gerenciado Azure em prod). A camada de tradução precisa saber fazer os dois. Isso é uma costura — aceita, mas anotada.  
- 🔴 Stateful em k8s (StatefulSet, volumes persistentes, storage distribuído) é a parte mais difícil de operar. NÃO operar isso nós mesmos em prod — apontar pra gerenciado. Se um dia formos operar, é projeto dedicado, não casual.  
- ⚠️ Backup/restore por app: trivial com gerenciado (Azure faz). Com container local, é responsabilidade documentada de quem hospeda.

**Princípio guardado:** código (stateless) escala multiplicando cópias — fácil. Estado (stateful) escala com operação especializada — empurrar pra quem opera melhor conforme o risco vira nosso.

---

## RESSALVA \#2 — Custo é o gargalo inicial, não a arquitetura

**Situação:** orçamento zero agora. Não vamos custear infra de prod.

**Implicações registradas:**

- Tudo precisa rodar local de graça (k3s \+ containers) → ✅ a arquitetura permite.  
- Serviços gerenciados Azure (AKS, Postgres, Redis) só entram quando o cliente pagante aparecer e custear na conta DELE (modelo BYOK \+ self-host).  
- 🔴 NÃO introduzir nenhuma dependência paga obrigatória no núcleo. Tudo que é pago deve ser opcional/substituível por equivalente local open-source.  
- A demonstração/MVP roda 100% local. A escala real é problema (bom) do futuro.

---

## RESSALVA \#3 — Sandbox de build (isolamento de código de IA)

**Dívida consciente do MVP:**

- 🔴 No MVP, o build de código de IA roda em container comum (NÃO é isolamento forte contra código hostil). Aceitável só porque o código é gerado via nossos prompts, não por atacante.  
- Em prod/self-host: ativar gVisor (roda em Linux real) ou Firecracker.  
- A etapa de build fica atrás de interface trocável → trocar o motor não reescreve o resto.  
- ⚠️ NUNCA esquecer que isso é dívida. Antes de aceitar código não-confiável de terceiros (ex: cliente edita o prompt livremente), o sandbox forte é pré-requisito, não opcional.

---

## RESSALVA \#4 — Database-per-app tem teto

- Modelo: cada app \= um database (isolamento real, mata RLS multi-app).  
- 🔴 Teto: um Postgres aguenta bem centenas a poucos milhares de databases. Pro MVP sobra. Em escala grande, migrar pra instâncias por tier ou Postgres gerenciado com branching (Neon-like / Azure flexible server).  
- Anotado pra revisitar quando o número de apps crescer.

---

## RESSALVA \#5 — IA e chaves (BYOK via LiteLLM \+ Infisical)

- Modelo: cliente traz a própria API key (BYOK). Custo de IA não passa por nós.  
- LiteLLM (self-hosted) unifica provedores e captura metering de tokens.  
- Chaves no Infisical, cifradas por tenant, nunca no browser nem no banco do app.  
- ⚠️ MVP pode começar com `.env` protegido e subir pro Infisical depois (mesma interface). Decidir quando.  
- 🔴 Decisão em aberto: BYOK puro vs híbrido (key nossa com teto pro free). Pra orçamento zero, BYOK puro no início.

---

## RESSALVA \#6 — As tools expressam intenção, não runtime

- A IA chama `gen`, `change`, `promote`, `scale`, `status` — nunca `kubectl`.  
- Camada de tradução (control-plane) converte intenção → manifesto k8s.  
- Mantém o agente estável mesmo se o runtime mudar no futuro.  
- 🔴 Decisões de tool em aberto:  
  1. `promote(commit, env)` único vs `publish`\+`back_version` separados.  
  2. `scale` recusa stateful e orienta pro gerenciado?  
  3. Precisa `read`/`write` além de `change`? (provavelmente sim)  
  4. "Versão" sobrevive como tag amigável ou commit basta como fonte de verdade?  
- Peça que faltava na ideia original: **service registry** (serviços se descobrem, connection string flui sem hardcode, diferente em dev/prod).

---

## RESSALVA \#7 — Self-host como requisito de design

Três disciplinas a manter desde o início (custam pouco agora, caro depois):

1. Tudo declarado em manifesto k8s (+ Helm chart pra empacotar).  
2. Zero dependência hard de serviço proprietário — tudo atrás de interface trocável ("um Postgres", não "o Azure Postgres").  
3. Config 100% por ambiente — nada hardcoded; cliente aponta pros recursos dele.

**Objetivo:** empresa roda a instância dela em AKS dentro do compliance dela, com a key de IA dela, sem nada nosso saindo da infra dela.

---

## MAPA DE EQUIVALÊNCIA Criação ↔ Prod Azure (pra não esquecer)

COMPONENTE        CRIAÇÃO (local+self-host, Compose)  PROD AZURE (k8s, cliente custeia)

──────────────────────────────────────────────────────────────────────────────

Runtime           docker compose \+ hot-reload         AKS (k8s), via trigger CI/CD

Modo de execução  dev (hot-reload, preview ao vivo)   build compilado otimizado

Stateless apps    serviço no compose                  pods/deployments no AKS

Postgres          container                           Azure Database for PostgreSQL

Redis             container                           Azure Cache for Redis

RabbitMQ          container                           Azure Service Bus / container

Borda/ingress     Traefik                             Traefik ou Azure App Gateway

Secrets           .env → Infisical                    Infisical self-host / Key Vault

LLM gateway       LiteLLM container                   LiteLLM no AKS

Observabilidade   Grafana/Prom/Loki                   mesmo stack ou Azure Monitor

Build sandbox     container (dívida)                  gVisor/Firecracker

Transição                  └──────── CI/CD traduz Compose→k8s ────────┘

                                  (uma vez, no publicar)

---

## LISTA RÁPIDA DE DÍVIDAS (🔴 \= revisitar antes de produção séria)

- [ ] 🔴 Gerar Compose (dev/hml) e k8s (prod) da MESMA fonte, pra não divergirem  
- [ ] 🔴 Fixar mesma versão/extensões de Postgres no container e na Azure  
- [ ] 🔴 Sandbox de build forte antes de aceitar código não-confiável de terceiros  
- [ ] 🔴 Materialização dual de stateful (container local/hml vs gerenciado Azure)  
- [ ] 🔴 Teto de database-per-app — plano de migração quando crescer  
- [ ] 🔴 Não operar StatefulSet de banco em prod — sempre gerenciado  
- [ ] 🔴 Decidir BYOK puro vs híbrido  
- [ ] 🔴 Fechar assinaturas finais das tools (promote, scale, read/write)  
- [ ] ⚠️ Salto modo-dev → build-de-prod: validar build real antes de promover  
- [ ] ⚠️ Em escala: orquestrar frota de containers de preview por sessão  
- [ ] ⚠️ Curva k8s — concentrada só em prod agora; orçar tempo de aprendizado  
- [ ] ⚠️ Migrar .env → Infisical em algum momento  
- [ ] ⚠️ Nenhuma dependência paga obrigatória no núcleo

