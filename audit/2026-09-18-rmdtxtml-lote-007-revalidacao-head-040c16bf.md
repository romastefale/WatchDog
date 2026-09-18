# Auditoria técnica RMDtxtML — lote 007

**Data:** 2026-09-18  
**Alvo:** `romastefale/RMDtxtML`, branch `draft`  
**Commit revalidado:** `546c4ad9fccd6dd92584fe9bafadc66d831c44c6`  
**Árvore revalidada:** `040c16bf201046796aaf4e37622debbca06a5606`  
**Escopo:** revalidação após mudança da branch durante a auditoria + novas divergências do renderer `blocks`/modelVersion 2.

> Nota temporal: os lotes 001–006 auditam a árvore anterior `692995e8fcdf5bac250605cdda20c23a0f1f729b`. Durante a auditoria, `draft` avançou. Este lote inicia uma nova baseline e não trata achados corrigidos como se ainda estivessem presentes.

## Mudanças relevantes observadas

A nova árvore introduziu `client/rich-message.mjs`, elevou `DOCUMENT_MODEL_VERSION` para 2 e passou a renderizar documentos com recursos avançados usando `InputRichMessage.blocks`. Também houve correções materiais de findings anteriores:

- `details` agora possui summary + blocos estruturais;
- captions são conteúdo inline e podem carregar `cite`;
- tabela ganhou caption;
- mapa ganhou width/height/caption;
- collage/slideshow carregam imagem ou vídeo;
- botões podem ser inline e `button_row` é 1–8;
- vários tipos adicionais de botão foram modelados;
- item de lista ganhou `value`/`type`;
- enums de tabela/botões e restrições de mapa foram fortalecidos.

Essas correções resolvem ou alteram substancialmente WD-011, WD-021, WD-023–WD-025, WD-029–WD-032 e WD-042–WD-046 na baseline antiga. Outros achados permanecem e serão revalidados separadamente.

## WD-061 — ALTO — backend de `blocks` não valida a união RichText

**Arquivo:** `src/telegram.mjs`  
**Funções:** `richTextCount()`, `inspectBlocks()`

`richTextCount()` serve apenas para contar caracteres. Para um objeto desconhecido, ele recursiona em `.text` e não verifica o discriminator `type` nem campos obrigatórios.

Exemplo conceitual que atravessa a validação estrutural:

`{type:'paragraph', text:{type:'tipo_inexistente', text:'x'}}`

O bloco `paragraph` é aceito, o contador devolve 1 e o objeto original é encaminhado à Telegram API.

A Bot API 10.3 define uma união fechada de tipos `RichText`.

**Impacto:** o backend afirma validar a mensagem, mas aceita árvores RichText semanticamente inválidas e desloca a falha para Telegram.

**Correção:** validator recursivo discriminado para cada variante RichText, incluindo profundidade, campos, tipos e domínios.

## WD-062 — ALTO — validator server-side de RichMessageButton não aplica invariantes específicas da ação

**Arquivo:** `src/telegram.mjs`

Para blocos `buttons`, o backend verifica:

- 1–8 itens;
- exatamente uma chave de ação;
- enum de style.

Mas não valida, entre outros:

- `web_app.url` HTTPS;
- `login_url.url` HTTPS;
- `callback_data` com 1–64 bytes;
- esquema permitido de `url`;
- shape/limites de `copy_text`;
- shape de `switch_inline_query_chosen_chat`;
- regra `style=link` somente para callback.

A documentação oficial 10.3 exige essas propriedades.

**Impacto:** um caller autenticado pode contornar o validator do editor e fazer o backend encaminhar um botão inválido.

## WD-063 — ALTO — UI ainda consegue inserir `web_app` HTTP e contorna `assertSafeModel` no momento da mutação

**Arquivos:** `docs/app.js`, `client/editor.mjs`

O validator do modelo foi corrigido para exigir `https://` em `web_app` e `login_url`. Porém a UI ainda chama `promptHttp('URL HTTPS')`, e `promptHttp()` aceita `http://`.

Além disso:

`insertSpec(spec) { return this.insertNode(schema.nodeFromJSON(spec)) }`

