# Auditoria técnica RMDtxtML — lote 002

**Data:** 2026-09-18  
**Alvo:** `romastefale/RMDtxtML`, branch `draft`  
**Árvore-base:** `692995e8fcdf5bac250605cdda20c23a0f1f729b`  
**Escopo:** modelo ProseMirror, projeção Rich HTML, round-trip, importação/migração e cobertura E2E.

## Sumário

Este lote confirma que a migração para documento semântico eliminou parte importante da dependência do DOM como fonte de verdade, mas introduziu divergências de fidelidade entre o modelo canônico e Rich HTML. Foram encontrados problemas de perda de estrutura em `details`, validação incompleta de botões e atributos, importação permissiva, validação de tabelas insuficiente e lacunas de testes justamente nos tipos ricos de maior complexidade.

## WD-011 — ALTO — `details` destrói estrutura e formatação no round-trip HTML → modelo → HTML

**Arquivo:** `client/editor.mjs`  
**Node:** `details`

Na importação, o corpo é reduzido a texto:

```js
body:[...el.children]
  .filter(x=>x.tagName!=='SUMMARY')
  .map(x=>x.textContent||'')
  .join('\n')
```

Na serialização, todo esse conteúdo volta como um único parágrafo:

```js
['details', attrs, ['summary', summary], ['p', body]]
```

Consequentemente, qualquer Rich HTML válido dentro de `details` — parágrafos múltiplos, listas, links, ênfase, mídia ou outra estrutura suportada — perde semântica na importação.

**Exemplo:**

Entrada:

```html
<details>
  <summary>S</summary>
  <p><strong>A</strong></p>
  <p><a href="https://example.com">B</a></p>
</details>
```

Após round-trip, o modelo retém essencialmente `A\nB` e a projeção não preserva os marks/blocos originais.

**Impacto:** perda silenciosa de dados durante migração schema 1, paste/import ou transferência antiga baseada em HTML.

**Correção:** modelar `details` como node com conteúdo estrutural real (summary + block content), não como attrs string.

## WD-012 — ALTO — validação de `button_row` não cobre semântica por tipo

**Arquivo:** `client/editor.mjs`  
**Função:** `assertSafeModel`

O código restringe tipos a:

`url`, `web_app`, `copy_text`, `switch_inline_query`, `disabled`.

Porém só exige URL para `url` e `web_app`. Não valida os campos obrigatórios/permitidos específicos dos demais tipos, nem comprimento/conteúdo de `data`, `text`, `query`, label, style ou align.

A serialização emite todos os campos presentes independentemente do tipo:

```js
type:b.type,
...(b.style?{style:b.style}:{}),
...(b.url?{url:b.url}:{}),
...(b.data?{data:b.data}:{}),
...(b.text?{text:b.text}:{}),
...(b.query?{query:b.query}:{})
```

**Impacto:** modelos considerados “seguros/válidos” pelo cliente podem produzir botões inválidos para a Bot API, deslocando erro para publicação.

**Correção:** discriminated union por tipo de botão e allowlist de atributos por variante.

## WD-013 — MÉDIO/ALTO — importador HTML é sanitizador parcial, não validador de Rich HTML

**Arquivo:** `client/editor.mjs`  
**Funções:** `cleanContainer`, `parseHtml`

`cleanContainer` remove apenas tags executáveis conhecidas e atributos `on*`. Depois, o parser ProseMirror ignora ou converte conteúdo desconhecido conforme suas regras.

Isso significa que importação pode **normalizar silenciosamente** HTML fora do contrato em vez de rejeitá-lo. Essa propriedade pode ser útil para paste de conteúdo arbitrário, mas é inadequada se a mesma função é usada como migração de um formato que se pretende preservar fielmente.

**Impacto:** documentos antigos podem mudar de significado sem diagnóstico explícito de elementos descartados.

**Correção:** separar dois pipelines:
1. paste permissivo/sanitizante;
2. importação/migração estrita com relatório de perdas.

## WD-014 — ALTO — validação de tabela não garante limite efetivo de 20 colunas por linha

**Arquivo:** `client/editor.mjs`  
**Função:** `assertSafeModel`

Cada célula é validada isoladamente (`colspan <= 20`), mas não há soma de `colspan` por `table_row`.

