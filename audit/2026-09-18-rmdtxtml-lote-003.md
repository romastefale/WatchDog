# Auditoria técnica RMDtxtML — lote 003

**Data:** 2026-09-18  
**Alvo:** `romastefale/RMDtxtML`, branch `draft`  
**Árvore-base auditada:** `692995e8fcdf5bac250605cdda20c23a0f1f729b`  
**Escopo:** conformidade estrita com Telegram Bot API 10.3 / Rich Messages.

**Fonte normativa usada neste lote:**
- https://core.telegram.org/bots/api#rich-messages
- https://core.telegram.org/bots/api#rich-message-formatting-options

O README afirma explicitamente que o produto cria e publica Rich Messages da **Telegram Bot API 10.3**. Portanto, diferenças entre o schema local e a gramática/objetos oficiais são tratadas como divergência funcional.

## WD-021 — ALTO — `button_row` aceita até 20 botões; Bot API 10.3 limita o bloco a 1–8

**Arquivo:** `client/editor.mjs`

A validação local aceita arrays de botões com tamanho 1–20. Na Bot API 10.3, `InputRichBlockButtons.buttons` é uma lista de **1–8 botões**.

**Impacto:** um modelo com 9–20 botões é considerado semanticamente válido pelo editor, serializado em Rich HTML e só pode falhar na API de destino.

**Correção:** limite 1–8 no modelo canônico, no importador e no backend.

## WD-022 — ALTO — suporte de botões 10.3 é incompleto

**Arquivos:** `client/editor.mjs`, `docs/app.js`

O schema/validador aceita somente `url`, `web_app`, `copy_text`, `switch_inline_query` e `disabled`.

A Bot API 10.3 também documenta `callback_data`, `login_url`, `switch_inline_query_current_chat` e `switch_inline_query_chosen_chat`, entre outros comportamentos associados.

**Impacto:** HTML 10.3 válido importado pode ser descartado/normalizado ou ficar irrepresentável no documento semântico.

## WD-023 — ALTO — botões inline `<tg-button>` não são representáveis

A Bot API 10.3 inclui `RichTextButton` e permite `<tg-button>` em conteúdo inline, inclusive dentro de parágrafo. O schema local só possui o node de bloco `button_row`; não há node/mark inline equivalente.

**Impacto:** mensagem 10.3 válida contendo botão inline não faz round-trip HTML → modelo → HTML.

## WD-024 — ALTO — texto do botão é reduzido a string e perde entidades permitidas

Cada botão importado guarda `label: b.textContent || ''`. A Bot API 10.3 permite no texto do botão plain text, custom emoji e date-time.

**Impacto:** `<tg-time>` e `<tg-emoji>` embutidos no botão são achatados para texto e perdem semântica.

## WD-025 — ALTO — collage/slideshow descartam vídeos aceitos oficialmente

**Arquivo:** `client/editor.mjs`

O importador usa apenas `querySelectorAll(':scope > img')` e o serializer emite somente `img`. A documentação oficial mostra `tg-collage` e `tg-slideshow` contendo **img e video**, inclusive combinados.

**Impacto:** vídeos desaparecem silenciosamente em importação/migração/round-trip.

## WD-026 — ALTO — referências `tg://photo|video|document|audio?id=...` são aceitas, mas o envio não popula `rich_message.media`

**Arquivos:** `client/editor.mjs`, `src/server.mjs`

O helper de mídia aceita referências `tg://photo`, `tg://video`, `tg://document` e `tg://audio`. Porém `send()` constrói `const richMessage={html:checked.html};` e nunca adiciona `rich_message.media`.

Na Bot API 10.3, `InputRichMessage.media` associa os IDs usados nesses links aos respectivos `InputMedia*`.

**Impacto:** o modelo declara válidas referências que o pipeline de envio não materializa.

## WD-027 — ALTO — tipo da referência `tg://` não é vinculado ao tipo do node

O mesmo validador de src é usado para `image`, `video`, `audio` e `document`. Assim, estados como `image.src = tg://document?id=x` ou `video.src = tg://audio?id=x` passam pela validação.

**Impacto:** o modelo “validado” pode gerar Rich HTML semanticamente incoerente com a mídia declarada.

## WD-028 — MÉDIO/ALTO — `<img src="tg://emoji?id=...">` é classificado como mídia de bloco

A documentação oficial permite essa sintaxe como representação de custom emoji. O parser local aceita o src no node `image`, que é de bloco, e depois o serializer o envolve em `figure`.

**Impacto:** uma entidade inline pode virar mídia de bloco, alterando significado e layout.

## WD-029 — MÉDIO — captions de tabela não são representáveis

A Bot API 10.3 suporta `<caption>` em `table`. O node local possui `bordered`, `striped` e `compact`, mas não possui caption; o serializer produz diretamente `tbody`.

**Impacto:** captions válidos desaparecem no round-trip.

## WD-030 — MÉDIO — caption de mapa não é representável

A documentação oficial suporta `figure` contendo `tg-map` e `figcaption`. O node local de mapa contém somente `lat`, `long` e `zoom`.

**Impacto:** perda de dados ao importar Rich HTML válido.

## WD-031 — MÉDIO — atributos por-item de lista ordenada são descartados

A Bot API 10.3 suporta `li value="7" type="i"`. O node local `list_item` possui somente `checked`.

**Impacto:** numeração explícita e estilo por item desaparecem.

## WD-032 — MÉDIO/ALTO — regras específicas de estilo/atributos de botão não são validadas

A Bot API restringe `style` a `danger`, `success`, `primary` ou `link`, sendo `link` permitido apenas para callback buttons, e exige exatamente uma ação além de texto/style.

A implementação aceita style arbitrário e pode serializar simultaneamente vários campos de ação (`url`, `data`, `text`, `query`).

**Impacto:** objetos inválidos passam por `assertSafeModel`.

## Síntese

O README estabelece Bot API 10.3 como contrato funcional, enquanto o schema local é um **subconjunto parcialmente incompatível** e, em alguns pontos, aceita estados que a própria API não aceita.

Se o produto optar deliberadamente por um subconjunto, esse subconjunto precisa ser enumerado, rejeitado explicitamente fora dele, coberto por fixtures de conformidade e não descrito genericamente como suporte 10.3 integral.

**Acumulado:** 32 achados registrados nos lotes 001–003.