`schema.nodeFromJSON()` valida o shape ProseMirror, mas não chama `assertSafeModel()`. Portanto um botão `web_app` HTTP pode entrar no `EditorState` antes da validação canônica.

**Impacto:** a UI pode criar estado que o próprio `normalizeModel()` considera inválido. Chamadas posteriores a `html()/metrics()/render()` podem falhar depois que o estado já foi mutado.

**Correção:** validar/normalizar antes de `insertNode`; separar `promptHttps` de `promptHttp`.

## WD-064 — ALTO — modelo aceita `tg://...` de mídia, mas o renderer Blocks e o backend não conseguem publicá-lo

**Arquivos:** `client/editor.mjs`, `client/rich-message.mjs`, `src/telegram.mjs`

O helper `media()` ainda aceita `tg://photo|video|document|audio|emoji?id=...` para nodes de mídia.

Na nova arquitetura, qualquer node de mídia ativa `blocks`. `mediaBlock()` coloca o src diretamente em `InputMedia*.media`. O backend, por sua vez, exige `^https?://` nesse campo.

Assim, o mesmo modelo é:

1. aceito por `assertSafeModel`;
2. renderizado para Blocks;
3. rejeitado no backend como `invalid_media`.

Além disso, na forma HTML, referências `tg://photo?id=...` exigem a lista `InputRichMessage.media`; o renderer atual não constrói essa tabela.

**Impacto:** estado semanticamente “válido” porém não publicável.

## WD-065 — ALTO — mudança para Blocks força animação e voice note a tipos errados

**Arquivos:** `client/editor.mjs`, `client/rich-message.mjs`

Na sintaxe Rich HTML oficial, Telegram determina o tipo de mídia pelo MIME/URL; os próprios exemplos usam:

- `<video ...animation.gif>` para animation;
- `<audio ...audio.ogg>` para voice note.

O schema local possui apenas `video` e `audio`. Como esses nodes ativam `blocks`, o renderer passa a enviar explicitamente:

- `InputRichBlockVideo` / `InputMediaVideo` para todo `<video>`;
- `InputRichBlockAudio` / `InputMediaAudio` para todo `<audio>`.

Não existem nodes `animation` e `voice_note` para preservar a distinção.

**Impacto:** conteúdo que no HTML seria classificado corretamente por MIME pode mudar de tipo ou ser rejeitado quando convertido para Blocks.

**Correção:** representar o tipo semântico real ou manter HTML para mídias cujo tipo depende de detecção.

## WD-066 — ALTO — links para `tg-reference` viram `anchor_link` no renderer Blocks

**Arquivo:** `client/rich-message.mjs`

O mark `reference` é renderizado como `RichTextReference`. Entretanto qualquer link cujo href começa por `#` é sempre convertido em:

`{type:'anchor_link', anchor_name: ...}`

A Bot API possui tipos distintos `RichTextAnchorLink` e `RichTextReferenceLink`.

Consequência: um documento com `<tg-reference name="note-1">...</tg-reference>` e `<a href="#note-1">...</a>` funciona em HTML, mas, se qualquer recurso avançado fizer `usesBlocks()` retornar true, o link passa a ser enviado como anchor link em vez de reference link.

**Impacto:** semântica depende da representação escolhida para a mensagem inteira; adicionar um heading/tabela/mídia pode alterar o significado de links internos existentes.

**Correção:** resolver referências pelo símbolo alvo durante a compilação para Blocks.

## WD-067 — MÉDIO/ALTO — contador do frontend duplica caption e texto de botão no modelVersion 2

**Arquivo:** `client/editor.mjs`  
**Função:** `nodeText()`

A função começa com:

`let text = node.textContent || ''`

No modelVersion 2, captions e labels de botão já são conteúdo filho e portanto já aparecem em `doc.textContent`. Depois, o walker soma novamente:

- `child.textContent` para image/video/audio/document/map/collage/slideshow;
- `child.textContent` para button.

**Impacto:** `stats().text` fica maior que o conteúdo real. A UI usa esse valor para bloquear envio acima de 32.768, podendo rejeitar mensagens que estão dentro do limite.

