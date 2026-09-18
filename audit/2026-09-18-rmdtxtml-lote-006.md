# Auditoria técnica RMDtxtML — lote 006

**Data:** 2026-09-18  
**Alvo:** `romastefale/RMDtxtML`, branch `draft`  
**Árvore-base auditada:** `692995e8fcdf5bac250605cdda20c23a0f1f729b`  
**Escopo:** roteamento HTTP, release gate, container, supply chain e força probatória dos testes.

## WD-051 — MÉDIO/ALTO — rotas GET/HEAD desconhecidas sob `/api/` caem no fallback SPA e retornam HTML

**Arquivo:** `src/server.mjs`

O router só trata explicitamente `GET /api/health`. Qualquer outro GET/HEAD segue para `serveStatic()`. Quando o pathname não tem extensão e o arquivo não existe, `serveStatic()` reescreve a requisição para `/index.html`.

Consequências:

- `GET /api/send` pode retornar a aplicação HTML com status 200, em vez de 405;
- `GET /api/bootstrap` idem;
- `GET /api/qualquer-coisa` pode parecer sucesso HTTP.

**Impacto:** mascara erro de integração, confunde health/monitoring/proxies e quebra a propriedade de que namespace `/api/*` sempre devolve contrato JSON.

**Correção:** separar o namespace API antes do fallback SPA; para `/api/*`, responder JSON 404/405 explicitamente.

## WD-052 — MÉDIO — health endpoint usa prefix match em vez de pathname exato

O código testa:

`req.url?.startsWith('/api/health')`

Assim, caminhos como `/api/health-anything` ou `/api/healthXYZ` também são tratados como health.

**Impacto:** roteamento ambíguo e possibilidade de falsos positivos em probes/configurações.

**Correção:** parsear `new URL(req.url, base).pathname` e comparar exatamente `/api/health`.

## WD-053 — MÉDIO — servidor não valida `Content-Type` dos endpoints JSON

`bodyJson()` lê bytes e executa `JSON.parse` sem exigir `application/json`.

**Impacto:** amplia a superfície de formatos aceitos e reduz previsibilidade do contrato HTTP. Em conjunto com controles de Origin incompletos, torna a fronteira menos explícita.

**Correção:** exigir media type JSON nos endpoints mutáveis e retornar 415 para tipos incompatíveis.

## WD-054 — MÉDIO — shell de produção não possui Content-Security-Policy

**Arquivos:** `src/server.mjs`, `docs/index.html`

As respostas estáticas incluem `nosniff`, `Referrer-Policy` e `Permissions-Policy`, mas não CSP. Ao mesmo tempo, o shell carrega JavaScript remoto de `https://telegram.org/js/telegram-web-app.js`.

**Impacto:** em caso de futura injeção de HTML/script em qualquer superfície, não há uma segunda barreira de browser restringindo script/style/connect/frame origins.

**Correção:** CSP explícita compatível com Telegram Mini Apps, começando por `default-src 'self'` e allowlists mínimas para scripts/conexões necessários.

## WD-055 — MÉDIO — imagem final executa como root

**Arquivo:** `Dockerfile`

A imagem `node:24.21.0-alpine` possui usuário não privilegiado disponível, mas o Dockerfile não declara `USER`.

**Impacto:** uma eventual execução arbitrária dentro do processo Node herda privilégios de root no namespace do container e aumenta o blast radius, especialmente sobre filesystem/volume montado.

**Correção:** preparar diretórios/volume e executar a aplicação com UID/GID não privilegiados.

## WD-056 — MÉDIO — workflow chamado “Release Gate” não cobre PR, main ou tags

**Arquivo:** `.github/workflows/release-gate.yml`

Triggers existentes:

- `push` apenas em `draft`;
- `workflow_dispatch`.

Não há trigger `pull_request`, push em `main` ou release/tag.

O README chama o gate de “obrigatório”, mas o arquivo de workflow por si só não estabelece essa obrigatoriedade fora de pushes em `draft`; branch protection externa não está versionada aqui.

**Impacto:** alterações podem chegar a outras refs sem que este workflow específico seja disparado automaticamente, dependendo da política externa.

**Correção:** alinhar triggers à estratégia real de merge/release e documentar branch protection como requisito verificável.

## WD-057 — MÉDIO — componentes críticos da supply chain são referenciados por tags mutáveis

**Arquivos:** `.github/workflows/release-gate.yml`, `Dockerfile`

Exemplos:

- `actions/checkout@v4`;
- `mcr.microsoft.com/playwright:v1.63.0-noble`;
- `node:24.21.0-alpine`.

Tags facilitam manutenção, mas não fixam conteúdo criptograficamente. Junto com a ausência de lockfile já registrada em WD-008, duas builds do mesmo commit não são garantidas byte-a-byte equivalentes.

**Correção:** pinar GitHub Actions por commit SHA e imagens base por digest, com processo explícito de atualização.

## WD-058 — MÉDIO — parte do gate testa presença textual de código, não comportamento

**Arquivos:** `test/editor.test.mjs`, `test/release.test.mjs`, `test/ui.test.mjs`

Vários testes usam `assert.match`, `includes` ou regex sobre arquivos fonte para afirmar propriedades arquiteturais, por exemplo verificar que nomes de nodes aparecem no schema ou que certas strings aparecem no servidor.

Esses testes podem permanecer verdes mesmo quando o comportamento real está incorreto. Os lotes anteriores demonstram isso: nodes e validators estão presentes, mas diversas invariantes 10.3 estão erradas.

**Impacto:** o número de testes e um gate verde superestimam a cobertura semântica.

**Correção:** manter checks estruturais apenas como complemento; migrar garantias importantes para testes executáveis com entrada/saída e propriedades.

## WD-059 — MÉDIO — E2E “iPhone” roda Chromium, não WebKit

**Arquivo:** `playwright.config.mjs`

O único projeto combina `devices['iPhone 15']` com `browserName:'chromium'`.

Isso emula viewport/device parameters, mas não o engine WebKit usado no ecossistema iOS. Também não há projeto Android/WebView separado.

**Impacto:** um único Chromium mobile não cobre diferenças relevantes de contenteditable, selection, IndexedDB, lifecycle e viewport entre engines — exatamente áreas centrais deste editor.

**Correção:** matriz mínima Chromium + WebKit; adicionar perfil Android se a compatibilidade Telegram Android fizer parte do contrato.

## WD-060 — ALTO — não existe suíte de conformidade derivada do contrato Bot API 10.3

O release gate executa testes unitários e Playwright, mas não existe fixture/matriz que enumere a gramática Rich Message oficial e prove:

- parse de cada elemento/atributo suportado;
- rejeição de combinações inválidas;
- `model → HTML → model` sem perda para o subconjunto declarado;
- equivalência entre validator cliente e servidor;
- limites oficiais.

Os achados WD-021–WD-050 coexistem com gate verde precisamente porque a suíte não trata a especificação Telegram como contrato executável.

**Impacto:** regressões/conformidade ficam dependentes de revisão manual.

**Correção:** fixtures versionadas por Bot API, property tests e casos negativos por invariante.

## Síntese do lote

O pipeline executa build + testes de verdade, o que é positivo, mas sua cobertura não justifica sozinho a afirmação de compatibilidade 10.3. Há diferença entre **gate executado** e **contrato provado**. Também há pontos de endurecimento de runtime e supply chain ainda ausentes.

**Acumulado:** 60 achados registrados nos lotes 001–006.