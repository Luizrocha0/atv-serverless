# Roteiro de Vídeo — Atividade Individual 1: Serverless (até 3 minutos)

**Objetivo:** Demonstrar, em até **3 minutos**, a execução local da função Lambda `POST /guias` (Rota Vital) pelo **Caminho A** (AWS SAM CLI + Docker Desktop), cobrindo:
1. Invocação com evento (`sam local invoke`) e comparação de tempos;
2. Gatilho HTTP (`sam local start-api`) sem e com `--warm-containers LAZY`;
3. Erro de tempo limite (`Timeout: 3` + `demorar: true`) e correção para `Timeout: 10`.

---

## Preparação antes de gravar (fora da contagem do vídeo)

1. Certifique-se de que o **Docker Desktop** está aberto e rodando (`docker info` retorna sem erro) e que `sam --version` funciona.
2. Abra **dois terminais Git Bash** lado a lado (ou superior/inferior) na pasta `c:\Users\ppfr7\OneDrive\Desktop\atv`.
   - **Terminal 1:** comandos de build, `invoke` e servidor `sam local start-api`.
   - **Terminal 2:** chamadas `curl` para `http://127.0.0.1:3000/guias`.
3. Deixe o arquivo `template.yaml` aberto no editor com `Timeout: 3` na seção `Globals.Function`, pronto para editar para `10` na parte 3.

---

## Sequência Cronometrada (0:00 – 3:00)

### Bloco 1 — Estrutura, Build e Invocação com Evento (`0:00 – 0:55`)

**O que mostrar no Terminal 1:**
```bash
ls -la
sam build
sam local generate-event apigateway aws-proxy --method POST --path guias --body '{"requisicaoId":"R-001","hospital":"H-Recife","bolsas":["O-NEG-0142"]}' > events/pedido.json
sam local invoke -e events/pedido.json
sam local invoke -e events/pedido.json
```

**Fala resumida:**
> "Olá! Esta é a entrega da Atividade 1 de Serverless pelo Caminho A, usando AWS SAM CLI, Docker Desktop e Python 3.12 para gerar guias de transporte do Rota Vital. Aqui temos `template.yaml` com a rota `POST /guias`, `Timeout` inicial de 3 segundos e `hello_world/app.py`. Já compilei com `sam build` e gerei o evento `events/pedido.json`. Ao rodar `sam local invoke` pela primeira vez, observamos a fase de inicialização (`Init Duration`) e o retorno 201 com `guia: G-R-001` e `invocacaoNesteAmbiente: 1`. Na segunda execução com `invoke`, o tempo cai porque a imagem já está preparada, mas cada `invoke` sobe um contêiner novo."

---

### Bloco 2 — Gatilho HTTP: Padrão vs `--warm-containers LAZY` (`0:55 – 2:00`)

**1. No Terminal 1 (servidor padrão):**
```bash
sam local start-api
```
*(Aguardar a mensagem `Running on http://127.0.0.1:3000`.)*

**2. No Terminal 2 (3 chamadas sem `--warm-containers`):**
```bash
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
```

**3. No Terminal 1 (`Ctrl+C`, reiniciar com `LAZY`):**
```bash
sam local start-api --warm-containers LAZY
```

**4. No Terminal 2 (repetir as 3 chamadas):**
```bash
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
```

**Fala resumida:**
> "Agora testamos o gatilho HTTP com `sam local start-api`. No modo padrão, cada uma das três requisições `POST /guias` retorna `invocacaoNesteAmbiente: 1`, porque o SAM cria e descarta um contêiner a cada chamada. Reiniciando com `--warm-containers LAZY`, o contêiner é inicializado na primeira chamada e reaproveitado nas seguintes, fazendo `invocacaoNesteAmbiente` subir para `1`, `2` e `3`. Isso mostra que variáveis globais só duram enquanto aquela instância específica do contêiner estiver aquecida."

---

### Bloco 3 — Experimento de Tempo Limite (`Timeout: 3` -> `Timeout: 10`) (`2:00 – 3:00`)

**1. No Terminal 2 (com `Timeout: 3` ainda ativo):**
```bash
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-003","demorar":true}'
```
*(Apontar o erro HTTP e o log `Task timed out after 3.00 seconds` no Terminal 1.)*

**2. No Editor (`template.yaml`) + Terminal 1:**
Alterar `Timeout: 3` para `Timeout: 10`, salvar o arquivo, parar o servidor (`Ctrl+C`) e rodar:
```bash
sam build
sam local start-api
```

**3. No Terminal 2 (repetir a requisição lenta):**
```bash
curl -sS -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId":"R-003","demorar":true}'
```

**Fala resumida:**
> "Por fim, enviamos `demorar: true`, que executa `time.sleep(5)`. Com `Timeout: 3` no `template.yaml`, a função é interrompida e o log registra `Task timed out after 3.00 seconds`. Alterando `Timeout` para `10`, recompilando com `sam build` e repetindo a chamada, a função conclui em cerca de 5 segundos e retorna HTTP `201 Created` com a guia `G-R-003`. Obrigado!"
