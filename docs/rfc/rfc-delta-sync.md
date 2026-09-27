# RFC: Delta Event-Log Sync (malha de memória multi-agente)

**Data:** 2026-09-13 (rev. 2026-09-27)
**Autor:** Claudio + Zero (Kiro CLI)
**Branch de código:** `feat/delta-sync` (a partir de `upstream/main`)
**Base:** `upstream/main` v11.14.0+ (agent_id #1100 fases 1+2 JÁ MERGEADAS)
**Versão:** 0.2 (draft — pronto para abrir como issue, a pedido do Henry no #1304)
**Inspiração:** Mnemosyne `sync.py` / `sync_server.py` (delta event-log + cripto client-side)
**Reintegra:** dor de sync via Insync (tasks internas 5d41dda2 stale-reads, corrupção SQLite+WAL)
**Relacionado:** #1304 (hybrid remote secondary — o caminho *hybrid-nativo* do mesmo destino; ver §7), #1100 (agent_id, base de R6), #57 upstream (federated retrieval, fora de escopo)
**Status:** DRAFT v0.2 — a abrir como issue própria no GitHub (Henry pediu "file it as its own issue" no #1304).

---

## 1. Problema

O sync de memória entre máquinas e agentes (Zero em 3 máquinas; T'Pol/Scotty na VPS) é feito hoje por **file-sync do `.db` inteiro** via Insync/OneDrive + hot-backup com score composto. Isso transfere o arquivo todo, corre risco de corrupção e não modela "memória por agente".

### Evidências

- Banco de 457 MB sincronizado inteiro a cada operação (Insync) — custo escala com o tamanho.
- Corrupção documentada: "Insync + SQLite = corrupção — WAL ativo durante sync" (lessons learned).
- Stale reads: task interna 5d41dda2 — o processo mantém conexão aberta e não vê o `.db` atualizado por outra máquina.
- Modelo atual não é "memória por agente": é um banco copiado, não uma troca deliberada entre agentes (Zero não "sabe o que T'Pol fez" de forma seletiva — ou copia tudo, ou nada).

### 🔬 Experimento exploratório (19/set/2026, backups reais dos 3 hosts)

Comparados os `content_hash` dos bancos mais recentes de cada host:

| host | memórias | tamanho |
|------|---------:|--------:|
| DNBSCDC289 | 21.757 | 449 MB |
| sirdata | 22.258 | 474 MB |
| socrates | 21.454 | 371 MB |

- Comum aos 3: **20.865** · União: **22.446** · **Divergência: 1.581 (7,0%)**.
- Exclusivas: DNBSCDC289 **0**, sirdata 100, socrates 188.

**Achados que dimensionam a RFC:**
1. O full-file sync transfere **~432 MB por operação por host**, mas o delta real é
   **7%** (1.581 memórias) — ~93% de cada sync é redundante. Um delta-sync por
   content_hash transferiria só o que difere → economia de ordem de magnitude.
2. O sync converge razoavelmente (93% comum) mas DERIVA: socrates tem 188 memórias
   exclusivas (locais que não subiram), sirdata 100. Deriva é justamente o que um
   event-log delta corrige — hoje o full-file não reconcilia seletivamente.
3. DNBSCDC289 com 0 exclusivas sugere que ele só recebe (ou o backup é mais antigo,
   17/set vs 19/set dos outros) — confirmar antes de fixar o modelo de reconciliação.
4. Os 4 registros de `store` corrompido (bytes de controle) estão nos 3 hosts →
   propagaram via full-file sync; o delta por hash não os re-propagaria se limpos.

### Causas

1. **Sync de arquivo, não de deltas.** Transfere o `.db` inteiro; não há protocolo de mudanças incrementais.
2. **Sem event-log.** Não há registro append-only de operações que permita reconciliação e auditoria.
3. **Sem fronteira por agente.** Não há noção de "o que este agente compartilha" vs "o que é privado".

### Risco / motivação estratégica

Corrupção e stale reads são problemas ativos. E o modelo de arquivo impede a visão de **malha de memória da tripulação**: cada agente com sua memória, compartilhando seletivamente o que faz sentido (Zero vê a pesquisa da T'Pol e a implementação do Scotty).

---

## 2. Objetivo

Substituir o file-sync por um **sync delta baseado em event-log**, bidirecional, com reconciliação por timeline+importância, opcionalmente com criptografia client-side, modelando compartilhamento **por agente**.

**Não-objetivos:** forçar criptografia (opcional); remover o hot-backup (permanece como rede de segurança); federar retrieval entre stores (RFC #57 upstream separada).

---

## 3. Requisitos (Prosa + EARS)

> Convenção EARS (DEVELOPMENT-STANDARDS §8.4.1).

### Funcional

**R1**: O sync transfere apenas mudanças desde o último sync.

> EARS: WHEN a sync runs, THE sync client SHALL transfer only the events changed since the last successful sync.

**R2**: As operações são registradas em log append-only auditável.

> EARS: WHEN a memory is created, updated or deleted, THE system SHALL append an event to an immutable event log with id, timestamp and operation type.

**R3**: O sync é bidirecional com reconciliação determinística.

> EARS: WHEN two instances have divergent events, THE sync SHALL reconcile them by timeline and importance, deterministically.

**R4**: O sync não corrompe o banco durante operação.

> EARS: WHILE syncing, THE sync SHALL operate over the event log and never copy the raw `.db` file with an active WAL.

**R5**: O conteúdo pode ser criptografado no cliente (opcional).

> EARS: WHERE client-side encryption is enabled, THE sync SHALL encrypt payload before transmission so the server sees only metadata.

**R6**: O compartilhamento é escopado por agente, usando o `agent_id` que **já existe** no upstream (v11.14.0: metadata.agent_id + header X-Agent-ID via #1278/#1297, tag `agent:<id>` no web layer).

> EARS: WHEN an agent marks memories as shareable, THE sync SHALL propagate only those to peers, keeping the rest private. THE agent identity SHALL be read from `metadata.agent_id` (the merged #1100 Phase 1/2 field), not from a new mechanism.

### Não-Funcional

**R7**: O sync é resiliente a stale reads.

> EARS: WHEN the local `.db` changes on disk, THE service SHALL detect the change (e.g. `PRAGMA data_version`) and avoid serving stale data.

**R8**: O hot-backup permanece como rede de segurança.

> EARS: THE delta sync SHALL coexist with the existing hot-backup rather than replace it during transition.

---

## 4. Design

- `sync.py`: event-log append-only (id, ts, op, payload, agent_id) + cliente delta bidirecional.
- `sync_server.py`: servidor HTTP (na VPS) com auth (API key/JWT); vê só metadata quando cripto ativa.
- Cripto opcional: Fernet/PyNaCl, chave client-side (nunca sai da máquina).
- Escopo por agente: flag `shareable` + `agent_id` na memória; reconciliação timeline+importância.
- Detecção de stale: `PRAGMA data_version` no serviço para reconectar/recarregar.
- Coexistência: manter Insync/hot-backup durante transição; migrar quando o delta sync provar estabilidade.

---

## 5. Fora de Escopo

- Federated retrieval cross-store (RFC #57 upstream).
- Remoção imediata do Insync (coexistência na transição).
- Criptografia obrigatória (opcional; rede LAN + OneDrive já é privada para nós).

---

## 6. Critérios de Aceite

- [ ] Sync transfere só deltas desde o último sync (não o `.db` inteiro).
- [ ] Event-log append-only auditável (id/ts/op/agent).
- [ ] Reconciliação bidirecional determinística por timeline+importância.
- [ ] Sync não copia `.db` com WAL ativo (sem corrupção).
- [ ] Compartilhamento escopado por agente (shareable vs privado).
- [ ] Serviço detecta mudança em disco e não serve dado stale.
- [ ] Coexiste com hot-backup na transição.

---

## 7. Relação com #1304 (hybrid remote secondary) — não bifurcar a superfície de sync

O Henry (no #1304) pediu que este framing fique **explícito**, para os dois esforços não divergirem:

- **#1304 = o caminho *hybrid-nativo*.** Torna o `secondary` do backend `hybrid` plugável (`cloudflare|http`), para que uma instância self-hosted seja o hub e cada cliente mantenha cache local. É **single-writer por cliente** contra um hub, drift detectado por `list_content_hashes()`. Resolve o hub-and-spoke de forma nativa no storage layer.
- **Esta RFC = o caminho *multi-writer*.** Event-log append-only com `agent_id`, reconciliação determinística, compartilhamento escopado por agente. Modela **N agentes escrevendo com autoria** (Zero/T'Pol/Scotty), não só N caches de um dono.

**Regra de convivência (acordada com o Henry):** o que **entrar primeiro** define a superfície; o que entrar **em segundo se dobra ao primeiro**. Como o #1304 é menor e já está sendo desenhado, é provável que ele entre antes — então esta RFC deve, quando implementada, **reusar** o que o #1304 estabelecer (o `remote_http` storage, o endpoint de bulk-hash, o model-match startup-check) em vez de criar um canal paralelo. O event-log é a camada de *autoria + reconciliação seletiva* por cima do transporte que o #1304 cria, não um transporte concorrente.

O ângulo de autoria se enuncia contra o `agent_id` **já mergeado** (#1100 fases 1+2), não contra a proposta original.
