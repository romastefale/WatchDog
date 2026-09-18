# Auditoria técnica RMDtxtML — lote 001

**Data:** 2026-09-18  
**Alvo:** `romastefale/RMDtxtML`, branch `draft`  
**Árvore auditada:** `692995e8fcdf5bac250605cdda20c23a0f1f729b`  
**Escopo deste lote:** backend HTTP, validação Telegram Rich HTML, autorização de destinos, persistência, transferência Web→Telegram, gate de release e divergências com Telegram Bot API 10.3.

## Sumário executivo

Foram identificadas divergências materiais entre as garantias declaradas no README, a implementação e o contrato atual da Telegram Bot API 10.3. As mais relevantes são: (1) o validador de Rich HTML não valida a gramática suportada pelo Telegram e aceita tags/atributos arbitrários; (2) a autorização de destinos configurados é global a qualquer usuário Telegram com `initData` válido, sem ACL por usuário; (3) o rate limiting usa cabeçalhos de IP não autenticados; (4) o endpoint de criação de transferências aceita requisições sem `Origin`; (5) a validação semântica do servidor só verifica o envelope e tamanho JSON, não o schema ProseMirror alegado; (6) o gate de release não executa auditoria de dependências e usa `npm install` sem lockfile versionado.

## WD-001 — CRÍTICO — “Rich HTML validado” não é verdade no boundary do servidor

**Arquivo:** `src/telegram.mjs`  
**Função:** `validateRichHtml`

O validador não implementa allowlist da gramática Rich HTML 10.3. Ele rejeita apenas algumas tags perigosas (`script`, `iframe`, `object`, `embed`, `style`, `link`, `meta`), atributos `on*` e URLs `javascript:`. Todo o restante é aceito.

Exemplos aceitos pelo servidor apesar de não pertencerem à lista de tags suportadas pelo Rich HTML do Telegram:

```html
<form action="https://example.invalid"><button>bad</button></form>
<svg><foreignObject>x</foreignObject></svg>
<marquee>x</marquee>
<a href="data:text/html,foo">x</a>
```

A documentação oficial da Bot API 10.3 declara explicitamente que **somente as tags listadas** são suportadas. Portanto, o servidor não “valida Rich HTML”; ele aplica um filtro parcial de segurança antes de delegar a validação real ao Telegram.

**Impacto:** conteúdo que passa pelo gate local pode falhar apenas em produção na chamada `sendRichMessage`. Isso contradiz README (“Rich HTML é validado novamente no backend antes de publicação”) e reduz o valor do gate de QA.

**Correção:** parser HTML real + allowlist exata de tags/atributos/contextos da Bot API 10.3; rejeitar tags desconhecidas; validar esquemas de URL por atributo; validar regras contextuais (`figcaption`, tabelas, mídia, botões, details etc.).

## WD-002 — ALTO — parser estrutural aceita HTML malformado e calcula limites sobre uma árvore inexistente

**Arquivo:** `src/telegram.mjs`  
**Função:** `richStructure`

O fechamento de tags procura a tag em qualquer posição da pilha e executa `stack.length=i`. Isso não exige fechamento LIFO e não sinaliza tags órfãs ou estruturas cruzadas.

Exemplo:

```html
<p><strong>x</p></strong>
```

A função não rejeita a estrutura. Também não rejeita tags de fechamento sem abertura correspondente.

Além disso, contagem de blocos é baseada em regex de abertura, não na árvore parseada segundo as regras do Telegram. Assim, limites de 500 blocos e profundidade 16 podem divergir da interpretação real da API.

**Impacto:** falso positivo de validação, comportamento inconsistente e testes que verificam apenas contagens sintéticas.

**Correção:** abandonar parser por regex; construir AST e validar balanceamento/contexto.

## WD-003 — ALTO — destinos configurados são autorizados globalmente para qualquer usuário Telegram autenticado

**Arquivo:** `src/telegram.mjs`  
**Funções:** `authorizedDestinations`, `publicDestinations`, `resolveDestination`

