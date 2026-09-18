# Auditoria técnica RMDtxtML — lote 005

**Data:** 2026-09-18  
**Alvo:** `romastefale/RMDtxtML`, branch `draft`  
**Árvore-base auditada:** `692995e8fcdf5bac250605cdda20c23a0f1f729b`  
**Escopo:** fidelidade de campos Rich Message 10.3, captions, details, mapa e URLs de botões.

**Fonte normativa:** documentação oficial Telegram Bot API atual em 2026-09-18: https://core.telegram.org/bots/api#rich-messages

## WD-042 — ALTO — captions de mídia são achatadas para texto e perdem RichText + credit

**Arquivo:** `client/editor.mjs`

Nos nodes `image`, `video`, `audio` e `document`, a importação de caption usa `figcaption.textContent`. Em `collage` e `slideshow` ocorre o mesmo.

A Bot API 10.3 define `RichBlockCaption` com:

- `text: RichText`;
- `credit: RichText`, correspondente a `<cite>`.

Assim, um caption como:

`<figcaption><strong>Título</strong> <cite><a href="https://example.com">Fonte</a></cite></figcaption>`

é reduzido a uma string plana. Bold, links, custom emoji, time e o papel semântico de `cite` são perdidos.

**Impacto:** round-trip não é lossless para captions oficialmente suportadas; migração schema 1 pode degradar conteúdo sem aviso.

**Correção:** modelar caption como estrutura semântica RichText + credit, não como string.

## WD-043 — ALTO — `details.summary` é RichText oficial, mas o modelo local guarda string

**Arquivo:** `client/editor.mjs`

O parser usa `summary.textContent`, e o node `details` armazena `summary` em atributo string.

Na Bot API 10.3, `InputRichBlockDetails.summary` é `RichText`. Portanto marks e entidades no `<summary>` não são preservados.

Este achado é adicional ao WD-011: WD-011 trata o corpo de `details`; WD-043 trata especificamente o summary.

**Impacto:** perda de formatação e entidades no título expansível.

## WD-044 — MÉDIO/ALTO — mapa não representa `width` e `height` oficiais

**Arquivo:** `client/editor.mjs`

O node `map` possui somente:

`lat`, `long`, `zoom`.

A Bot API 10.3 também define `width` e `height` opcionais e impõe restrições geométricas: soma/dimensões limitadas e razão largura/altura limitada.

Como os campos não existem no modelo local, Rich HTML que os utiliza não pode ser preservado. E, se forem adicionados futuramente sem validator geométrico, o atual `assertSafeModel` também não possui a lógica necessária.

**Impacto:** subconjunto incompleto e round-trip lossy de mapas 10.3.

## WD-045 — ALTO — botão `web_app` aceita HTTP, embora WebAppInfo exija HTTPS

**Arquivos:** `docs/app.js`, `client/editor.mjs`

A UI usa `promptHttp()`, cujo regex aceita `http://` e `https://`. O validador semântico também usa `^https?://` para `web_app`.

A Bot API define `RichMessageButton.web_app` como `WebAppInfo`; `WebAppInfo.url` deve ser HTTPS.

**Impacto:** o editor cria e valida localmente um botão que pode ser recusado pela API.

**Correção:** `web_app` deve exigir `https://` especificamente; não compartilhar o mesmo validator do botão URL genérico.

## WD-046 — ALTO — botão URL rejeita `tg://` oficialmente permitido

**Arquivo:** `client/editor.mjs`

Para `button.type === 'url'`, `assertSafeModel` exige `^https?://`.

A Bot API 10.3 permite URL HTTP ou `tg://` no campo URL de `RichMessageButton`, e exemplos oficiais usam links Telegram.

Há uma inconsistência interna adicional: links textuais comuns do editor já aceitam `tg://user?id=...`, mas botões URL não.

**Impacto:** conteúdo oficialmente válido não é representável/validável no schema local.

## WD-047 — MÉDIO — UI promete “URL HTTPS” mas aceita HTTP

**Arquivo:** `docs/app.js`

`promptHttp()` mostra ao usuário mensagens como “URL HTTPS”, mas valida com `/^https?:\/\//i`.

Isto afeta imagem, vídeo, áudio, documento, collage/slideshow e `web_app`.

**Impacto:** contrato visual e política real divergem. Para mídia HTTP isso pode ser permitido pela gramática do Rich Message, mas para `web_app` torna-se erro funcional (WD-045).

**Correção:** separar helpers por política (`httpOrHttps`, `httpsOnly`, `telegramDeepLink`) e mensagens coerentes.

## WD-048 — MÉDIO/ALTO — caption de collage/slideshow também perde estrutura própria

Embora WD-042 cubra o achatamento geral de captions, collage/slideshow agravam a divergência: o node já reduz os itens de mídia a `{src,alt}` e o caption a uma string única.

Na API oficial, `InputRichBlockCollage.caption` e `InputRichBlockSlideshow.caption` são `RichBlockCaption`, preservando RichText e credit.

**Impacto:** duas dimensões de perda no mesmo bloco: tipo/estrutura dos itens (WD-025) e estrutura do caption.

## WD-049 — MÉDIO — mapa perde também RichBlockCaption completo

WD-030 já registrou a ausência total de caption no mapa. A especificação mostra que esse caption não é apenas texto: é `RichBlockCaption`, portanto também pode possuir `credit` e RichText.

**Impacto:** a lacuna funcional do mapa é maior que a simples ausência de uma string de legenda.

## WD-050 — MÉDIO — esquema semântico não expressa formalmente o subconjunto suportado da API

Há recursos 10.3 que o schema rejeita, recursos que ele aceita parcialmente e estados que ele considera válidos apesar de incompatíveis com a API.

Os lotes 003 e 005 mostram que a fronteira não é uma lista simples de features ausentes: o mesmo conceito pode ser parcialmente suportado (ex.: botão, caption, mapa) com perda de atributos ou invariantes.

**Impacto:** consumidores do formato `.rmdtxtml` não conseguem inferir, a partir de `schema:2/modelVersion:1`, qual subconjunto Telegram é garantidamente preservado.

**Correção:** publicar um contrato versionado do modelo, incluindo matriz de `supported`, `lossy`, `unsupported`, e tratar mudanças de fidelidade como evolução de modelVersion.

## Síntese do lote

Os maiores problemas de fidelidade estão em campos cujo tipo oficial é **RichText/RichBlockCaption**, mas que foram modelados como strings. Isso cria perda silenciosa justamente em importação/migração, onde o README promete continuidade do documento.

**Acumulado:** 50 achados registrados nos lotes 001–005.