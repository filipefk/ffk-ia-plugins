---
name: trello-card
description: Cria ou lê um card no Trello a partir de uma descrição em linguagem natural. A API Key, o Token e o ID da lista padrão são obtidos nas variáveis de ambiente TRELLO_API_KEY, TRELLO_TOKEN e TRELLO_LIST_ID.
---

## Como usar

```
/trello-card <descrição livre da funcionalidade ou bug>
/trello-card ler <id>
```

## Referências

- [Regras de criação de cards](references/card-rules.md) — tipo, título, labels, confirmação
- [Modelos de cards (Markdown)](references/card-models.md) — templates para Story, Bug e Task

## Instruções de execução

Ao ser invocada, verifique o conteúdo de `$ARGUMENTS` na ordem abaixo (a primeira regra que casar vence):

1. Se começar com `ler` seguido de um ID (ex: `ler abc123`) → **fluxo de leitura**.
2. Caso contrário → **fluxo de criação simples**.

---

## Fluxo de leitura

### Passo 1 — Buscar o card via API

Execute o seguinte script PowerShell:

```powershell
$apiKey = $env:TRELLO_API_KEY
if (-not $apiKey) { throw "Variável de ambiente TRELLO_API_KEY não encontrada." }

$token = $env:TRELLO_TOKEN
if (-not $token) { throw "Variável de ambiente TRELLO_TOKEN não encontrada." }

$cardId = "<id extraído dos argumentos>"
$url = "https://api.trello.com/1/cards/$cardId`?key=$apiKey&token=$token&fields=name,desc,url,idList,labels,due,dateLastActivity&members=true"

$response = Invoke-RestMethod -Uri $url -Method Get
$response | ConvertTo-Json -Depth 10
```

### Passo 2 — Exibir os dados do card

Apresente as informações relevantes em formato legível:

```
ID:       <id>
Título:   <name>
URL:      <url>
Labels:   <labels[].name separados por vírgula>
Vence em: <due ou "sem prazo">

Descrição:
<desc>
```

---

## Fluxo de criação simples

### Passo 1 — Interpretar a descrição

Leia o texto em `$ARGUMENTS` e infira tipo, título e descrição conforme as regras em [card-rules.md](references/card-rules.md) e os modelos em [card-models.md](references/card-models.md).

### Passo 2 — Exibir o card para confirmação

Mostre ao usuário o card montado (formato em [card-rules.md](references/card-rules.md)) e use `AskUserQuestion` com as opções **Criar card** e **Cancelar**.

Se o usuário cancelar, encerre sem fazer nada.

### Passo 3 — Buscar o ID do label via API

Antes de criar o card, obtenha o `idLabel` correto para o tipo inferido buscando os labels do board:

```powershell
$apiKey = $env:TRELLO_API_KEY
if (-not $apiKey) { throw "Variável de ambiente TRELLO_API_KEY não encontrada." }

$token = $env:TRELLO_TOKEN
if (-not $token) { throw "Variável de ambiente TRELLO_TOKEN não encontrada." }

$listId = $env:TRELLO_LIST_ID
if (-not $listId) { throw "Variável de ambiente TRELLO_LIST_ID não encontrada." }

# Obter o board a partir da lista
$listUrl = "https://api.trello.com/1/lists/$listId`?key=$apiKey&token=$token&fields=idBoard"
$list = Invoke-RestMethod -Uri $listUrl -Method Get
$boardId = $list.idBoard

# Buscar labels do board
$labelsUrl = "https://api.trello.com/1/boards/$boardId/labels?key=$apiKey&token=$token"
$labels = Invoke-RestMethod -Uri $labelsUrl -Method Get
$labels | ConvertTo-Json
```

A partir da resposta, identifique o label cujo `name` corresponda ao tipo inferido (Bug, Story ou Task) ou cuja `color` corresponda à cor definida nas regras. Se não encontrar label correspondente, crie o card **sem label**.

### Passo 4 — Criar o card via API

```powershell
$apiKey = $env:TRELLO_API_KEY
if (-not $apiKey) { throw "Variável de ambiente TRELLO_API_KEY não encontrada." }

$token = $env:TRELLO_TOKEN
if (-not $token) { throw "Variável de ambiente TRELLO_TOKEN não encontrada." }

$listId = $env:TRELLO_LIST_ID
if (-not $listId) { throw "Variável de ambiente TRELLO_LIST_ID não encontrada." }

$body = @{
    name   = "<título do card>"
    desc   = "<descrição em Markdown>"
    idList = $listId
}

# Adicionar idLabels somente se encontrou o label no Passo 3
# $body["idLabels"] = @("<idLabel>")

$url = "https://api.trello.com/1/cards?key=$apiKey&token=$token"

$response = Invoke-RestMethod -Uri $url -Method Post `
    -ContentType "application/json" `
    -Body ($body | ConvertTo-Json)
$response.id
$response.url
```

### Passo 5 — Reportar o resultado

- **Sucesso:** exibir o ID e a URL do card criado (campo `url` da resposta).
- **Erro:** exibir a mensagem de erro retornada pela API ou pelo PowerShell.
