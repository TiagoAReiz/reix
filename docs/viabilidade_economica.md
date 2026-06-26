# Viabilidade Econômica & Estratégia de Custo

> O eixo que separa arquitetura bonita de negócio que sobrevive. Consolida a
> análise de viabilidade, os centros de custo, e as estratégias pra cada um.
> Companion: arquitetura_consolidada.md, ressalvas_decisoes.md, tools_refinadas.md

---

## 1. O PROBLEMA ECONÔMICO CENTRAL

**Densidade vs Isolamento.** Tudo que ganhamos em isolamento (container por app,
banco por app, sandbox por sessão) tem custo de OCIOSIDADE. Uma coisa que existe
mas não atende ninguém ainda ocupa recurso e custa. O custo cresce com o número
de coisas que EXISTEM, não com as que são USADAS.

**Por que isso importa:** é provavelmente o que empurrou Lovable e Google AI
Studio pras escolhas simplificadas deles.

```
Google AI Studio → funções tipo Lambda. Scale-to-zero NATIVO, custo ocioso ZERO.
                   Preço: cold start, sem estado, sem WebSocket, limite execução.

Lovable          → app static (CDN, custo ~zero) + Supabase compartilhado.
                   Sem infra dedicada por app. Preço: arquitetura travada.

NÓS              → infra RICA por app (o que ambos evitaram, por ser caro ocioso).
                   Ganhamos poder (apps complexos, escaláveis); pagamos custo unitário.
                   É um produto PREMIUM diferente, não o mesmo mercado do Lovable.
```

**Conclusão de posicionamento:** a arquitetura é viável, mas NÃO é barata por
design como a do Lovable. Isso é escolha de posicionamento (premium / apps
reais), não defeito. A viabilidade depende de disciplina de custo inegociável.

---

## 2. OS QUATRO MAIORES CENTROS DE CUSTO (e a estratégia de cada um)

### 2.1 Chamadas de IA → RESOLVIDO por BYOK ✅

O custo que mais assusta numa plataforma de IA (tokens) NÃO passa por nós. O
cliente traz a própria key, paga o próprio consumo direto ao provedor. Maior
acerto de viabilidade do design. **Manter BYOK custe o que custar.**

### 2.2 Build → RESOLVIDO por FILA + GitHub Actions ✅

Build é pontual, intenso e TOLERA espera (assíncrono por natureza). Isso permite
fila.

```
usuário publica → promote → ENFILEIRA job (prioridade do tier)
   → worker pega quando há slot → GitHub Actions roda (infra do GitHub, não nossa)
   → build + validação → artefato → deploy k8s → notifica
```

**Por que funciona:**
- Fila ABSORVE picos sem provisionar pra pico (amortecedor demanda↔capacidade).
- GitHub Actions = builder terceirizado, scale-to-zero (runner provisiona no job,
  descarta no fim). Custo de build não é fixo nosso.
- Latência de fila vira ALAVANCA DE TIER (não problema):

```
FREE       → fila normal, baixa prioridade (espera mais — é grátis)
PAGO       → fila prioritária, mais paralelismo (fura a fila)
ENTERPRISE → fila dedicada / capacidade reservada (nunca espera)
```
- Feedback honesto da fila ("posição 12, ~2min") torna a espera tolerável.

**Trade-off gerenciável:** latência-de-fila vs custo-de-paralelismo. Muito melhor
que container-24/7 porque o paralelismo de build escala sob demanda e volta a zero.

**Onde morde (🔴 anotado):**
- GitHub Actions tem teto de concorrência. Em volume, deixa de ser grátis →
  migra pra self-hosted runners ou builder próprio. Padrão "comprar até doer".
- Build nunca é instantâneo (provisionar runner leva segundos), mesmo com fila vazia.

### 2.3 Sandbox / Preview (criação) → o caso MAIS DIFÍCIL

É o maior centro de custo POTENCIAL. Cada usuário criando = um container de dev
acordado com hot-reload. 100 pessoas que abriram o editor e foram tomar café =
100 containers ociosos.

**Característica chave:** stateful DURANTE a sessão, descartável ENTRE sessões
(o estado reconstrói do repo, porque o código está commitado).

**Estratégia: NÍVEIS DE SONO (não um sleep só):**

```
SESSÃO ATIVA        → container rodando, custo real
COCHILO (minutos)   → hibernado (preserva estado, volta rápido, custo baixo)
SONO PROFUNDO (h+)  → destruído (custo ZERO, reconstrói do repo ao voltar)
ABANDONADO (dias)   → garbage collected (sem rastro além do repo)
```

Gatilho: heartbeat de atividade + timeout no control-plane decide quando rebaixar
o nível. Barato de implementar, e é o que separa "pago por sessão ativa" de
"pago por todo mundo que já tocou no produto".

**Compute no Azure — VM é a escolha ERRADA aqui:**

| Opção | Sleep | Veredito |
|-------|-------|----------|
| VM | lento (minutos), disco cobra parado | ❌ pior pra sandbox |
| Azure Container Instances (ACI) | por segundo, para = custo zero | ✅ container avulso sob demanda |
| Azure Container Apps (serverless) | scale-to-zero nativo, sem operar k8s | ✅ ideal pra dormir/acordar |
| AKS + KEDA (scale-to-zero) | aproveita cluster de prod | ✅ se já tem o cluster |

→ Tirar VM da mesa pro sandbox, EXCETO se precisar isolamento microVM por
segurança (aí é gVisor/Firecracker — dimensão de segurança, não custo).

### 2.4 Apps em prod → RESOLVIDO por SCALE-TO-ZERO

App de ecommerce que vende de dia e dorme de noite não pode ter container 24/7.

