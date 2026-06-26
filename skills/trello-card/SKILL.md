---
name: trello-card
description: Cria ou lê um card no Trello a partir de uma descrição em linguagem natural. Requer apenas TRELLO_API_KEY e TRELLO_TOKEN. Board e coluna são resolvidos por nome.
---

## Como usar

```
/trello-card <descrição livre da funcionalidade ou bug>
/trello-card board:"Nome do Board" <descrição>
/trello-card board:"Nome do Board" coluna:"Nome da Coluna" <descrição>
/trello-card ler <id>
```

## Referências

- [Regras de criação de cards](references/card-rules.md) — tipo, título, labels, confirmação
- [Modelos de cards (Markdown)](references/card-models.md) — templates para Story, Bug e Task

## Variáveis de ambiente

| Variável         | Descrição                        |
|------------------|----------------------------------|
| `TRELLO_API_KEY` | Chave de API do Trello           |
| `TRELLO_TOKEN`   | Token de autenticação do Trello  |

## Instruções de execução

Ao ser invocada, verifique o conteúdo de `$ARGUMENTS` na ordem abaixo (a primeira regra que casar vence):

1. Se começar com `ler` seguido de um ID (ex: `ler abc123`) → **fluxo de leitura**.
2. Caso contrário → **fluxo de criação**.

---

## Fluxo de leitura

### Passo 1 — Buscar o card via API

```bash
TRELLO_API_KEY="${TRELLO_API_KEY:?Variável TRELLO_API_KEY não encontrada}"
TRELLO_TOKEN="${TRELLO_TOKEN:?Variável TRELLO_TOKEN não encontrada}"

CARD_ID="<id extraído dos argumentos>"
curl -s "https://api.trello.com/1/cards/${CARD_ID}?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}&fields=name,desc,url,idList,labels,due,dateLastActivity&members=true"
```

### Passo 2 — Exibir os dados do card

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

## Fluxo de criação

### Passo 1 — Extrair parâmetros dos argumentos

Analise `$ARGUMENTS` e extraia, se presentes:

- `board`: valor do parâmetro `board:"..."` ou `board:...` (sem aspas se for uma só palavra)
- `coluna`: valor do parâmetro `coluna:"..."` ou `coluna:...`
- O restante do texto (sem os parâmetros acima) é a **descrição do card**

Exemplos:
- `board:"Meu Projeto" bug no login` → board=`Meu Projeto`, descrição=`bug no login`
- `bug no login` → board não informado, descrição=`bug no login`
- `board:"Meu Projeto" coluna:"Em andamento" nova feature` → board=`Meu Projeto`, coluna=`Em andamento`, descrição=`nova feature`

### Passo 2 — Resolver o board

**Se o board foi informado:**

```bash
TRELLO_API_KEY="${TRELLO_API_KEY:?Variável TRELLO_API_KEY não encontrada}"
TRELLO_TOKEN="${TRELLO_TOKEN:?Variável TRELLO_TOKEN não encontrada}"

curl -s "https://api.trello.com/1/members/me/boards?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}&filter=open&fields=id,name"
```

Busque na resposta o board cujo `name` corresponda ao valor informado (comparação case-insensitive). Se não encontrar, informe o erro e encerre.

**Se o board NÃO foi informado:**

Execute o mesmo curl acima e liste os boards encontrados para o usuário escolher usando `AskUserQuestion`. Monte as opções com os nomes dos boards retornados (máximo 4; se houver mais, liste os 4 primeiros e informe que há outros). Após a escolha, use o `id` do board selecionado.

### Passo 3 — Resolver a coluna

Com o `BOARD_ID` obtido no passo anterior:

```bash
curl -s "https://api.trello.com/1/boards/${BOARD_ID}/lists?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}&filter=open&fields=id,name,pos"
```

- **Se a coluna foi informada:** encontre na resposta a lista cujo `name` corresponda (case-insensitive). Se não encontrar, informe o erro e encerre.
- **Se a coluna NÃO foi informada:** use a lista com o menor valor de `pos` (a primeira coluna).

Guarde o `id` da lista encontrada como `LIST_ID`.

### Passo 4 — Interpretar a descrição

Leia o texto da descrição extraída no Passo 1 e infira tipo, título e descrição conforme as regras em [card-rules.md](references/card-rules.md) e os modelos em [card-models.md](references/card-models.md).

### Passo 5 — Exibir o card para confirmação

Mostre ao usuário o card montado (formato em [card-rules.md](references/card-rules.md)), incluindo o nome do board e da coluna onde será criado. Use `AskUserQuestion` com as opções **Criar card** e **Cancelar**.

Se o usuário cancelar, encerre sem fazer nada.

### Passo 6 — Buscar o ID do label via API

```bash
curl -s "https://api.trello.com/1/boards/${BOARD_ID}/labels?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}"
```

Identifique o label cujo `name` corresponda ao tipo inferido (Bug, Story ou Task) ou cuja `color` corresponda à cor definida nas regras. Se não encontrar label correspondente, crie o card **sem label**.

### Passo 7 — Criar o card via API

```bash
BODY=$(cat <<EOF
{
  "name": "<título do card>",
  "desc": "<descrição em Markdown>",
  "idList": "${LIST_ID}"
}
EOF
)

# Adicionar "idLabels": ["<idLabel>"] ao JSON acima somente se encontrou o label no Passo 6

curl -s -X POST "https://api.trello.com/1/cards?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "$BODY"
```

### Passo 8 — Reportar o resultado

- **Sucesso:** exibir o nome do board, nome da coluna, ID e URL do card criado (campo `url` da resposta).
- **Erro:** exibir a mensagem de erro retornada pela API.
