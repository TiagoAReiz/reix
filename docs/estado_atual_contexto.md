# Estado Atual & Contexto Não-Encaixado

> Documento-ponte pra retomar o projeto. Assume que você (Claude, noutro
> contexto) JÁ TEM acesso aos outros documentos gerados. Este NÃO repete o que
> já está neles — traz só o que ficou solto, o ponto exato onde paramos, e as
> decisões recentes que ainda não entraram em spec formal.

## Documentos existentes (contexto completo do projeto)
1. **arquitetura_consolidada.md** — visão geral: duas camadas, control-plane,
   apps gerados, tools, ambientes, fluxos.
2. **modelo_dominio_controlplane.md** — stack, conexões, assincronismo,
   estrutura do repo, entidades, manifesto.
3. **tools_refinadas.md** — assinaturas das tools (gen, change, promote, etc).
4. **viabilidade_economica.md** — custos, sleep, fila de build, quotas.
5. **ressalvas_decisoes.md** — decisões e dívidas técnicas (🔴/⚠️).

Este documento é o 6º e cobre as lacunas.

---

## 1. O QUE É O PROJETO (resumo de 30 segundos)

Plataforma que gera apps completos via IA, fugindo do molde fixo do Lovable. A IA
COMPÕE a arquitetura (múltiplos serviços de um catálogo curado) via um loop
agêntico com tools. Cada tool faz request HTTP pro agent runtime dentro do
sandbox de criação, que materializa (cria serviço, edita código) com preview ao
vivo. Publicar → master do repo → CI/CD → containers k8s. Isolamento por app
(repo, banco, container próprios). BYOK pra IA. Self-host possível. Roda local em
Compose (grátis), prod em k8s/Azure. Operável solo comprando robustez.

---

## 2. FLUXO FINAL VALIDADO (a recapitulação que fechou)

```
CRIAÇÃO (dev, local, preview ao vivo):
  control-plane → loop agêntico com IA → tool → request HTTP →
  AGENT RUNTIME no sandbox → cria serviço / edita código (com travas) →
  hot-reload → preview atualiza

PUBLICAÇÃO:
  publish → merge na master do repo do app → CI/CD → gera containers k8s
```

### ⚠️ Ajuste importante que foi feito na recapitulação
A API de tools (agent runtime) roda no **SANDBOX DE CRIAÇÃO**, NÃO no app em
prod. É andaime de construção:
```
CRIAÇÃO: container = [app em modo dev + hot-reload] + [agent runtime (as mãos)]
PROD:    container = [app compilado rodando]  ← SEM agent runtime
```
O app em prod NÃO tem API que aceita "edite seus próprios arquivos" — seria
brecha de segurança. O agent runtime existe enquanto constrói, some quando publica.

---

## 3. DECISÕES RECENTES AINDA NÃO EM SPEC FORMAL

### 3.1 Onde ficam os repositórios

Depende do modo de deployment, resolvido pelo adapter de Git:
```
SaaS (nós hospedamos)  → repos privados na NOSSA org GitHub, um por app.
                         Cliente não conecta GitHub nem sabe que há Git embaixo.
self-host (compliance) → repos na org do cliente OU Gitea na instância dele.
                         Código fica no controle do cliente.
```

**Decidido pro MVP:** nossa própria org GitHub, repo privado por app. Razões:
mais simples solo, API GitHub excelente, repo privado grátis/ilimitado, cliente
nem precisa de conta. O adapter mantém self-host possível sem decidir agora.

**Importante:** no SaaS, o tenant NÃO conecta "o GitHub da empresa dele" — os
repos são nossos, ele nem toca em Git. Conexão do GitHub do cliente só existe no
self-host ou como feature secundária "exportar pro meu GitHub".

### 3.2 Geração inicial de um app (provisionamento em 5 passos)

Prepara um esqueleto vazio MAS FUNCIONAL (scaffold) antes de a IA entrar. A IA
não parte do nada — parte de um app mínimo que já roda.

```
usuário clica "criar app" (ex: "loja-acme")
  │
  1. CRIA ENTIDADE App no banco
     → id (UUID), slug, tenant_id, created_by
     → lifecycle=criando, runtime=sem-deploy
     → ainda sem repo, sem serviços
  │
  2. CRIA REPO (adapter Git → GitHub API)
     → repo privado na nossa org
     → inicializa com SCAFFOLD BASE (estrutura, app mínimo que roda,
       app.manifest.json inicial, .gitignore, README, config)
     → cria branches dev + master
     → guarda repo_url no App
  │
  3. REGISTRA no tenant catalog
     → app_id → repo_url, tier, dono; espelha manifest inicial no banco
  │
  4. PREPARA SANDBOX (ambiente de criação)
     → sobe container, clona o scaffold, instala deps, sobe dev server +
       hot-reload, inicia o agent runtime
     → guarda sandbox_ref no App; cria a Session
  │
  5. PRONTO — preview mostra o scaffold rodando; IA pode receber 1º prompt
```

**Ordem importa (segurança/consistência):** entidade primeiro (checkpoint mesmo
se algo falhar), depois repo, depois sandbox. Cada passo é um checkpoint;
recursos órfãos evitados.

