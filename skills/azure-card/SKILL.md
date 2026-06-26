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

Execute o seguinte script bash:

```bash
AZURE_USER_API_KEY="${AZURE_USER_API_KEY:?Variável AZURE_USER_API_KEY não encontrada}"
AZURE_URL_BOARD="${AZURE_URL_BOARD:?Variável AZURE_URL_BOARD não encontrada}"

ID="<id extraído dos argumentos>"
URL="${AZURE_URL_BOARD}/_apis/wit/workitems/${ID}?api-version=7.1"

curl -s -H "Authorization: Bearer ${AZURE_USER_API_KEY}" "$URL"
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

```bash
AZURE_USER_API_KEY="${AZURE_USER_API_KEY:?Variável AZURE_USER_API_KEY não encontrada}"
AZURE_URL_BOARD="${AZURE_URL_BOARD:?Variável AZURE_URL_BOARD não encontrada}"

CHILD_TYPE="<tipo do filho, ex: Task>"
CHILD_TYPE_ENCODED=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${CHILD_TYPE}'))")
PARENT_ID="<id do card pai existente>"
URL="${AZURE_URL_BOARD}/_apis/wit/workitems/\$${CHILD_TYPE_ENCODED}?api-version=7.1"

# Use /fields/Microsoft.VSTS.TCM.ReproSteps para Bug; /fields/System.Description para os demais
DESCRIPTION_PATH="<caminho do campo conforme tipo>"

PAYLOAD_FILE=$(mktemp --suffix=.json)

cat > "$PAYLOAD_FILE" <<EOF
[
  { "op": "add", "path": "/fields/System.Title", "value": "<título do filho>" },
  { "op": "add", "path": "${DESCRIPTION_PATH}", "value": "<descrição HTML do filho>" },
  {
    "op": "add",
    "path": "/relations/-",
    "value": {
      "rel": "System.LinkTypes.Hierarchy-Reverse",
      "url": "${AZURE_URL_BOARD}/_apis/wit/workitems/${PARENT_ID}",
      "attributes": { "comment": "" }
    }
  }
]
EOF

curl -s -X POST "$URL" \
  -H "Authorization: Bearer ${AZURE_USER_API_KEY}" \
  -H "Content-Type: application/json-patch+json; charset=utf-8" \
  --data-binary "@${PAYLOAD_FILE}"

rm -f "$PAYLOAD_FILE"
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

```bash
AZURE_USER_API_KEY="${AZURE_USER_API_KEY:?Variável AZURE_USER_API_KEY não encontrada}"
AZURE_URL_BOARD="${AZURE_URL_BOARD:?Variável AZURE_URL_BOARD não encontrada}"

CHILD_TYPE="<tipo do filho, ex: Task>"
CHILD_TYPE_ENCODED=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${CHILD_TYPE}'))")
PARENT_ID="<id do card pai criado no passo anterior>"
URL="${AZURE_URL_BOARD}/_apis/wit/workitems/\$${CHILD_TYPE_ENCODED}?api-version=7.1"

# Use /fields/Microsoft.VSTS.TCM.ReproSteps para Bug; /fields/System.Description para os demais
DESCRIPTION_PATH="<caminho do campo conforme tipo>"

PAYLOAD_FILE=$(mktemp --suffix=.json)

cat > "$PAYLOAD_FILE" <<EOF
[
  { "op": "add", "path": "/fields/System.Title", "value": "<título do filho>" },
  { "op": "add", "path": "${DESCRIPTION_PATH}", "value": "<descrição HTML do filho>" },
  {
    "op": "add",
    "path": "/relations/-",
    "value": {
      "rel": "System.LinkTypes.Hierarchy-Reverse",
      "url": "${AZURE_URL_BOARD}/_apis/wit/workitems/${PARENT_ID}",
      "attributes": { "comment": "" }
    }
  }
]
EOF

RESPONSE=$(curl -s -X POST "$URL" \
  -H "Authorization: Bearer ${AZURE_USER_API_KEY}" \
  -H "Content-Type: application/json-patch+json; charset=utf-8" \
  --data-binary "@${PAYLOAD_FILE}")

rm -f "$PAYLOAD_FILE"

if echo "$RESPONSE" | grep -q '"id"'; then
  echo "Filho criado — $RESPONSE"
else
  echo "ERRO ao criar filho '<título do filho>': $RESPONSE"
fi
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

```bash
AZURE_USER_API_KEY="${AZURE_USER_API_KEY:?Variável AZURE_USER_API_KEY não encontrada}"
AZURE_URL_BOARD="${AZURE_URL_BOARD:?Variável AZURE_URL_BOARD não encontrada}"

WORK_ITEM_TYPE="<tipo do work item, ex: User Story>"
WORK_ITEM_TYPE_ENCODED=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${WORK_ITEM_TYPE}'))")
URL="${AZURE_URL_BOARD}/_apis/wit/workitems/\$${WORK_ITEM_TYPE_ENCODED}?api-version=7.1"

# Use /fields/Microsoft.VSTS.TCM.ReproSteps para Bug; /fields/System.Description para os demais
DESCRIPTION_PATH="<caminho do campo conforme tipo>"

PAYLOAD_FILE=$(mktemp --suffix=.json)

cat > "$PAYLOAD_FILE" <<EOF
[
  { "op": "add", "path": "/fields/System.Title", "value": "<título>" },
  { "op": "add", "path": "${DESCRIPTION_PATH}", "value": "<descrição HTML>" }
]
EOF

curl -s -X POST "$URL" \
  -H "Authorization: Bearer ${AZURE_USER_API_KEY}" \
  -H "Content-Type: application/json-patch+json; charset=utf-8" \
  --data-binary "@${PAYLOAD_FILE}"

rm -f "$PAYLOAD_FILE"
```

### Passo 4 — Reportar o resultado

- **Sucesso:** exibir a URL do card criado (campo `_links.html.href` da resposta).
- **Erro:** exibir a mensagem de erro retornada pela API ou pelo PowerShell.
