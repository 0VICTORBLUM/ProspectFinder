# Prospect Finder

Plataforma interna de prospecção B2B para encontrar empresas que possam comprar, revender ou aplicar produtos de **impermeabilização** (foco de 70%) e **construção a seco** (foco de 30%). Começa por PR, SC e SP e aceita qualquer estado.

Stack: Next.js 15 (App Router), React 19, TypeScript estrito, Tailwind CSS 4, Route Handlers, Google Sheets API (conta de serviço), Vitest.

> **Estado atual:** funcional e testado com dados fictícios e com a camada do Google Sheets simulada. **Ainda não foi validado** com os arquivos reais da Receita Federal nem com uma planilha real (veja [Pendências](#pendências-que-exigem-sua-ação)). Não trate como pronto para produção antes de fechar essa lista.

## O que a aplicação faz

- **Pesquisa**: escolha estado (ou Brasil inteiro), cidades, segmento (manual ou estratégia 70/30 com percentuais editáveis), palavras-chave, CNAEs extras, situação cadastral, tipo de estabelecimento, limite e fontes. A pesquisa roda em lotes curtos, com progresso, cancelamento e contadores (verificadas, encontradas, salvas, duplicadas, descartadas).
- **Leads**: tabela com busca por nome/CNPJ, filtros (segmento, estado, cidade, status), ordenação, paginação, seleção, detalhe com a explicação da pontuação, edição de status e observações, exportação CSV e link para a fonte original.
- **Pontuação 0–100 determinística** e explicável (CNAE, nome, evidência em site confirmado, região, website confirmado, telefone, canal). Dado ausente aparece como *desconhecido* e não pontua. Pesos em `src/lib/scoring.ts`.
- **Transparência**: cada pesquisa é marcada como dados reais, demonstração ou mista; fontes indisponíveis e armazenamento só em memória geram avisos explícitos; a interface nunca informa "salvo no Sheets" quando não foi.

## Arquitetura

```
src/
  app/                 páginas (/pesquisa, /leads, /login) e Route Handlers (/api/...)
  components/          shell (navegação) e componentes de UI
  lib/
    types.ts           Lead, SearchRequest, SearchResult, SearchStatus, DataSource, QualificationResult
    normalize.ts       CNPJ, telefone, nome, UF, cidade, CNAE, website
    segments.ts        catálogo de CNAEs e palavras-chave por segmento (configurável)
    scoring.ts         qualificação determinística
    leads.ts           construção, deduplicação e mesclagem de leads
    distribution.ts    divisão 70/30 (maior resto) e relatório de metas
    search.ts          orquestração por lotes (criar, executar um lote, cancelar)
    connectors/        um conector por fonte (cnpj-local, demo); registre novos em index.ts
    enrichment.ts      enriquecimento opcional por site oficial (robots.txt, anti-SSRF)
    ssrf.ts            validação de URL pública e fetch seguro
    store/             memory.ts (volátil) e sheets.ts (Google Sheets, com retentativa)
    csv.ts             CSV com proteção contra injeção de fórmulas
    auth.ts, api.ts    token de acesso, limite de requisições, validação
  middleware.ts        exige autenticação em páginas e APIs
scripts/               importador dos Dados Abertos do CNPJ
tests/                 79 testes (Vitest)
```

### Como a pesquisa roda sem estourar o limite serverless

`POST /api/searches` cria a pesquisa. O navegador chama `POST /api/searches/{id}/step` repetidamente; cada chamada processa **um lote** dentro de `SEARCH_STEP_BUDGET_MS` (15 s por padrão) e devolve o estado. O estado fica no armazenamento (aba `Pesquisas` do Sheets), então qualquer instância serverless pode continuar de onde parou. Não há temporizadores simulando resultado.

## Fontes de dados

| Fonte | Situação |
|---|---|
| **Dados Abertos do CNPJ** (Receita Federal) | Implementada via arquivo local filtrado (`data/cnpj/empresas.jsonl`). Não é lida na requisição: você a gera com `npm run import:cnpj`. |
| **Sites oficiais** (enriquecimento) | Implementado, opcional. Respeita `robots.txt`, não contorna bloqueios, bloqueia IPs internos, limita tempo e tamanho. O site deduzido do domínio do e-mail corporativo é marcado como **inferido** e só vira **verificado** se a página citar o CNPJ ou o nome da empresa. |
| **Demonstração** | Empresas **fictícias**, sempre rotuladas. Serve só para testar a interface. |
| **Google Maps Platform** | **Não implementado**, de propósito. Exige chave, cobrança e revisão dos termos de armazenamento. Não há scraping do Maps. Para adicionar, crie um conector em `src/lib/connectors/` e deixe-o desabilitado até haver configuração. |

### Gerar o arquivo do CNPJ

1. Baixe os arquivos de **Estabelecimentos**, **Empresas** e **Municípios** em https://dados.gov.br/ (conjunto "Cadastro Nacional da Pessoa Jurídica – CNPJ") e descompacte.
2. Rode:

```bash
npm run import:cnpj -- \
  --estabelecimentos "dados/Estabelecimentos*.csv" \
  --empresas "dados/Empresas*.csv" \
  --municipios dados/Municipios.csv \
  --uf PR,SC,SP \
  --cnaes 4330401,4330403,4330402,4330499,4120400,4399101,4399199,4679699,4744099
```

O script processa em streaming, mantém só campos de prospecção e **descarta o e-mail de empresário individual** (tende a ser pessoal). Na Vercel o arquivo não existe por padrão: use a importação localmente, ou hospede o app onde o arquivo possa ficar em disco.

> O layout de colunas do importador foi escrito a partir da documentação pública e testado só com linhas sintéticas. **Confira-o contra os arquivos atuais** (a Receita pode alterá-lo) antes de confiar nos resultados.

### Catálogo de CNAEs

O CNAE **4330-4/01** (impermeabilização) é o ponto de partida pedido. Os demais códigos de `src/lib/segments.ts` são sugestões **não validadas** contra a tabela oficial do IBGE/CONCLA (https://concla.ibge.gov.br/). A interface exibe esse aviso (`CNAE_CATALOG_VALIDATED = false`). Valide e ajuste antes de usar como critério definitivo.

## Requisitos e instalação

- Node.js 20 ou superior (testado no 22) e npm.

```bash
npm install
cp .env.example .env.local   # preencha os valores (veja abaixo)
npm run dev                  # http://localhost:3000
npm test                     # testes
npm run typecheck
npm run build
```

Sem `APP_ACCESS_TOKEN`, o app abre sem login **apenas em desenvolvimento**. Em produção ele bloqueia tudo até o token ser definido.

## Variáveis de ambiente

| Variável | Uso |
|---|---|
| `APP_ACCESS_TOKEN` | Token de acesso (login e `Authorization: Bearer`). Obrigatório em produção. |
| `GOOGLE_SHEETS_ID` | ID da planilha (trecho entre `/d/` e `/edit` na URL). |
| `GOOGLE_SERVICE_ACCOUNT_EMAIL` + `GOOGLE_PRIVATE_KEY` | Credenciais da conta de serviço (mantenha os `\n` literais na chave). |
| `GOOGLE_SERVICE_ACCOUNT_JSON` | Alternativa: JSON completo da conta de serviço em uma linha. |
| `CNPJ_DATA_FILE` | Caminho do JSONL gerado (padrão `data/cnpj/empresas.jsonl`). |
| `SEARCH_STEP_BUDGET_MS`, `SEARCH_MAX_LIMIT` | Tempo por lote e limite máximo de resultados. |
| `ENRICHMENT_TIMEOUT_MS`, `ENRICHMENT_USER_AGENT` | Enriquecimento: timeout e identificação do robô (inclua um contato). |
| `ENABLE_DEMO_SOURCE` | `false` desabilita a fonte fictícia. |

## Configurar o Google Sheets

1. No Google Cloud, crie um projeto e **ative a Google Sheets API**.
2. Crie uma **conta de serviço** (sem papéis de projeto) e gere uma chave JSON.
3. Crie uma planilha vazia e **compartilhe-a com o e-mail da conta de serviço** com permissão de *Editor* (é o mínimo necessário; nenhum outro acesso é concedido).
4. Preencha `GOOGLE_SHEETS_ID` e as credenciais no `.env.local`.
5. Na primeira gravação o app cria as abas `Leads` e `Pesquisas` com cabeçalhos.

Detalhes que importam:
- A coluna `dados_json` é a fonte da verdade. **Edite status e observações pelo app**; alterações manuais nas outras colunas não voltam para o app.
- Os valores são gravados como texto (`RAW`), então nada vira fórmula na planilha.
- Cada lead tem id estável (`cnpj-<14 dígitos>`), então regravar é **idempotente**: se uma gravação falhar no meio, refazer a pesquisa não duplica linhas.
- A planilha não é banco de alta concorrência: serve bem para centenas de leads por mês. A leitura é por lista completa; para dezenas de milhares de linhas, migre para um banco (a interface `LeadStore` em `src/lib/store/types.ts` isola isso).
- Falhas 429/5xx e de rede têm até 5 tentativas com espera crescente; 4xx (como 403 sem permissão) aparecem como erro explícito.

## Deploy na Vercel

1. Importe o repositório e defina as variáveis de ambiente acima (nunca versione `.env.local`).
2. `maxDuration` do lote está em 30 s; mantenha `SEARCH_STEP_BUDGET_MS` abaixo do limite do seu plano.
3. O arquivo do CNPJ não acompanha o deploy (está no `.gitignore`). Sem ele, a fonte aparece como indisponível, e a pesquisa real não funciona na Vercel até você resolver onde hospedá-lo.
4. O limite de requisições é em memória e vale por instância serverless. Para proteção forte, use um serviço externo (por exemplo, regras da Vercel ou um Redis).

## Segurança e LGPD

- Credenciais ficam só no servidor; nenhum segredo vai ao cliente.
- Autenticação por token em páginas e APIs (cookie `httpOnly`, `SameSite=Strict`, hash do token), com falha fechada em produção.
- Entradas validadas com Zod; tamanho de corpo, limites de resultados e de requisições definidos.
- O navegador **não escolhe URLs**: o servidor só consulta sites que vieram das fontes, com bloqueio de IPs privados/internos, portas fora de 80/443, credenciais na URL e revalidação a cada redirecionamento. Limitação conhecida: a checagem de DNS não fixa o IP da conexão, então um ataque de *DNS rebinding* sofisticado não é totalmente descartado.
- Apenas `status` e `observações` podem ser editados. Não há ações destrutivas em massa.
- CSV: valores iniciados por `= + - @` (e tab/CR) recebem apóstrofo; separador `;` e BOM para o Excel em português.
- LGPD: coleta mínima (dados cadastrais de pessoa jurídica, telefone/e-mail comerciais), origem registrada por campo, e-mail de empresário individual descartado na importação. Mantenha uma base legal (legítimo interesse) documentada e ofereça canal de descadastro ao contatar.
- Dependências: `npm audit` ainda aponta `postcss` embutido no Next (afeta o processamento de CSS no build, não requisições) e `uuid` via `google-auth-library` (função não usada aqui). A correção automática exigiria `--force`; reavalie quando houver nova versão do Next.

## Testes

`npm test` roda 79 testes: validação, normalização de CNPJ/telefone, deduplicação, pontuação, distribuição 70/30, paginação, filtros, CSV seguro, conector do CNPJ (com arquivo fictício), importador (linhas sintéticas), SSRF e robots.txt, enriquecimento, Google Sheets simulado (retentativas, idempotência, falha parcial, 403), orquestração da pesquisa (fonte fora do ar, queda no meio, falha do Sheets, cancelamento) e permissões dos endpoints. Todos os dados de teste são fictícios e ficam em `tests/`.

Não coberto por teste automatizado: renderização no navegador (a interface foi verificada apenas por build e por chamadas HTTP ao servidor de produção) e integração com a API real do Google.

## Solução de problemas

- **"Fonte indisponível" no CNPJ**: gere o arquivo (`npm run import:cnpj`) e confira `CNPJ_DATA_FILE`.
- **"Não autorizado"/redireciona para /login**: defina `APP_ACCESS_TOKEN` e entre com ele.
- **Erro 403 do Sheets**: a planilha não foi compartilhada com o e-mail da conta de serviço, ou a API não está ativada.
- **Erro 404 do Sheets**: `GOOGLE_SHEETS_ID` incorreto.
- **Chave privada inválida**: mantenha os `\n` literais em `GOOGLE_PRIVATE_KEY` ou use `GOOGLE_SERVICE_ACCOUNT_JSON`.
- **Leads sumiram após reiniciar**: o app estava em modo memória (Sheets não configurado).

## Custos potenciais

Next.js, Vercel (plano gratuito para uso leve), Google Sheets API (gratuita dentro da cota) e os Dados Abertos do CNPJ não têm custo direto. Google Maps Platform, se um dia integrado, é cobrado por uso.

## Pendências que exigem sua ação

1. Baixar os arquivos reais do CNPJ, rodar o importador e **validar o layout e o resultado**.
2. **Validar os CNAEs** do catálogo na tabela oficial do IBGE/CONCLA.
3. Criar a planilha, a conta de serviço e testar a leitura/gravação **reais** no Sheets.
4. Definir `APP_ACCESS_TOKEN` e decidir onde hospedar o arquivo do CNPJ.
5. Testar a interface no navegador (desktop e celular).