```
com tráfego  → container rodando
sem tráfego  → dorme (scale-to-zero via KEDA / Knative / Container Apps)
volta tráfego→ acorda no 1º request (cold start — mas só pra app ocioso,
               onde não dói porque ninguém está usando)
```

Reintroduz o cold start que tentamos evitar — mas só onde ele NÃO importa.

---

## 3. OS CUSTOS SILENCIOSOS (não dormem)

### 3.1 Banco por app

Scale-to-zero de banco é mais difícil (estado + conexão). Postgres parado ainda
ocupa. 🔴 É provavelmente o item mais caro com muitos apps ociosos.
- Saída: tiers serverless do Azure SQL/Postgres que PAUSAM sob inatividade
  (equivalente ao que o Neon faz com branching + scale-to-zero).
- Sem pausa de banco, banco-por-app é o maior custo latente da arquitetura.

### 3.2 Storage acumulado

Diferente de compute, storage NÃO dorme — só cresce. Bancos, repos, artefatos de
build, imagens. Cada app deixa rastro permanente. 🔴 Storage de milhares de apps
mortos é custo silencioso que scale-to-zero nenhum resolve.
- Saída: garbage collection (seção 4).

### 3.3 Control-plane (o único 24/7 verdadeiro)

Não pode dormir — é quem atende requisições e acorda os outros. MAS: é UM serviço,
leve, COMPARTILHADO entre todos os tenants. Custo pequeno, fixo, NÃO cresce com o
número de apps. É o mínimo aceitável. ✅

---

## 4. CONTROLE DE LIMITES = DEFESA DE CUSTO (não feature)

Reposicionamento crítico: limite de criação não é só anti-abuso — é o mecanismo
de SOBREVIVÊNCIA econômica. Cada app criado é custo latente (storage, repo, slot
no catalog) mesmo dormindo.

- **Quota de apps por tier** = defesa de custo direta. O limite reflete quanto
  custo ocioso aceitamos bancar por tier. Free 1-2 apps; pago mais; etc.
- **Garbage collection de apps mortos**: criados e abandonados (nunca publicados,
  sem atividade há meses) → hibernação profunda ou remoção. Senão acumula custo
  latente eterno de quem testou uma vez.
- **Sleep agressivo no free, relaxado no pago**: o tier define quão rápido dorme.
  Free dorme em minutos; enterprise fica acordado mais. Sleep vira alavanca de
  produto e de pricing.

---

## 5. VIABILIDADE POR FASE (resposta honesta em camadas)

```
MVP / DEMO (local, orçamento zero)
  → 100% viável, custo zero. Roda local, poucos apps, nada ocioso em escala.
  → Perigo: não confundir "funciona local" com "é barato em produção".

ESCALA PEQUENA (primeiros pagantes)
  → Viável SE sleep em tudo (preview, prod, banco) desde a virada pra prod.
  → BYOK + self-host: boa parte do custo é do cliente, não nossa.
  → Sem sleep, poucos clientes ociosos já deixam a conta feia.

ESCALA GRANDE (competir com Lovable de igual)
  → Economia aperta. Arquitetura rica por app custa mais que a compartilhada deles.
  → Precisaria de capital OU margem alta por cliente.
  → É produto PREMIUM (apps reais, não protótipos) — mercado diferente, preço
    diferente. Não é o mesmo jogo do Lovable.
```

---

## 6. AS DISCIPLINAS INEGOCIÁVEIS (o que faz a conta fechar)

```
1. SLEEP em tudo que tem estado ou compute
   └─ preview (níveis de sono) · prod (scale-to-zero) · banco (pausa serverless)

2. BYOK pra tirar o custo de IA de nós  ✅ (já no design)

3. QUOTAS + GARBAGE COLLECTION como defesa de custo (não feature)

4. FILA pro build, terceirizada no Actions, modulada por tier  ✅
```

Com as quatro, a conta fecha pro modelo self-host/premium. Sem elas, um punhado
de clientes ociosos afunda a margem.

---

## 7. ESTRUTURA DE CUSTO RESULTANTE (saudável)

```
CUSTO VARIÁVEL (proporcional a uso real):
  ├─ sandbox    → tempo de criação ativa (níveis de sono + container serverless)
  ├─ build      → nº de publicações (fila + Actions terceirizado)
  ├─ prod       → tráfego real (scale-to-zero)
  └─ IA         → ZERO pra nós (BYOK, cliente paga)

CUSTO FIXO (pequeno, contido):
  ├─ control-plane → um serviço leve, compartilhado, 24/7 (mínimo aceitável)
  └─ storage       → contido por garbage collection

→ Custo cresce com USO REAL, não com cadastros. Estrutura saudável pra produto
  premium. Os quatro maiores buracos (IA, build, sandbox, prod) ficam tampados.
```

---

## 8. DÍVIDAS/PONTOS DE CUSTO A REVISITAR

- [ ] 🔴 Banco por app: ativar pausa serverless (Azure) antes de escala — maior custo latente
- [ ] 🔴 Storage: implementar garbage collection de apps mortos
- [ ] 🔴 GitHub Actions: teto de concorrência — plano pra self-hosted runners em volume
- [ ] 🔴 Quotas por tier: definir números reais (quantos apps por tier)
- [ ] ⚠️ Sandbox: escolher entre ACI / Container Apps / AKS+KEDA (testar custo real)
- [ ] ⚠️ Níveis de sono: definir timeouts (cochilo → sono profundo → garbage)
- [ ] ⚠️ Detecção de inatividade: heartbeat + timeout no control-plane
- [ ] ⚠️ Cold start aceitável em prod ocioso — validar UX do "acordar"
```
