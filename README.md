# Atividade Individual 1 — Serverless (Rota Vital)

- **Disciplina:** Infraestrutura de Software · ADS · CESAR School · 2026.2
- **Caminho escolhido:** **Caminho A** — Execução local com **AWS SAM CLI (`1.174.0`)** + **Docker Desktop (`29.6.2`)** (runtime **Python 3.12**, pacote **`zip`**)

## Estrutura do Projeto

```text
atv/
├── .gitignore
├── README.md
├── ROTEIRO_VIDEO.md
├── template.yaml            # Rota POST /guias, Runtime python3.12, Timeout: 3s (Passos 3-5a) -> 10s (Passo 5b/final)
├── events/
│   └── pedido.json          # Evento API Gateway aws-proxy POST /guias (R-001)
├── hello_world/
│   ├── __init__.py
│   ├── app.py               # Handler da função de geração de guia de transporte
│   └── requirements.txt
└── evidencias/
    ├── LEIAME.txt
    ├── 01-ambiente-sam-build.txt / .png
    ├── 02-invoke-evento.txt / .png
    ├── 03-start-api-sem-warm.txt / .png
    ├── 04-start-api-warm-lazy.txt / .png
    └── 05-timeout-3s-vs-10s.txt / .png
```

---

## Como Executar Localmente (Caminho A)

> **Dicas para Windows (Git Bash vs PowerShell):**
> 1. Execute os comandos com redirecionamento (`> events/pedido.json`) e `curl` no **Git Bash** (`"C:\Program Files\Git\bin\bash.exe"`), pois o PowerShell 5.1 grava `>` em UTF-16LE (com BOM), remove aspas internas de JSON passadas para binários nativos e mapeia `curl` como alias de `Invoke-WebRequest`.
> 2. O instalador `Amazon.SAM-CLI` cria `C:\Program Files\Amazon\AWSSAMCLI\bin\sam.cmd`. No PowerShell, `sam` funciona diretamente; no Git Bash, se `sam` não for reconhecido diretamente, use `sam.cmd` ou defina `alias sam='sam.cmd'`.

### 1. Compilar o projeto SAM

```bash
sam build
```

### 2. Gerar evento e invocar localmente (Passo 3)

```bash
mkdir -p events
sam local generate-event apigateway aws-proxy --method POST --path guias --body '{"requisicaoId":"R-001","hospital":"H-Recife","bolsas":["O-NEG-0142"]}' > events/pedido.json
```

> **Observação sobre o SAM CLI `1.174.0`:** o comando `sam local generate-event apigateway aws-proxy` gera o campo `"body"` codificado em Base64 com `"isBase64Encoded": true`. Como `hello_world/app.py` (mantido idêntico ao enunciado) faz `json.loads(event.get("body") or "{}")` diretamente, o arquivo `events/pedido.json` deste repositório teve o `body` decodificado para string JSON pura e `"isBase64Encoded": false` para permitir a invocação direta sem erro `JSONDecodeError`.

Em seguida, execute duas vezes:

```bash
sam local invoke -e events/pedido.json
sam local invoke -e events/pedido.json
```

### 3. Gatilho HTTP local sem e com `--warm-containers LAZY` (Passo 4)

Em um terminal, suba a API sem reutilização de contêiner:

```bash
sam local start-api
```

Em outro terminal (Git Bash), envie 3 requisições observando `invocacaoNesteAmbiente`:

```bash
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
```

Encerre o servidor (`Ctrl+C`), reinicie com `--warm-containers LAZY` e repita as 3 chamadas:

```bash
sam local start-api --warm-containers LAZY
```

### 4. Experimento de Timeout (Passo 5)

Com `Globals.Function.Timeout: 3` no `template.yaml` (e compilado com `sam build`), envie:

```bash
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-003","demorar":true}'
```

Registre o erro de tempo limite (`timed out after 3 seconds` / HTTP `502`). Depois, altere `Timeout: 3` para `Timeout: 10` em `template.yaml` (estado mantido atualmente no repositório), recompile e reinicie a API:

```bash
sam build
sam local start-api
```

Repita a requisição com `"demorar":true` e confirme o retorno HTTP `201`.

---

## Evidências

Todas as execuções reais foram registradas na pasta [`evidencias/`](./evidencias/) em texto integral (`.txt`) e capturas de terminal em alta legibilidade (`.png`):

