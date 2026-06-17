---
name: azure-card
description: Cria ou lê um work item no Azure DevOps a partir de uma descrição em linguagem natural. Suporta criação de hierarquias pai/filho (ex User Story com Tasks filhas). A URL do board e a API KEY do usuário são obtidos nas variáveis de ambiente AZURE_URL_BOARD e AZURE_USER_API_KEY
---

## Como usar

```
/azure-card <descrição livre da funcionalidade ou bug>
/azure-card ler <id>
```

## Referências

- [Regras de criação de cards](references/card-rules.md) — tipo, título, campo por tipo, confirmação
- [Modelos de cards (HTML)](references/card-models.md) — templates HTML para User Story, Bug e Task

## Instruções de execução

Ao ser invocada, verifique o conteúdo de `$ARGUMENTS` na ordem abaixo (a primeira regra que casar vence):

1. Se começar com `ler` seguido de um número (ex: `ler 543`) → **fluxo de leitura**.
2. Se mencionar um ID numérico de card pai existente (ex: "filho do card 123", "filho do 456", "vinculado ao card 789", "child of 321") → **fluxo de criação filho de card existente**. Esta regra prevalece sobre a seguinte.
3. Se a descrição mencionar criação de cards filhos sem ID de pai (ex: "criar tasks filhas", "com dois filhos", "tasks filhas") → **fluxo de criação com filhos**.
4. Caso contrário → **fluxo de criação simples**.

---

## Fluxo de leitura

### Passo 1 — Buscar o card via API

Execute o seguinte script PowerShell:

```powershell
$token = $env:AZURE_USER_API_KEY
if (-not $token) { throw "Variável de ambiente AZURE_USER_API_KEY não encontrada." }

$boardUrl = $env:AZURE_URL_BOARD
if (-not $boardUrl) { throw "Variável de ambiente AZURE_URL_BOARD não encontrada." }

$id = <id extraído dos argumentos>
$url = "$boardUrl/_apis/wit/workitems/$id`?api-version=7.1"

$response = Invoke-RestMethod -Uri $url -Method Get `
    -Headers @{ Authorization = "Bearer $token" }

$response | ConvertTo-Json -Depth 10
```

### Passo 2 — Exibir os dados do card

Apresente as informações relevantes em formato legível:

```
ID:      <id>
Tipo:    <System.WorkItemType>
Título:  <System.Title>
Estado:  <System.State>
Área:    <System.AreaPath>
Criado:  <System.CreatedDate>

Descrição:
<System.Description sem tags HTML>
```

---

## Fluxo de criação filho de card existente

Use este fluxo quando o usuário quiser criar um card filho vinculado a um card pai já existente no Azure DevOps (identificado pelo ID numérico nos argumentos).

### Passo 1 — Extrair o ID do pai e interpretar o filho

A partir dos `$ARGUMENTS`, extraia o **ID do card pai** existente e, com o restante da descrição, infira o card filho seguindo as regras em [card-rules.md](references/card-rules.md) e os modelos em [card-models.md](references/card-models.md).

### Passo 2 — Exibir para confirmação

Mostre o plano antes de criar (formato descrito em [card-rules.md](references/card-rules.md)):

```
Card pai existente: ID <parentId>

Filho a criar:
Tipo:    <tipo>
Título:  <título>

Descrição:
<descrição em texto legível, sem tags HTML>
```

Use `AskUserQuestion` com as opções **Criar card filho** e **Cancelar**.

Se o usuário cancelar, encerre sem fazer nada.

### Passo 3 — Criar o card filho com link ao pai

Para Bug, o campo de conteúdo é `Microsoft.VSTS.TCM.ReproSteps`; para os demais tipos, é `System.Description` (ver [card-rules.md](references/card-rules.md)).

```powershell
$token = $env:AZURE_USER_API_KEY
if (-not $token) { throw "Variável de ambiente AZURE_USER_API_KEY não encontrada." }

$boardUrl = $env:AZURE_URL_BOARD
if (-not $boardUrl) { throw "Variável de ambiente AZURE_URL_BOARD não encontrada." }

$childType = [Uri]::EscapeDataString("<tipo do filho, ex: Task>")
$parentId  = <id do card pai existente>
$url = "$boardUrl/_apis/wit/workitems/`$$childType`?api-version=7.1"

# Use /fields/Microsoft.VSTS.TCM.ReproSteps para Bug; /fields/System.Description para os demais
$descriptionPath = "<caminho do campo conforme tipo>"

$body = @(
    @{ op = "add"; path = "/fields/System.Title"; value = "<título do filho>" }
    @{ op = "add"; path = $descriptionPath;        value = "<descrição HTML do filho>" }
    @{
        op    = "add"
        path  = "/relations/-"
        value = @{
            rel        = "System.LinkTypes.Hierarchy-Reverse"
            url        = "$boardUrl/_apis/wit/workitems/$parentId"
            attributes = @{ comment = "" }
        }
    }
) | ConvertTo-Json -Depth 5

$response = Invoke-RestMethod -Uri $url -Method Post `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "application/json-patch+json" `
    -Body $body

$response.id
$response._links.html.href
```

### Passo 4 — Reportar o resultado

Exiba o ID e URL do card filho criado, mencionando o ID do pai ao qual foi vinculado.

---

## Fluxo de criação com filhos

Use este fluxo quando o usuário pedir explicitamente para criar cards filhos vinculados ao card pai (ex: "User Story com duas Tasks filhas").

