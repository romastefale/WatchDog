# Auditoria técnica RMDtxtML — lote 004

**Data:** 2026-09-18  
**Alvo:** `romastefale/RMDtxtML`, branch `draft`  
**Árvore-base auditada:** `692995e8fcdf5bac250605cdda20c23a0f1f729b`  
**Escopo:** concorrência lógica, transferências, idempotência, persistência e abuso operacional.

## WD-033 — ALTO — transferência do mesmo documento pode sobrescrever silenciosamente uma versão local mais nova

**Arquivo:** `docs/document.js`  
**Função:** `adoptTransfer()`

Quando `transfer.document.id === current.id`, a implementação considera `same=true`, preserva o histórico local e substitui o modelo pelo conteúdo transferido. Porém não compara `transfer.document.revision` com `current.meta.revision`.

Sequência possível:

1. dispositivo A possui documento `id=X`, revisão 12;
2. dispositivo B ainda possui uma cópia antiga de `X`, revisão 7;
3. B cria uma transferência;
4. A reivindica a transferência;
5. `adoptTransfer()` detecta apenas que o ID é igual e substitui o conteúdo da revisão 12 pelo conteúdo antigo da revisão 7;
6. o checkpoint posterior incrementa a revisão local, fazendo o conteúdo antigo parecer uma nova revisão.

**Impacto:** stale write / perda silenciosa de atualização. A identidade do documento existe, mas não há controle otimista de concorrência.

**Cobertura:** o teste existente de adoção usa IDs diferentes e não cobre conflito same-ID/newer-local.

**Correção:** compare-and-swap semântico: exigir base revision, detectar `incomingRevision < currentRevision`, oferecer conflito/merge ou criar fork.

## WD-034 — ALTO — idempotência não é vinculada ao payload; troca de destino pode resultar em falso sucesso

**Arquivos:** `src/store.mjs`, `src/server.mjs`, `docs/app.js`

A chave persistida é apenas `(user_id, request_id)`. Não são armazenados nem comparados:

- destination/chat;
- hash do HTML;
- `is_rtl`;
- `skip_entity_detection`;
- demais flags de envio.

O frontend mantém `sendRequestId` após erro de transporte para permitir retry idempotente. Alterações no editor limpam a chave, mas a troca de `destinationId` no `<select>` **não** limpa `sendRequestId`.

Falha concreta:

1. envio para destino A chega ao Telegram e o servidor grava estado `done`;
2. a resposta HTTP ao cliente se perde;
3. o cliente preserva o mesmo `requestId`;
4. usuário troca o destino para B;
5. retry usa a mesma chave com payload/destino diferente;
6. servidor encontra `done` e devolve o resultado antigo com `idempotent:true`, sem enviar para B;
7. UI pode indicar sucesso embora B não tenha recebido nada.

**Correção:** persistir fingerprint canônico do request e rejeitar reutilização da chave com payload diferente (`409 idempotency_key_reused`); no cliente, limpar a chave quando qualquer parâmetro material do envio mudar.

## WD-035 — ALTO — criação anônima + limite global de 1.000 permite expulsar transferências legítimas

**Arquivos:** `src/server.mjs`, `src/store.mjs`

`POST /api/transfers` não exige `initData`. Em `createTransfer()`, quando existem 1.000 transferências ativas, o Store remove as mais antigas até abrir espaço.

Portanto um atacante capaz de criar transferências suficientes pode provocar a remoção antecipada de tokens válidos de outros usuários.

Este achado compõe-se diretamente com WD-004 e WD-005:

- rate limit usa headers de proxy potencialmente spoofáveis;
- ausência de `Origin` é tratada como origem confiável;
- criação não exige identidade Telegram.

**Impacto:** negação de serviço direcionada ao handoff Web → Telegram.

**Correção:** autenticar criação quando possível; quota por usuário/sessão; não expulsar transferências válidas de terceiros; rejeitar criação com 429/503 quando capacidade global for atingida.

## WD-036 — ALTO — endpoint anônimo permite pressão de disco substancial

O corpo HTTP aceita até 512 KiB. Cada transferência pode persistir HTML, envelope semântico e metadados. O limite global é de 1.000 transferências ativas.