| Etapa | Log real (`.txt`) | Captura (`.png`) | Resumo observado |
|---|---|---|---|
| Ambiente & Build | [`01-ambiente-sam-build.txt`](./evidencias/01-ambiente-sam-build.txt) | [`01-ambiente-sam-build.png`](./evidencias/01-ambiente-sam-build.png) | `SAM CLI 1.174.0`, `Docker 29.6.2`, `sam validate` OK, `sam build` Succeeded |
| Passo 3 (`invoke` 2x) | [`02-invoke-evento.txt`](./evidencias/02-invoke-evento.txt) | [`02-invoke-evento.png`](./evidencias/02-invoke-evento.png) | Exec. 1: `113.36 ms` (`11.86 s` total); Exec. 2: `117.73 ms` (`11.95 s` total); `201`, `G-R-001`, `invocacaoNesteAmbiente: 1` |
| Passo 4a (sem warm) | [`03-start-api-sem-warm.txt`](./evidencias/03-start-api-sem-warm.txt) | [`03-start-api-sem-warm.png`](./evidencias/03-start-api-sem-warm.png) | 3x `POST /guias` -> `invocacaoNesteAmbiente` = `1, 1, 1`; `SAM_CONTAINER_ID` diferente em todas |
| Passo 4b (`--warm-containers LAZY`) | [`04-start-api-warm-lazy.txt`](./evidencias/04-start-api-warm-lazy.txt) | [`04-start-api-warm-lazy.png`](./evidencias/04-start-api-warm-lazy.png) | 3x `POST /guias` -> `invocacaoNesteAmbiente` = `1, 2, 3`; `113.98 ms` (cold) -> `2.31 ms` / `6.00 ms` (warm) |
| Passo 5 (`Timeout: 3` vs `10`) | [`05-timeout-3s-vs-10s.txt`](./evidencias/05-timeout-3s-vs-10s.txt) | [`05-timeout-3s-vs-10s.png`](./evidencias/05-timeout-3s-vs-10s.png) | `Timeout: 3` -> `timed out after 3 seconds` (`502`); `Timeout: 10` + `sam build` -> `201` (`5122.80 ms`) |

---

## Respostas às 5 Perguntas

### 1. Por que a primeira invocação demorou mais que a segunda e o que acontece na fase de Init?

Na primeira execução absoluta após o `sam build`, quando a imagem Docker do runtime ainda não está no cache local (`Local image was not found`), o SAM CLI precisa baixar/construir a imagem `public.ecr.aws/lambda/python:3.12-rapid-x86_64` antes de subir o contêiner (~45 s no primeiro download). Com a imagem já em cache ([`evidencias/02-invoke-evento.txt`](./evidencias/02-invoke-evento.txt)):

- **Execução 1 (`sam local invoke -e events/pedido.json`):** `SAM_CONTAINER_ID: 93393e39...` · `Init Duration: 0.07 ms` · `Duration: 113.36 ms` · `Billed Duration: 114 ms` · tempo total de relógio `11.86 s` (`11863 ms`) · `invocacaoNesteAmbiente: 1`.
- **Execução 2 (`sam local invoke -e events/pedido.json`):** `SAM_CONTAINER_ID: 08ef7a2b...` · `Init Duration: 0.08 ms` · `Duration: 117.73 ms` · `Billed Duration: 118 ms` · tempo total de relógio `11.95 s` (`11947 ms`) · `invocacaoNesteAmbiente: 1`.

**O que acontece na fase de Init e por que `invoke` vs `warm` diferem:**
1. Na **fase de Init** (*cold start*), o serviço provisiona o ambiente de execução isolado (no local, sobe um contêiner Docker com o runtime `python3.12`), inicializa o interpretador Python, carrega o pacote de código (`/var/task/app.py`) e executa todo o código de nível de módulo fora de `lambda_handler` — no nosso código: `import json, time` e `contador = 0`.
2. Como o comando `sam local invoke` cria e destrói um contêiner novo a cada chamada (`SAM_CONTAINER_ID` distinto nas duas execuções), **ambas** as invocações no Passo 3 passam por *cold start* (por isso ambas exibem `Init Duration` e tempos semelhantes de ~11,9 s no Docker local, depois que o pull inicial da imagem já foi feito na primeiríssima execução).
3. Já quando o ambiente é reaproveitado (*warm start*, demonstrado no Passo 4 com `--warm-containers LAZY` em [`evidencias/04-start-api-warm-lazy.txt`](./evidencias/04-start-api-warm-lazy.txt)), a fase de **Init** ocorre apenas na 1ª requisição (`Init Duration: 0.07 ms`, `Duration: 113.98 ms`) e é **pulada** nas seguintes, fazendo a duração cair para **`2.31 ms`** (2ª) e **`6.00 ms`** (3ª) — mais de **18x–49x mais rápido** na execução da função.