### Passo 1 — Interpretar pai e filhos

A partir dos `$ARGUMENTS`, infira o **card pai** e os **cards filhos** seguindo as regras em [card-rules.md](references/card-rules.md) e os modelos em [card-models.md](references/card-models.md).

Para cada filho, liste:
- **Tipo:** normalmente `Task`, mas pode ser qualquer tipo inferido
- **Título:** frase curta com o sufixo obrigatório ` (Criado pelo Claude Code)`
- **Descrição:** HTML estruturado conforme o modelo do tipo

### Passo 2 — Exibir para confirmação

Mostre o plano completo antes de criar qualquer coisa:

```
PAI
Tipo:    <tipo>
Título:  <título>

Filhos:
  1. [Task] <título filho 1>
  2. [Task] <título filho 2>
  ...
```

Use `AskUserQuestion` com as opções **Criar todos** e **Cancelar**.

Se o usuário cancelar, encerre sem fazer nada.

### Passo 3 — Criar o card pai

Execute o script de criação simples (Passo 3 do fluxo de criação simples) para o card pai. Guarde o `$parentId` retornado.

### Passo 4 — Criar cada card filho com link de pai

Para cada filho, execute o script abaixo. Lembre: Bug usa `Microsoft.VSTS.TCM.ReproSteps`; demais tipos usam `System.Description`.

```powershell
$token = $env:AZURE_USER_API_KEY
if (-not $token) { throw "Variável de ambiente AZURE_USER_API_KEY não encontrada." }

$boardUrl = $env:AZURE_URL_BOARD
if (-not $boardUrl) { throw "Variável de ambiente AZURE_URL_BOARD não encontrada." }

$childType = [Uri]::EscapeDataString("<tipo do filho, ex: Task>")
$parentId  = <id do card pai criado no passo anterior>
$url = "$boardUrl/_apis/wit/workitems/`$$childType`?api-version=7.1"

# Use /fields/Microsoft.VSTS.TCM.ReproSteps para Bug; /fields/System.Description para os demais
$descriptionPath = "<caminho do campo conforme tipo>"

$body = @(
    @{ op = "add"; path = "/fields/System.Title"; value = "<título do filho>" }
    @{ op = "add"; path = $descriptionPath;        value = "<descrição HTML do filho>" }
    @{
        op    = "add"
        path  = "/relations/-"
        value = @{
            rel        = "System.LinkTypes.Hierarchy-Reverse"
            url        = "$boardUrl/_apis/wit/workitems/$parentId"
            attributes = @{ comment = "" }
        }
    }
) | ConvertTo-Json -Depth 5

try {
    $response = Invoke-RestMethod -Uri $url -Method Post `
        -Headers @{ Authorization = "Bearer $token" } `
        -ContentType "application/json-patch+json" `
        -Body $body
    "Filho criado — ID: $($response.id)"
    $response._links.html.href
} catch {
    "ERRO ao criar filho '<título do filho>': $($_.Exception.Message)"
}
```

> `System.LinkTypes.Hierarchy-Reverse` significa "este item é filho de parentId".

### Passo 5 — Reportar o resultado

Exiba um resumo com os IDs e URLs de todos os cards criados. Se algum filho falhou, liste-o como erro sem interromper o relatório dos demais:

```
Pai criado:
  ID <parentId> — <url pai>

Filhos criados:
  ID <id1> — <url filho 1>
  ID <id2> — <url filho 2>

Filhos com erro (se houver):
  <título filho> — <mensagem de erro>
```

---

## Fluxo de criação simples

### Passo 1 — Interpretar a descrição

Leia o texto em `$ARGUMENTS` e infira tipo, título e descrição conforme as regras em [card-rules.md](references/card-rules.md) e os modelos HTML em [card-models.md](references/card-models.md).

### Passo 2 — Exibir o card para confirmação

Mostre ao usuário o card montado (formato em [card-rules.md](references/card-rules.md)) e use `AskUserQuestion` com as opções **Criar card** e **Cancelar**.

Se o usuário cancelar, encerre sem fazer nada.

### Passo 3 — Criar o card via API

Bug usa `Microsoft.VSTS.TCM.ReproSteps`; demais tipos usam `System.Description`.

```powershell
$token = $env:AZURE_USER_API_KEY
if (-not $token) { throw "Variável de ambiente AZURE_USER_API_KEY não encontrada." }

$boardUrl = $env:AZURE_URL_BOARD
if (-not $boardUrl) { throw "Variável de ambiente AZURE_URL_BOARD não encontrada." }

$workItemType = [Uri]::EscapeDataString("<tipo do work item, ex: User Story>")
$url = "$boardUrl/_apis/wit/workitems/`$$workItemType`?api-version=7.1"

# Use /fields/Microsoft.VSTS.TCM.ReproSteps para Bug; /fields/System.Description para os demais
$descriptionPath = "<caminho do campo conforme tipo>"

$body = @(
    @{ op = "add"; path = "/fields/System.Title"; value = "<título>" }
    @{ op = "add"; path = $descriptionPath;        value = "<descrição HTML>" }
) | ConvertTo-Json -Depth 5

$response = Invoke-RestMethod -Uri $url -Method Post `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "application/json-patch+json" `
    -Body $body

$response.id
$response._links.html.href
```

### Passo 4 — Reportar o resultado

- **Sucesso:** exibir a URL do card criado (campo `_links.html.href` da resposta).
- **Erro:** exibir a mensagem de erro retornada pela API ou pelo PowerShell.