`AUTHORIZED_DESTINATIONS` e `ALLOWED_CHAT_IDS` são adicionados à lista de destinos de todo usuário que apresente `initData` válido. Não existe associação `user_id → destination`, papel administrativo, tenant, membership check ou ACL.

O uso de identificadores opacos impede exposição direta de `chat_id`, mas **não constitui autorização**: o próprio endpoint `/api/bootstrap` entrega o ID opaco a qualquer sessão Telegram válida e `/api/send` o resolve para o chat real.

**Impacto:** se um canal/grupo privilegiado for configurado globalmente, qualquer usuário da Mini App pode enviar Rich Messages para esse destino através do bot, sujeito apenas às permissões do próprio bot.

**Correção:** ACL explícita por `user.id`, grupo de usuários ou policy server-side; não entregar destinos privilegiados no bootstrap de usuários não autorizados. Adicionar testes negativos com dois usuários.

## WD-004 — ALTO — rate limiting pode depender de `X-Forwarded-For` fornecido pelo cliente

**Arquivo:** `src/server.mjs`  
**Função:** `clientIp`

A ordem atual é:

```js
cf-connecting-ip || x-forwarded-for || socket.remoteAddress
```

Não há validação de proxy confiável. Em uma implantação onde o edge/proxy não sobrescreva esses cabeçalhos, o cliente pode variar `X-Forwarded-For` e criar buckets arbitrários.

**Impacto:** bypass dos limites de `/api/transfers` e `/api/transfers/claim`; potencial abuso de SQLite, Bot username lookup e armazenamento de transferências.

**Correção:** confiar em forwarding headers somente quando `remoteAddress` pertence ao proxy/edge esperado; caso contrário usar socket IP. Em Railway, documentar e testar a semântica efetiva do proxy.

## WD-005 — MÉDIO/ALTO — ausência de `Origin` é tratada como origem confiável

**Arquivo:** `src/server.mjs`  
**Função:** `trustedOrigin`

```js
return !value || origins(env).has(value)
```

Logo, qualquer request sem `Origin` passa pelo controle. Para endpoints autenticados por `initData`, o segredo de sessão ainda é necessário; porém `POST /api/transfers` é deliberadamente não autenticado e aceita requests sem Origin.

**Impacto:** o controle descrito como “CORS/origin checks” não é um boundary de autorização. Clientes não-browser podem contorná-lo por construção.

**Correção:** separar CORS de autenticação/abuse-control. Se criação de transferência deve ser exclusiva do frontend oficial, exigir token/session server-side ou Origin obrigatório com política documentada. Se deve ser API pública, remover a falsa garantia do README e proteger por autenticação/rate limiting robusto.

## WD-006 — ALTO — servidor aceita “semantic schema 2” sem validar o modelo semântico

**Arquivo:** `src/server.mjs`  
**Função:** `transferSemantic`

A função exige apenas:

- `schema === 2`;
- `format === 'semantic'`;
- `model` ser objeto;
- `modelVersion` numérico 1..1000;
- JSON serializado ≤ 300000 caracteres.

Ela não valida tipos de nodes, attrs, marks, profundidade, URLs, conteúdo, nem conformidade com o schema ProseMirror usado pelo cliente. O teste `web transfer preserves model version` inclusive aceita `modelVersion: 7`, enquanto `docs/document.js` rejeita qualquer versão diferente de `1`.

**Divergência objetiva:** backend declara compatibilidade 1..1000; frontend declara apenas `MODEL_VERSION=1`.

**Impacto:** payload armazenado como “semântico válido” pode tornar-se impossível de adotar no cliente (`unsupported_model_version`) ou causar falha de normalização. A validação está no lado errado da fronteira de confiança.

**Correção:** compartilhar schema/version validator entre cliente e servidor; aceitar somente versões implementadas; validar modelo antes de persistir.

## WD-007 — MÉDIO — limite de texto não mede “UTF-8 characters” conforme a especificação

**Arquivo:** `src/telegram.mjs`  
**Função:** `richTextLength`

