# RFC: Harvest Design-Extraction (colher análise longa + ToolResults ricos)

**Data:** 2026-09-26 (rev. 2026-09-27)
**Autor:** Claudio + Zero (Kiro CLI)
**Branch de código:** `feat/harvest-design-extraction` (a partir de `upstream/main`)
**Base:** `upstream/main` v11.14.0
**Versão:** 0.2 (draft — pronto para abrir como issue própria, a pedido do Henry no #1287)
**Inspiração:** diagnóstico #1100 (13/set) + discussion #1287 (beacon loop, direcionamento do Henry) + #1103 (LLM summarization, VijaySreekar — ponta oposta)
**Reintegra:** o GAP DE PRODUTO deixado aberto pela RFC-harvest-provenance (que resolveu proveniência+tracker, NÃO cobertura de conteúdo rico)
**Status:** DRAFT v0.2 — a abrir como issue própria (Henry: "write the RFC as you planned... keep it on its own thread rather than folding it into #1286").

---

## 1. Problema

**"Sessões riquíssimas, colheita pobre."** O harvest atual é estruturalmente incapaz de colher o conteúdo mais valioso das sessões: **análise de design/arquitetura longa** e **ToolResults ricos**. O prato principal é jogado fora; sobram fragmentos.

### Causa raiz (estrutural, não bug pontual)

Duas limitações compostas no pipeline de harvest:

1. **Extractor prioriza frases-gatilho curtas.** O extractor casa padrões como "decidi", "bug era", "causa raiz", "confidence gate 0.6". Análise de design extensa — o formato de trabalho de RFC/arquitetura — **não casa esses padrões e é descartada**. O raciocínio ("*por que* decidimos X") se perde; na melhor hipótese sobra a conclusão seca ("decidiu X").

2. **Parser ignora ToolResults por completo.** `harvest/parser.py:32` — `KIRO_KIND_MAP = {"Prompt": "user", "Response": "assistant", "AssistantMessage": "assistant"}`. A linha 137 (`role = self.KIRO_KIND_MAP.get(kind)`) retorna `None` para `ToolResult` → descartado. Todo dado analítico que embasou decisões (queries de banco, saídas de investigação, diffs, contagens) nunca entra no harvest.

### Evidência decisiva (caso #1100, memória-âncora `241248324`)

As sessões do design do #1100 (agent_id) continham **23 análises de design longas (>400 chars)** em AssistantMessage + **72 linhas de dados analíticos** em ToolResults. O harvest capturou **"1 memória"**. Exemplo real perdido — insight de design puro, não frase-gatilho:

> "A grande maioria não tem identificação do agente. Dos 18.067 ativos: 2.679 têm agent no metadata (~15%)..."

O conteúdo **não se perdeu do disco** (está nas sessões em `ai-backup/kiro-sessions-cli/`), mas **nunca virou memória navegável**.

### O que este RFC NÃO é (desfazendo a contradição do piloto R10)

O piloto R10 (memória `9f490378`, 13/set) mediu que das ~6.929 memórias **já colhidas**, só ~302 eram cruas → **recolheita em massa do que já está no banco = desperdício**. Correto, e **não conflita com este RFC**. São perguntas diferentes:

| Universo | Veredito |
|----------|----------|
| Já colhido (6.929 mems) | Não re-processar em massa — o que virou memória presta (R10) |
| **Nunca colhido (design longo + ToolResults nas sessões)** | **Gap real e aberto — é o que este RFC ataca** |

O argumento "o miner só acha sobras" (jimy-r, #1287) vale para agentes que checkpoint-am bem no momento. O caso #1100 **refuta isso para o nosso uso**: o design substantivo NÃO foi escrito no momento — viveu nas análises longas e ToolResults, e foi descartado por limitação estrutural. Não era sobra; era o prato principal.

### COVERAGE ≠ YIELD (a distinção do Henry, #1287 — o eixo desta RFC)

Henry cravou a distinção que reordena a discussão: nosso "23 blocos → 1 memória" **não é um número de yield, é um número de cobertura**, e os dois estavam sendo lidos como a mesma coisa. Yield-quase-zero pode significar **(a)** "a auditoria não achou nada que valesse guardar" OU **(b)** "o extractor não tinha representação para o que estava lá". O <5% de trigger-rate (nosso) e o 0/18 do jimy-r são do tipo **(a)** — yield real baixo. O caso #1100 é do tipo **(b)** — gap de capacidade. **Não pertencem à mesma coluna.** Uma capacidade que descarta conteúdo por não saber representá-lo reporta um gap como auditoria vazia — e é desligada pelo motivo errado.

Consequência de design (Henry): **a cobertura tem que ser mensurável ANTES de mudar o extractor.** Hoje "23 blocos eram descartáveis" e "23 blocos foram descartados" são indistinguíveis de fora. Daí a §Phase 0 abaixo ser pré-requisito, não parte do extractor.

### Risco de não fazer

Toda sessão de design denso (como a própria sessão de 26/set: análise do auto-supersede, investigação do `_store_associations_in_graph_table`, raciocínio da Opção 1 no #1318) perde o *porquê* das decisões. O banco acumula conclusões sem fundamentação → o agente futuro repete a investigação do zero.

---

## 2. Objetivo

Adicionar um **modo de extração de design** ao harvest que capture análise longa e (opcionalmente) ToolResults ricos, produzindo memórias densas e navegáveis — sem inundar o banco de ruído.

**Não-objetivos:** recolheita em massa do corpus já colhido (R10 refutou); substituir o extractor de frase-gatilho (coexistem); mineração de transcript cru em busca de "sobras" (#1287 mostrou yield baixo).

---

## 3. Requisitos (Prosa + EARS)

> Convenção EARS (DEVELOPMENT-STANDARDS §8.4.1): uma ação por frase, sujeito = componente, testável.

### Phase 0 — instrumento de cobertura (PRÉ-REQUISITO, gate do Henry)

O extractor só é construído **depois** deste instrumento provar que há cobertura a recuperar. É uma mudança menor que o extractor e é o que torna qualquer resultado posterior (positivo ou negativo) confiável.

- **R0.1** — WHEN o parser processa um transcript, THE parser SHALL contar o que **viu e descartou por tipo de bloco** (Prompt/Response/AssistantMessage/ToolResult/outros), emitindo um relatório de cobertura sem alterar o que é colhido.
- **R0.2** — THE relatório de cobertura SHALL distinguir "bloco visto e extraído" de "bloco visto e descartado", por tipo, de modo que "N blocos eram descartáveis" e "N blocos foram descartados" deixem de ser indistinguíveis de fora.
- **R0.3** — THE design-extractor (R2/R3) SHALL ser gated no que o instrumento de cobertura reportar: não se implementa o extractor antes de o instrumento mostrar volume de descarte relevante por tipo.

> Racional (Henry): "an instrument that counts what the parser saw and discarded, per block type, is a smaller change than the LLM extractor and it is the thing that tells us whether the extractor was worth building." O jimy-r reforçou com dado próprio: um per-type count teria mostrado a estreiteza do detector (126 candidatos de um só detector) semanas antes da taxa de drain revelar.

### Funcional — captura de conteúdo rico

- **R1** — THE parser SHALL incluir `ToolResult` no mapeamento de tipos, preservando o conteúdo (com truncagem configurável para saídas volumosas).
- **R2** — WHERE um bloco de AssistantMessage excede um limiar de comprimento (`MCP_HARVEST_DESIGN_MIN_CHARS`, default a definir), THE extractor SHALL tratá-lo como candidato a design-extraction em vez de descartá-lo por não casar frase-gatilho.
- **R3** — THE design-extractor SHALL usar o LLM (cadeia de provider existente) com um prompt específico para extrair decisões de arquitetura, trade-offs e o *porquê* — não apenas a conclusão.
- **R4** — THE design-extractor SHALL preservar proveniência (RFC-harvest-provenance): `harvest:method:llm` + `harvest_model`, mais uma marca de modo (`harvest:mode:design`).

### Funcional — controle de ruído (lição do R10 + #1287)

- **R5** — THE design-extraction SHALL ser opt-in por configuração (`MCP_HARVEST_DESIGN_ENABLED`), default off até validação de yield.
- **R6** — THE design-extraction SHALL registrar uma taxa de adoção desde o dia um (candidatos gerados vs. memórias que sobreviveram ao dedup), para distinguir "yield real" de "gate morto" (lição #1287: yield tem que ser contado).
- **R6.1** — THE dashboard/telemetria SHALL distinguir TRÊS estados, não dois (Henry): (1) "rodou e não achou nada"; (2) "bateu o kill-threshold e se desligou"; (3) **"rodou, o conteúdo estava presente, e nada no pipeline conseguiu representá-lo"** — este terceiro é um gap de capacidade (um bug que o loop não consegue reportar sobre si mesmo), não uma auditoria vazia, e sem distingui-lo o loop é desligado pelo motivo errado.
- **R7** — WHERE um candidato de design é ≥ `similarity_threshold` de uma memória existente, THE ingest SHALL evoluí-la (versioned) em vez de duplicar.

### Alternativas mais baratas que o LLM (Henry: "one of them may be enough")

- **R2.1** — ANTES de comprometer um passo LLM, THE RFC SHALL avaliar duas alternativas mais baratas que o instrumento de cobertura (Phase 0) permite medir: **(a)** um trigger-set melhor/mais amplo que capture início de análise longa; **(b)** simplesmente parsear ToolResults (R1) e mantê-los com truncagem. Uma das duas pode fechar a maior parte do gap sem custo/latência de LLM — o LLM-extractor (R3) só se justifica se o Phase 0 mostrar que o gap sobrevive a (a)+(b).

### Não-funcional

- **R8** — THE modo design SHALL rodar preferencialmente server-side no scheduler sobre sinais implícitos (convergência do #1287: capacidade que depende do agente lembrar de chamar não roda), não como ação manual.
- **R9** — THE truncagem de ToolResult SHALL ter teto configurável para não estourar o contexto do LLM nem o custo.

---

## 4. Experimento exploratório necessário ANTES de implementar

Espelhando a disciplina do R10 (pilotar + medir antes de rodar em massa):

1. Selecionar 5-10 sessões de design denso conhecidas (ex.: #1100, #1318, auto-supersede 26/set).
2. Rodar o design-extractor protótipo (prompt LLM "extraia decisões de arquitetura + porquê + trade-offs") contra elas.
3. **Medir:** quantas memórias densas prestáveis por sessão vs. ruído. Comparar com o que o harvest atual capturou dessas mesmas sessões.
4. **Gate de decisão:** só vale implementar em produção se o design-extractor recuperar substancialmente mais conhecimento navegável que o extractor atual, sem inflar ruído. Se o yield for baixo (como o miner do #1287), reavaliar.

---

## 5. Relação com outros itens

- **#1103** (LLM summarization, VijaySreekar — ATIVO, não nosso) — **ponta OPOSTA do pipeline**: #1103 resume em *retrieval-time* o que SAI da busca (economiza tokens de saída, query-aware); este RFC extrai em *harvest-time* o que ENTRA na memória. Complementares, não colidem. NÃO é o nosso two-phase extractive (memory_explore/detail, #56 — esse já shippou, é LLM-free).
- **RFC-harvest-provenance** — este RFC herda a proveniência dela; é a Fase seguinte ("cobertura de conteúdo" após "proveniência + tracker").
- **#1287** (beacon loop) — a filosofia server-side + implícito + yield-contado vem de lá.
- **Session Miner** (task-orch `89feeabc`, done) — o protótipo standalone que originou o harvest de produção; este RFC reabre a dimensão "conteúdo rico" que o miner não resolveu.

---

## 6. Migração e compatibilidade (usual four — Henry)

Qualquer coisa que mude **o que o harvest escreve** precisa de migração/compat declarada:
- **Phase 0 (R0.x)** não muda o que é escrito — só instrumenta. Zero migração, seguro por padrão.
- **R1 (ToolResult no parser)** muda o que ENTRA como candidato — aditivo; sessões antigas não são re-harvestadas retroativamente (o R10 já decidiu: não recolher em massa). Só afeta harvest novo daqui pra frente.
- **R4 (marca `harvest:mode:design`)** é tag/metadata aditiva (padrão RFC-harvest-provenance), backward-compatible.
- **Default OFF (R5)** garante que nada muda para setups existentes até opt-in explícito.

## 7. Estado

DRAFT v0.2 — pronto para abrir como issue própria (não dobrar no #1286; Henry pediu thread própria). Após abrir: rodar o Phase 0 (instrumento de cobertura) e trazer os números de cobertura por tipo de bloco ANTES de propor o extractor — o padrão "bring numbers first" que funcionou nos PRs anteriores. O escopo do primeiro PR é o próprio Phase 0 (instrumento), não o extractor.