**Correção:** contar cada unidade semântica uma única vez; adicionar separadamente apenas leaf atoms que não entram em `textContent` (math/custom emoji/time).

## WD-068 — ALTO — migração modelVersion 1 → 2 trunca silenciosamente button rows

**Arquivo:** `client/editor.mjs`  
**Função:** `migrateModel()`

A versão antiga permitia até 20 botões por row. Na migração atual:

`buttons.slice(0,8)`

mantém apenas os oito primeiros.

**Impacto:** documento persistido legítimo da versão anterior pode perder botões 9–20 durante load/import, sem warning, sem fork e sem marca de migração lossy.

**Correção:** falhar com diagnóstico/migração assistida, ou dividir deterministamente em múltiplas rows se isso preservar a intenção e for compatível com o contrato.

## WD-069 — ALTO — servidor aceita versões semânticas que o cliente não consegue migrar; claim pode consumir o token antes da falha

**Arquivos:** `src/server.mjs`, `docs/document.js`, `client/editor.mjs`

`transferSemantic()` aceita `modelVersion` de 1 a 1000 e continua sem validar o modelo. O cliente atual suporta modelVersion 2 e migração explícita apenas de 1 → 2.

Uma transferência com, por exemplo, `modelVersion:7` pode ser criada e armazenada. No claim, o backend marca o token como utilizado antes de o navegador tentar `adoptTransfer()`. A adoção então pode lançar `unsupported_model_version`.

**Impacto:** transferência formalmente aceita pelo servidor pode ser irrecuperável para o cliente atual depois do primeiro claim.

**Correção:** negociar versões suportadas antes do claim ou manter claim em estado pendente até confirmação do cliente; no mínimo rejeitar na criação versões não suportadas pelo release atual.

## WD-070 — MÉDIO — README descreve renderer Rich HTML, mas envio avançado agora usa `blocks`

**Arquivo:** `README.md`

O diagrama e a narrativa ainda descrevem `renderer Rich HTML → Telegram Bot API` e enfatizam validação Rich HTML no backend.

Na baseline atual, `renderRichMessage()` usa `blocks` sempre que encontra um node classificado como avançado; apenas documentos simples continuam no modo HTML.

**Impacto:** documentação arquitetural está desatualizada justamente na fronteira que decide serialização e validação.

**Correção:** documentar os dois backends de render (`HTML` e `blocks`), a regra de seleção e as garantias de equivalência.

## Achados antigos ainda confirmados nesta baseline

Sem renumerar, a inspeção do novo head confirma que continuam presentes, entre outros:

- WD-001/002: validator Rich HTML regex/blacklist permanece;
- WD-003: destinos configurados continuam globais para qualquer usuário Telegram validado;
- WD-004: IP continua confiando headers de proxy sem trust boundary explícita;
- WD-005: Origin ausente continua aceito;
- WD-006: envelope semântico server-side continua sem validação do schema;
- WD-008: continua sem lockfile e com `npm install --no-audit`;
- WD-033: transferência same-document continua sem controle de revisão;
- WD-034: idempotência continua sem fingerprint do payload e troca de destino não limpa `sendRequestId`;
- WD-035/036: criação anônima + capacidade global/pressão de SQLite continuam;
- WD-037: `INIT_DATA_MAX_AGE` ainda pode virar NaN e falhar aberto;
- WD-038/039/041: cache global de bot, health mutável e retenção pós-claim permanecem;
- WD-051/052: fallback SPA sob `/api/*` e prefix match de health permanecem;
- WD-054/055/056/057/059: CSP, usuário root, cobertura de workflow/supply-chain e matriz de browsers permanecem.

## Síntese

A nova árvore corrige uma fração importante das divergências de formato, mas a mudança para um compilador `model → InputRichMessage.blocks` criou uma nova classe de bugs: **o modelo, o compilador e o validator do servidor agora formam três contratos diferentes**.

A próxima etapa deve tratar o compilador Blocks como uma fronteira formal, com validator recursivo compartilhado e fixtures Telegram 10.3.

**Histórico:** 60 achados anteriores + 10 achados/revalidações novas nesta baseline.