### 2. O que o campo `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers` e o que isso diz sobre guardar estado em variável global?

Os logs reais do Passo 4 mostram exatamente a diferença de ciclo de vida do ambiente de execução:

- **Sem `--warm-containers` ([`evidencias/03-start-api-sem-warm.txt`](./evidencias/03-start-api-sem-warm.txt)):**
  - Requisição 1: `SAM_CONTAINER_ID: 98d099e4...` · `Init Duration: 0.11 ms` · `Duration: 97.52 ms` · **`invocacaoNesteAmbiente: 1`**
  - Requisição 2: `SAM_CONTAINER_ID: b0bca8a2...` · `Init Duration: 0.06 ms` · `Duration: 111.42 ms` · **`invocacaoNesteAmbiente: 1`**
  - Requisição 3: `SAM_CONTAINER_ID: d040adc4...` · `Init Duration: 0.11 ms` · `Duration: 145.05 ms` · **`invocacaoNesteAmbiente: 1`**
  - A cada chamada HTTP, o SAM sobe um contêiner novo e o descarta ao final, reiniciando `contador = 0` todas as vezes.

- **Com `--warm-containers LAZY` ([`evidencias/04-start-api-warm-lazy.txt`](./evidencias/04-start-api-warm-lazy.txt)):**
  - Requisição 1: `SAM_CONTAINER_ID: e674b8fb...` · `Init Duration: 0.07 ms` · `Duration: 113.98 ms` · **`invocacaoNesteAmbiente: 1`**
  - Requisição 2: `Lambda function 'HelloWorldFunction' is already running` (sem `Init Duration`) · `Duration: 2.31 ms` · **`invocacaoNesteAmbiente: 2`**
  - Requisição 3: `Lambda function 'HelloWorldFunction' is already running` (sem `Init Duration`) · `Duration: 6.00 ms` · **`invocacaoNesteAmbiente: 3`**
  - O contêiner é criado sob demanda na 1ª requisição (*lazy*) e mantido aquecido na memória para as chamadas seguintes.

**O que isso diz sobre guardar estado em variável global:**
- Uma variável global (`contador`) **não é persistente nem compartilhada globalmente entre toda a aplicação**: ela vive apenas na memória RAM de **uma instância específica de contêiner** enquanto essa instância permanecer aquecida.
- Em produção na AWS, o Lambda recicla contêineres após inatividade, atualizações de código/configuração ou erros, e cria **múltiplas instâncias paralelas** sob concorrência (cada uma com seu próprio `contador` independente).
- Portanto, **nunca** se deve armazenar estado de negócio, filas, sessões ou contadores confiáveis em variáveis globais — estado durável deve ir para serviços externos (DynamoDB, PostgreSQL/RDS, ElastiCache, SQS). Variáveis globais em Lambda servem apenas para **otimização de recursos reutilizáveis** inicializados no Init (clientes SDK, pools de conexão, cache somente leitura descartável).

### 3. O que aconteceu com `demorar = true`, qual é o tempo máximo de uma execução no Lambda e o que fazer com um trabalho mais longo?

Conforme registrado em [`evidencias/05-timeout-3s-vs-10s.txt`](./evidencias/05-timeout-3s-vs-10s.txt):

1. **Com `Timeout: 3` em `template.yaml`:** ao enviar `{"requisicaoId":"R-003","demorar":true}`, a função executa `time.sleep(5)` e é interrompida à força aos 3 segundos pelo ambiente de execução:
   - Log do SAM: `Function 'HelloWorldFunction' timed out after 3 seconds` e `Invalid lambda response received: Lambda response must be valid json`.
   - Resposta HTTP recebida pelo cliente: **`HTTP/1.1 502 BAD GATEWAY`** com corpo `{"message":"Internal server error"}`.
2. **Após alterar para `Timeout: 10` em `template.yaml` e recompilar com `sam build`:** a mesma requisição conclui os 5 segundos de espera e retorna sucesso:
   - Log do SAM: `REPORT RequestId: 636cd256-...  Init Duration: 0.09 ms  Duration: 5122.80 ms  Billed Duration: 5123 ms`.
   - Resposta HTTP: **`HTTP/1.1 201 CREATED`** com corpo `{"guia": "G-R-003", "hospital": null, "bolsas": [], "invocacaoNesteAmbiente": 1}`.

