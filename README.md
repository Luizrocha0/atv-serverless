# Atividade Individual 1 — Serverless (Rota Vital)

- **Disciplina:** Infraestrutura de Software · ADS · CESAR School · 2026.2
- **Caminho escolhido:** **Caminho A** — Execução local com **AWS SAM CLI** + **Docker Desktop** (runtime **Python 3.12**, pacote **`zip`**)

## Estrutura do Projeto

```text
atv/
├── .gitignore
├── README.md
├── ROTEIRO_VIDEO.md
├── template.yaml            # Rota POST /guias, Runtime python3.12, Timeout inicial: 3s
├── hello_world/
│   ├── __init__.py
│   ├── app.py               # Handler da função de geração de guia de transporte
│   └── requirements.txt
└── evidencias/
    └── LEIAME.txt           # Pasta reservada para logs (.txt) e capturas (.png) reais
```

## Status Atual do Ambiente (Etapa R1)

Durante a preparação automatizada do ambiente Windows, os dois pré-requisitos de execução local exigiram ação manual com privilégios de administrador / sessão interativa:

1. **AWS SAM CLI (`Amazon.SAM-CLI`)**: `winget install --id Amazon.SAM-CLI --exact --silent --accept-package-agreements --accept-source-agreements` baixou e validou o hash do instalador `AWS_SAM_CLI_64_PY3.msi` (`v1.174.0`), porém travou na etapa `Iniciando a instalação do pacote...` aguardando confirmação de UAC (shell sem elevação administrativa).
2. **Docker Desktop**: o cliente CLI (`Docker version 29.6.2`) está instalado, mas o serviço `com.docker.service` permaneceu `Stopped` e o daemon Linux não subiu automaticamente (`open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified`).

> **Ações manuais necessárias antes de executar os experimentos:**
> 1. Abrir um **PowerShell como Administrador**, executar:
>    ```powershell
>    winget install --id Amazon.SAM-CLI --exact --silent --accept-package-agreements --accept-source-agreements
>    ```
>    e confirmar o prompt UAC (se solicitado). Em seguida, abrir um novo terminal e validar `sam --version`.
> 2. Iniciar o **Docker Desktop** manualmente pelo Menu Iniciar (autorizando privilégios se solicitado) e aguardar até que `docker info` retorne código `0`.

---

## Como Executar Localmente (Caminho A)

> **Dica importante no Windows:** execute os comandos com redirecionamento (`> events/pedido.json`) e `curl` no **Git Bash** (`"C:\Program Files\Git\bin\bash.exe"`), pois o PowerShell 5.1 grava `>` em UTF-16LE (com BOM), remove aspas internas de JSON passadas para binários nativos e mapeia `curl` como alias de `Invoke-WebRequest`.

### 1. Compilar o projeto SAM

```bash
sam build
```

### 2. Gerar evento e invocar localmente (Passo 3)

```bash
mkdir -p events
sam local generate-event apigateway aws-proxy --method POST --path guias --body '{"requisicaoId":"R-001","hospital":"H-Recife","bolsas":["O-NEG-0142"]}' > events/pedido.json

sam local invoke -e events/pedido.json
sam local invoke -e events/pedido.json
```

Anote o tempo total e as métricas (`Init Duration`, `Duration`, `Billed Duration`) de cada uma das duas invocações.

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

Com `Globals.Function.Timeout: 3` ativo em `template.yaml`, envie:

```bash
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-003","demorar":true}'
```

Registre o erro de tempo limite (`Task timed out after 3.00 seconds`). Depois, altere `Timeout: 3` para `Timeout: 10` em `template.yaml`, recompile e reinicie a API:

```bash
sam build
sam local start-api
```

Repita a requisição com `"demorar":true` e confirme o retorno HTTP `201`.

---

## Evidências

Status: `[Aguardando execução local com SAM CLI e Docker Desktop]`

Quando os passos acima forem executados localmente, salve os logs reais (`.txt`) e capturas de terminal (`.png`) em `evidencias/`:
- `01-ambiente-sam-build.txt` / `.png`
- `02-invoke-evento.txt` / `.png`
- `03-start-api-sem-warm.txt` / `.png`
- `04-start-api-warm-lazy.txt` / `.png`
- `05-timeout-3s-vs-10s.txt` / `.png`

---

## Respostas às 5 Perguntas

### 1. Por que a primeira invocação demorou mais que a segunda e o que acontece na fase de Init?

`[Aguardando execução local com SAM CLI e Docker Desktop]`

*(Os tempos reais de `Init Duration`, `Duration` e tempo de execução das duas chamadas do Passo 3 serão preenchidos aqui após a execução local com `sam local invoke` e Docker Desktop ativos.)*

### 2. O que o campo `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers` e o que isso diz sobre guardar estado em variável global?

`[Aguardando execução local com SAM CLI e Docker Desktop]`

*(Os valores observados de `invocacaoNesteAmbiente` nas 3 requisições sem `--warm-containers` e nas 3 requisições com `--warm-containers LAZY` serão preenchidos aqui após a execução real do Passo 4.)*

### 3. O que aconteceu com `demorar = true`, qual é o tempo máximo de uma execução no Lambda e o que fazer com um trabalho mais longo?

`[Aguardando execução local com SAM CLI e Docker Desktop]`

*(O registro exato da falha com `Timeout: 3` e da recuperação com `Timeout: 10` será preenchido após a execução real do Passo 5, junto à análise de limite máximo e padrões assíncronos.)*

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
2. **Custo por requisições:**
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
- **Finalidade:** Estruturação inicial do projeto AWS SAM (`template.yaml`, `.gitignore`, `hello_world/`), redação da base do `README.md` (cálculo detalhado da Pergunta 4 e síntese arquitetural da Pergunta 5 a partir dos artefatos do `Proj-EAD`) e elaboração do `ROTEIRO_VIDEO.md`.
- **Validação realizada:**
  - O código de `hello_world/app.py` foi conferido linha a linha contra o enunciado oficial da atividade (`ORIGINAL_REQUEST.md`).
  - A sintaxe do código Python (`app.py`) e do manifesto `template.yaml` foi verificada localmente, assim como os dados arquiteturais da Pergunta 5 foram conferidos diretamente contra os documentos e classes do repositório oficial `Proj-EAD`.
  - A validação em tempo de execução (`sam build`, `sam local invoke`, `sam local start-api` e teste de timeout) permanece pendente de desbloqueio manual do **AWS SAM CLI** (UAC) e do **Docker Desktop** neste ambiente Windows, não tendo sido fabricada nenhuma saída ou captura de tela.

---

## Roteiro de Vídeo

Consulte o arquivo [`ROTEIRO_VIDEO.md`](./ROTEIRO_VIDEO.md) para o roteiro cronometrado (até 3 minutos) com os comandos na ordem exata de gravação.