Ordem de grandeza: centenas de MiB de conteúdo ativo podem ser mantidos/reciclados no SQLite por clientes anônimos, além de WAL e overhead de páginas.

O mecanismo de cap não impede churn: ele apaga os mais antigos e continua aceitando novos registros.

**Impacto:** I/O, crescimento de WAL/database, desgaste operacional e expulsão de transferências úteis.

**Correção:** orçamento total em bytes, quota autenticada, limite menor por origem/usuário, rejeição em capacidade e métricas de storage pressure.

## WD-037 — MÉDIO/ALTO — `INIT_DATA_MAX_AGE` inválido desativa a expiração em vez de falhar fechado

**Arquivos:** `src/server.mjs`, `src/telegram.mjs`

O servidor passa:

`maxAgeSeconds: Number(env.INIT_DATA_MAX_AGE || 86400)`

e a validação usa:

`if (nowSeconds - authDate > maxAgeSeconds) ...`

Se a variável estiver definida com valor não numérico, `Number(...)` vira `NaN`; comparações `x > NaN` são sempre falsas.

**Impacto:** uma configuração incorreta pode remover silenciosamente a verificação de idade da sessão Telegram.

**Correção:** parse/bounds na inicialização e encerramento do processo em configuração inválida; nunca aceitar `NaN` como política.

## WD-038 — MÉDIO — cache global de username pode cruzar tokens/bots no mesmo processo

**Arquivo:** `src/server.mjs`

`botName` é variável global de módulo. `botUsername()` retorna o valor cacheado antes de consultar o token recebido.

Se duas instâncias de `createServer()` com tokens diferentes coexistirem sequencialmente/no mesmo processo e `BOT_USERNAME` não estiver fixado, o segundo contexto pode reutilizar o username descoberto para o primeiro bot.

**Impacto:** URL `t.me/<username>?startapp=...` pode apontar para bot diferente daquele cujo token protege o backend.

**Correção:** cache por token/instância, não global.

## WD-039 — MÉDIO — healthcheck executa manutenção mutável no SQLite

`GET /api/health` chama `data.prune()` antes de responder. `prune()` executa DELETEs/UPDATEs sobre transfers, rates e sends.

Healthchecks externos são normalmente frequentes. Mesmo sem linhas afetadas, o endpoint de liveness/readiness deixa de ser estritamente read-only e pode competir por write locks/I/O com claim/send.

**Impacto:** acoplamento desnecessário entre observabilidade e caminho de escrita; sob pressão, um healthcheck pode contribuir para latência ou falsos negativos.

**Correção:** health read-only; manutenção por timer interno controlado ou no caminho das operações que já escrevem.

## WD-040 — MÉDIO — tentativa de persistência em `pagehide` é assíncrona e não é garantida

**Arquivo:** `docs/app.js`

O código chama `persistDocument(...).catch(...)` em `pagehide`, mas não existe mecanismo que mantenha a página viva até a conclusão da transação IndexedDB.

Há autosave com debounce de 700 ms, então uma edição seguida de fechamento/navegação rápida pode ocorrer antes do autosave; o `pagehide` assíncrono reduz o risco, mas não oferece garantia transacional.

**Impacto:** janela de perda dos últimos edits.

**Correção:** reduzir dependência de lifecycle terminal; persistir incrementalmente/mais cedo, usar `visibilitychange` como sinal principal e testar fechamento imediato após edição.

## WD-041 — MÉDIO — conteúdo reclamado permanece armazenado por até uma hora

`pruneTransfers` só remove uma transferência já reclamada quando `claimed_at <= now - 1h`.

O claim é one-time no nível de leitura, mas HTML/modelo continuam no banco após consumo.

**Impacto:** retenção desnecessária de conteúdo potencialmente sensível e maior superfície em caso de acesso ao volume.

**Correção:** apagar payload no mesmo transaction boundary do claim ou substituir o conteúdo por tombstone mínimo necessário para impedir segundo claim.

## Síntese do lote

Os riscos mais relevantes aparecem nas interações entre mecanismos corretos isoladamente: bearer transfer token + capacidade global + rate limit por IP; idempotência + troca de destino; identidade documental + ausência de controle de revisão.

Esses casos precisam de testes de estado e concorrência, não apenas testes unitários de happy path.

**Acumulado:** 41 achados registrados nos lotes 001–004.