A implementação remove tags via regex e conta code points JavaScript (`[...string].length`). Entidades HTML são substituídas por um único `x`. Isso não equivale necessariamente à contagem que o Telegram aplica depois de parsear Rich HTML, especialmente em sequências Unicode, entidades numéricas e texto presente em atributos/estruturas especiais.

O próprio teste assume que `😀`.repeat(32768) deve ser aceito. A documentação oficial formula o limite como “32768 UTF-8 characters”, incluindo texto alternativo de custom emoji e fonte de fórmula.

**Impacto:** divergências de borda no limite podem ser descobertas somente pela API.

**Correção:** reproduzir a regra de contagem oficial com casos de conformidade contra a API ou, no mínimo, aplicar margem conservadora e testar entidades/custom emoji/fórmulas.

## WD-008 — MÉDIO — gate “reprodutível” sem lockfile e sem `npm ci`

**Arquivos:** `package.json`, `Dockerfile`, árvore do repositório

Não há `package-lock.json` na árvore auditada. O Docker executa:

```dockerfile
RUN npm install --ignore-scripts --no-audit --no-fund
```

Embora as dependências diretas estejam pinadas, dependências transitivas podem variar entre builds. Além disso, `--no-audit` elimina uma classe de detecção do gate.

**Impacto:** duas imagens construídas do mesmo commit podem resolver árvores transitivas diferentes; regressões ou advisories podem entrar sem alteração no repositório.

**Correção:** versionar lockfile, usar `npm ci`, manter cache/registry policy e adicionar auditoria/SBOM conforme o nível de release.

## WD-009 — MÉDIO — healthcheck pode reportar “persistent: true” sem verificar durabilidade real

**Arquivo:** `src/store.mjs`

`persistent` é definido apenas por:

```js
Boolean(RAILWAY_VOLUME_MOUNT_PATH) && !dbPath
```

Não há verificação de mountpoint, filesystem, escrita persistente, fsync, espaço livre ou recuperação após restart. A flag significa “variável de ambiente presente”, não “persistência comprovada”.

**Impacto:** observabilidade pode reportar garantia mais forte que a evidência disponível.

**Correção:** renomear para `configuredPersistentPath` ou implementar probe que verifique o mount esperado e exponha separadamente configuração e estado.

## WD-010 — MÉDIO — testes não cobrem a principal divergência de autorização

Os testes confirmam que IDs de chat não vazam e que o ID opaco resolve, mas não testam isolamento entre usuários. O teste atual:

```js
authorizedDestinations({user:{id:42}}, env, token)
```

só verifica quantidade e opacidade. Falta provar que um usuário não autorizado não recebe destinos configurados.

**Impacto:** a suíte cristaliza “opacidade = segurança” sem testar a propriedade de autorização.

**Correção:** testes multiusuário e policy fixtures.

## Divergências documentais

1. **README:** “Rich HTML é validado novamente no backend” — implementação faz filtro parcial, não validação de conformidade Bot API.
2. **README:** “modelo semântico valida estrutura e atributos antes de renderizar” — isso pode ser verdade no cliente, mas o backend aceita envelopes sem validar o modelo.
3. **README:** “CORS/origin checks” — requests sem Origin são confiáveis; não é boundary completo.
4. **README/package:** “versões fixadas” — apenas dependências diretas estão pinadas; sem lockfile, resolução transitiva não está fixada.
5. **Backend/frontend:** backend aceita `modelVersion` 1..1000; frontend aceita somente 1.

## Referência externa usada para conformidade

Telegram Bot API 10.3, seção Rich Messages / Rich HTML / InputRichMessage / sendRichMessage, consultada em 2026-09-18. A especificação atual limita Rich HTML às tags documentadas e define limites de 32768 caracteres, 500 blocos, 16 níveis, 50 mídias e 20 colunas.

## Próximo lote

Auditar integralmente `client/editor.mjs`, serialização HTML↔modelo, `docs/app.js`, migração schema 1→2, import/export, XSS/DOM parsing, semântica de listas/tabelas/mídia/botões 10.3, idempotência sob concorrência, SQLite/WAL e cobertura E2E.