- **Qual é o tempo máximo de uma execução no AWS Lambda?**
  - Para funções AWS Lambda padrão (*on-demand*), o timeout padrão é **3 segundos** e o limite máximo configurável é **900 segundos (15 minutos)**.
  - Importante observar também que, quando exposta de forma síncrona atrás do **Amazon API Gateway** (REST/HTTP API), existe um limite rígido da integração HTTP de **29 segundos** (30 s no HTTP API) — ou seja, mesmo com Lambda configurado para 15 min, uma chamada HTTP síncrona via API Gateway dará timeout se demorar mais que 29 s.

- **O que fazer com um trabalho mais longo?**
  - **Desacoplar em fluxo assíncrono:** a API recebe a requisição, grava o pedido/status e coloca uma mensagem em uma fila (**Amazon SQS**), respondendo imediatamente `HTTP 202 Accepted` com um `jobId` para o cliente consultar depois (polling) ou receber notificação (webhook / WebSocket / SNS).
  - **Orquestrar etapas com AWS Step Functions:** dividir o processamento em múltiplas tarefas menores em sequência ou paralelo (*Map* / *Distributed Map*) com controle de estado, retentativas e compensações.
  - **Delegar cargas contínuas ou pesadas (> 15 min) para contêineres/batch:** processar as mensagens da fila com **AWS Fargate / Amazon ECS**, **AWS Batch** ou workers em **EC2**, onde não há teto de 15 minutos por execução.

### 4. Cálculo explícito do custo de 1 milhão de execuções com 128 MB e 100 ms (`us-east-1`)

Considerando os preços de referência da região `us-east-1` informados no enunciado:
- **Solicitações:** `US$ 0,20` por 1 milhão de requisições
- **Computação (duração × memória):** `US$ 0,0000166667` por `GB-s`

**Parâmetros:**
- Número de execuções = `1.000.000`
- Memória configurada = `128 MB = 128 / 1024 GB = 0,125 GB`
- Duração por execução = `100 ms = 0,1 s`

**Cálculo passo a passo (custo bruto, sem Free Tier):**
1. **Total de GB-segundos consumidos:**
   $$\text{GB-s} = 1.000.000 \times 0,1\text{ s} \times 0,125\text{ GB} = 12.500\text{ GB-s}$$
2. **Custo por solicitações:**
   $$\text{Custo}_{\text{req}} = \frac{1.000.000}{1.000.000} \times \text{US\$ } 0,20 = \text{US\$ } 0,20$$
3. **Custo por computação:**
   $$\text{Custo}_{\text{comp}} = 12.500 \times \text{US\$ } 0,0000166667 = \text{US\$ } 0,20833375$$
4. **Custo total bruto (excluindo Free Tier):**
   $$\text{Total bruto} = \text{US\$ } 0,20 + \text{US\$ } 0,20833375 = \mathbf{\text{US\$ } 0,40833375} \ (\approx \mathbf{\text{US\$ } 0,41})$$

> **Nota sobre o AWS Always Free Tier:** o nível gratuito mensal do AWS Lambda inclui **1 milhão de solicitações** e **400.000 GB-s** por mês. Como ambos os consumos do cenário (`1.000.000` requisições e `12.500 GB-s`) cabem dentro dessa franquia, o valor faturado seria **`US$ 0,00`** caso a cota gratuita da conta não tenha sido consumida por outras funções no mês.

### 5. Em que parte do Rota Vital uma função assim faria sentido e em qual não faria? (4 contextos reais do `Proj-EAD`)

Com base na especificação arquitetural do projeto **Rota Vital** em `Proj-EAD` (`docs/arquitetura/u2-parte-2/planejamento.md`, ADRs `0001` e `0002` e `docs/medicoes.md`), o sistema é dividido em **4 contextos delimitados**:

1. **Expedição (`AtenderRequisicao`, `GuiaDeSaida`, `Roteirizador`) — ONDE MAIS FAZ SENTIDO:**
   - É um *Contexto de Suporte* responsável por traduzir os itens baixados (`AdaptadorEstoque`), calcular a rota sintética (`Roteirizador.calcular`, até 256 paradas) e emitir a `GuiaDeSaida` diretamente na resposta HTTP (a persistência de histórico de guias ainda é adiada segundo `planejamento.md`).
   - A geração da guia de transporte (`POST /guias`) é **curta** (meta média `0,20–0,25 s`, `p95 <= 500 ms`), **stateless**, determinística a partir dos dados recebidos no evento e com volume variável (`3.600` a `150.000` pedidos/mês nos cenários da especificação), beneficiando-se de escala automática e pagamento por execução — desde que a etapa de baixa transacional no banco continue isolada no núcleo transacional ou coordenada via Saga.