Um modelo com 21 células de `colspan=1`, ou 2 células com `colspan=20`, passa pela validação semântica do cliente. O backend possui uma tentativa independente de contar colunas no HTML, mas o modelo canônico é declarado válido antes disso.

**Divergência:** “modelo semântico valida estrutura e atributos” não inclui uma restrição estrutural central da mensagem rica.

**Correção:** validar cada linha agregando colspans e rejeitar largura efetiva >20.

## WD-015 — MÉDIO — `rowspan` é aceito até 100 sem validação estrutural da tabela resultante

O cliente permite `rowspan` 1..100, mas não verifica se o span ultrapassa número de linhas, causa sobreposição de células ou produz grade inconsistente.

**Impacto:** árvore ProseMirror válida pode representar tabela geometricamente inválida/ambígua para o renderer de destino.

**Correção:** construir grid lógico e validar ocupação por rowspan/colspan.

## WD-016 — MÉDIO — atributos enumerados são aceitos como strings arbitrárias

Exemplos:

- `ordered_list.style`;
- `table_cell.align`;
- `table_cell.valign`;
- `button_row.align`;
- `button.style`;
- `time.format`.

O `node.check()` do ProseMirror valida shape/tipos do schema, não domínio semântico dos valores. `assertSafeModel` não restringe esses enums.

**Impacto:** serialização de valores não suportados e falha tardia na Bot API.

**Correção:** enums explícitos alinhados à Bot API 10.3.

## WD-017 — MÉDIO — URL policy do editor e policy do backend não são equivalentes

O editor aceita links iniciados por:

```
#  http:  https:  mailto:  tel:  tg://user?id=
```

e mídia por `http(s)` ou determinados `tg://...`.

O backend, entretanto, não possui allowlist equivalente; sua única rejeição específica é `javascript:` em `href|src|url`.

**Impacto:** existem duas definições independentes de segurança/conformidade. Payload enviado diretamente ao backend pode contornar as restrições do editor.

**Correção:** uma implementação server-side canônica e compartilhamento de fixtures de conformidade.

## WD-018 — MÉDIO — `collage` e `slideshow` descartam propriedades de mídia além de src/alt

A representação dos itens é:

```js
{src, alt}
```

Qualquer propriedade adicional válida da mídia que exista no Rich HTML de origem não é representável. A importação também só consulta filhos `img` diretos.

**Impacto:** round-trip não é uma bijeção para a gramática rica; recursos aceitos pelo Telegram podem desaparecer.

**Correção:** declarar formalmente o subconjunto suportado ou expandir o modelo; testes de round-trip por feature.

## WD-019 — MÉDIO — teste de round-trip cobre apenas três tipos simples de mídia

**Arquivo:** `e2e/editor.spec.mjs`

Existe teste de estabilidade para `image`, `video` e `document`, mas faltam equivalentes para:

- details;
- tables com rowspan/colspan;
- collage;
- slideshow;
- map;
- button rows;
- custom emoji;
- time;
- math;
- references/anchors;
- listas ordenadas com atributos.

**Impacto:** justamente as estruturas com attrs complexos permanecem sem prova E2E de `model → HTML → model`.

**Correção:** matriz parametrizada de round-trip e casos negativos.

## WD-020 — MÉDIO — migração schema 1 não produz relatório de perda

**Arquivos:** `docs/document.js`, `client/editor.mjs`

`normalize()` registra:

```js
migration={fromSchema,at,original}
```

mas não registra quais elementos/atributos foram descartados durante `migrateHtml`. Como o parser é normalizador permissivo, a existência do `original` ajuda recuperação manual, porém o usuário/sistema não recebe sinal de que houve perda.

**Impacto:** migração pode ser apresentada como concluída com sucesso mesmo quando não é semanticamente lossless.

**Correção:** migration diagnostics (`warnings`, `droppedNodes`, `droppedAttrs`, `lossy:true`) e gate explícito para export/import.

## Conclusão do lote

A arquitetura semântica é estruturalmente superior a snapshots de DOM, mas o repositório atualmente mistura três propriedades distintas sob a palavra “válido”:

1. validade ProseMirror;
2. segurança básica de HTML/URL;
3. conformidade Telegram Rich HTML.

Essas propriedades não são equivalentes. A auditoria encontrou múltiplos casos em que (1) passa e (3) pode falhar, além de casos em que importação converte conteúdo sem preservar a estrutura.

**Acumulado:** 20 achados registrados nos lotes 001–002.