**Decisão em aberto — o SCAFFOLD:** template genérico bem montado que serve de
base pra qualquer app, OU a IA compõe o scaffold conforme o tipo de app que o
usuário descreveu? Conecta com como o 1º prompt é tratado. NÃO decidido ainda.

**Decisão em aberto — QUANDO subir o sandbox:** na criação (responsivo) vs sob
demanda/lazy no 1º prompt (economiza sandbox de app criado e abandonado). Pro
MVP, na criação é mais simples; em escala, lazy casa com sleep.

### 3.3 Sequenciamento de autenticação (decisão recente e importante)

**Decidido: começar SEM auth, rodando local, com um Tenant e User próprios fixos.
Autenticar e proteger DEPOIS.**

Racional: auth não valida a hipótese central (fazer a IA compor apps via tools).
É ortogonal ao motor. Adiar acelera a iteração no núcleo.

**A condição que torna isso barato (INEGOCIÁVEL):**
Construir multi-tenant DESDE JÁ, mas com um único tenant e sem porta de entrada.
- `tenant_id`/`user_id` já permeiam TODO o modelo e TODAS as operações (tools,
  executor, queries) desde o dia 1 — como se viessem de um login.
- A identidade vem de um PONTO ÚNICO trocável: uma função tipo
  `get_current_context()` que HOJE retorna o tenant/user fixo e AMANHÃ lê o JWT.
```
COM auth (depois): request → valida JWT → extrai tenant_id/user_id → contexto
SEM auth (agora):  request → tenant_id/user_id FIXO → contexto
                   └─ MESMO contexto, origem diferente. Trocar auth = 1 ponto.
```
- Erro a evitar: fazer tools/queries ASSUMIREM tenant único (sem passar
  tenant_id explícito). Isso vira reescrita quando auth chegar.

**Gate explícito:** 🔴 antes de expor PUBLICAMENTE, auth é pré-requisito, não
opcional. Adiar vale só enquanto o acesso é controlado por estar local/privado.

**Nota:** autenticação ("quem é você") ≠ autorização ("pode fazer isso com este
app?"). A autorização é a que protege tenant A de tocar no app do tenant B. Com
tenant único agora é trivial. Quando auth chegar, se tenant_id já permeia tudo, a
autorização vira "toda query filtra por tenant_id do contexto" — groundwork já feito.

---

## 4. O PONTO EXATO ONDE PARAMOS

O núcleo do domínio está fechado. As entidades (Tenant, User, App, Service,
Session, Deployment), o manifesto (duplo armazenamento), as conexões (agent
runtime na criação, Git/CI-CD na prod), o assincronismo (background + eventos), a
stack (Python/FastAPI), a estrutura do repo, e o sequenciamento de auth — tudo
decidido.

### Próximos passos possíveis (nenhum iniciado)
```
🔴 ACESSO SEGURO (auth/autorização) — reservado, mas com sequenciamento já
   definido (adiar, groundwork multi-tenant pronto). Detalhar quando for expor.

🔴 ASSINATURAS FINAIS DAS TOOLS — fechar o promote unificado, schemas exatos,
   o contrato JSON agente↔executor (formato de tool call e de resultado).
   É o que destrava escrever a primeira tool de verdade.

🔴 SCAFFOLD — decidir template genérico vs composto por IA. Conecta com o
   tratamento do 1º prompt.

⚠️ QUOTAS — números reais por tier.

PRÁTICO (sair do papel):
   - docker-compose de criação com control-plane + 1 node + 1 postgres
   - esqueleto do control-plane (estrutura de pastas da seção 4 do
     modelo_dominio) com a tool `gen` e o service registry funcionando
   - o agent runtime mínimo (servidor HTTP que edita/lê/roda no container)
```

### Recomendação de ordem pra sair do papel
1. Fechar o contrato agente↔executor (formato de tool call/resultado) — pequeno
   e destrava tudo.
2. Montar o docker-compose de criação + esqueleto control-plane Python.
3. Implementar `gen` + service registry (o núcleo de onde a geração dual
   Compose/k8s deriva) com o tenant/user fixo.
4. Implementar o agent runtime mínimo e conectar 1 tool ponta a ponta.
5. Só então: scaffold, mais tools, e por último auth (antes de expor).

---

## 5. PRINCÍPIOS QUE NÃO PODEM SER ESQUECIDOS (o "porquê" de tudo)

- **Comprar robustez, construir só o diferencial** (operável solo).
- **Tools expressam intenção, não comandos de runtime** (IA não sabe se é
  Compose ou k8s; adapter traduz).
- **Referência ≠ segredo** (entidades guardam nomes; credenciais no secret
  manager, resolvidas em runtime).
- **O repo sempre manda** (manifesto: Git = verdade, banco = espelho).
- **Isolamento por app** (repo, banco, container próprios).
- **BYOK** (custo de IA não passa por nós).
- **Self-host como requisito de design** (tudo atrás de interface trocável,
  config por ambiente, nada proprietário hard no núcleo).
- **Sleep em tudo que tem estado/compute** (viabilidade econômica).
- **Multi-tenant desde já, auth depois num ponto único** (sequenciamento).