2. **Estoque (`Bolsa`, `OcorrenciaBolsa`, `LoteDeBaixa`, `JdbcEstoqueStorage`) — ONDE NÃO FAZ SENTIDO:**
   - É um *Contexto Principal (Core)* com forte invariante transacional: controla ocorrências físicas de bolsas (`bolsas`), validade, compatibilidade sanguínea e dedução atômica FEFO (`alocar`).
   - Não deve ser implementado como uma função Lambda isolada usando estado em memória/global (variáveis globais se perdem a cada reciclagem de contêiner e não são compartilhadas entre instâncias concorrentes), nem sem controle transacional rígido com banco relacional (`ACID`, rollback `409 Conflict` em caso de estoque insuficiente e proteção contra contenção de conexões).

3. **Requisições (`RequisicaoController`, `FilaJdbc`) — ONDE NÃO FAZ SENTIDO:**
   - É um *Contexto Principal (Core)* que mantém a fila hospitalar FIFO persistida no PostgreSQL (`requisicoes`), executando `LOCK TABLE requisicoes IN SHARE ROW EXCLUSIVE MODE` dentro da **mesma transação ACID** que consome o estoque (ADR-0002).
   - Guardar fila em variável global de Lambda quebraria a ordenação e perderia requisições em cold starts/escalonamento horizontal; além disso, múltiplas Lambdas concorrentes disputando `LOCK TABLE` no PostgreSQL criariam gargalo severo. Numa arquitetura serverless nativa, a fila precisaria migrar para um serviço gerenciado (como **Amazon SQS FIFO**), não para estado em memória na função.

4. **Comparação Textual (`DuplicateController`, `DuplicateDetectionService`) — CASO HÍBRIDO / NUANÇADO:**
   - É um *Contexto Genérico* totalmente **stateless** e desacoplado do fluxo clínico (*"caminhos separados"*), sem dependência de banco de dados — o que, à primeira vista, favorece isolá-lo da API principal.
   - Porém, executa comparação de distância de Levenshtein `O(n²)` intensiva em CPU: segundo `docs/medicoes.md`, `8.000` itens levam `23,9 s` (1 thread) ou `7,2 s` (8 threads); `16.000` levam `97,9 s` (1 thread) ou `41,3 s` (8 threads), **já ultrapassando o timeout síncrono de 29 s do API Gateway**; e cargas de `100.000` a `1.000.000` estimam `~1,04 h` a `~4,32 dias`, **excedendo o limite máximo de 15 min (900 s) do AWS Lambda**.
   - Logo, **não faz sentido** como Lambda HTTP síncrona nem para lotes grandes; só funciona em Lambda para **micro-lotes pequenos ou particionados de forma assíncrona** (e para volumes maiores deve rodar em **SQS + ECS/Fargate ou AWS Batch**).

---

## Uso de IA (PO-CESAR-021)

- **Ferramenta utilizada:** Assistente de IA integrado ao ambiente de desenvolvimento (modelo de linguagem para engenharia de software).
- **Finalidade:** Estruturação do projeto AWS SAM (`template.yaml`, `.gitignore`, `hello_world/`), automação da captura dos logs reais e geração das imagens PNG de terminal em `evidencias/`, redação e preenchimento do `README.md` (incluindo o cálculo da Pergunta 4 e a análise arquitetural da Pergunta 5 com base no `Proj-EAD`) e elaboração do `ROTEIRO_VIDEO.md`.
- **Validação realizada:**
  - O código de `hello_world/app.py` foi conferido byte a byte contra o enunciado oficial da atividade.
  - Todos os experimentos locais foram executados de forma real com **AWS SAM CLI (`1.174.0`)** e **Docker Desktop (`29.6.2`)** (`sam validate`, `sam build`, 2x `sam local invoke`, 3x `sam local start-api` padrão, 3x `--warm-containers LAZY` e experimento de `Timeout: 3` -> `Timeout: 10`), conferindo os valores de resposta HTTP, tempos de `REPORT RequestId` e saídas em `evidencias/`.
  - Os cálculos da Pergunta 4 foram conferidos numericamente e os contextos da Pergunta 5 foram verificados diretamente contra os documentos e classes do repositório `Proj-EAD`.

---

## Roteiro de Vídeo

Consulte o arquivo [`ROTEIRO_VIDEO.md`](./ROTEIRO_VIDEO.md) para o roteiro cronometrado (até 3 minutos) com os comandos na ordem exata de gravação.
