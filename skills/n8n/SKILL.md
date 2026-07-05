---
name: n8n
description: Consulta e edita workflows do n8n via API REST — lista os fluxos existentes, exibe os nodes de um fluxo, obtém o JSON completo, atualiza nodes e conexões, duplica, ativa e desativa fluxos. Também consulta e edita data tables (colunas e linhas). Alterações sempre com backup prévio e confirmação do usuário. Requer N8N_API_KEY e N8N_URL.
---

## Como usar

```
/n8n listar
/n8n listar nome:"Nome do Fluxo"
/n8n listar ativo:true
/n8n nodos <id ou url do fluxo>
/n8n obter <id ou url do fluxo>
/n8n atualizar <id ou url do fluxo> <descrição das alterações>
/n8n duplicar <id ou url do fluxo> nome:"Nome da Cópia"
/n8n ativar <id ou url do fluxo>
/n8n desativar <id ou url do fluxo>
/n8n tabelas
/n8n tabelas nome:"Nome da Tabela"
/n8n tabela <id ou nome>
/n8n tabela <id ou nome> coluna <nome>:<tipo>
/n8n tabela <id ou nome> atualizar <descrição das alterações nas linhas>
```

## Variáveis de ambiente

| Variável      | Descrição                                                                        |
|---------------|-----------------------------------------------------------------------------------|
| `N8N_API_KEY` | Chave de API do n8n (enviada no header `X-N8N-API-KEY`)                          |
| `N8N_URL`     | URL base da instância n8n, sem o sufixo `/api/v1` (ex: `https://n8n.empresa.com`) |

> Ao montar a URL da API, remova a barra final de `N8N_URL` antes de concatenar `/api/v1` (`${N8N_URL%/}/api/v1`).

## Instruções de execução

Ao ser invocada, verifique o conteúdo de `$ARGUMENTS` na ordem abaixo (a primeira regra que casar vence):

1. Se começar com `nodos` seguido de uma URL (contém `/workflow/`) → **fluxo de nodes**, extraindo o ID do fluxo da URL.
2. Se começar com `nodos` seguido de um ID → **fluxo de nodes**, usando o ID diretamente.
3. Se começar com `obter` → **fluxo de obtenção do JSON**.
4. Se começar com `atualizar` → **fluxo de atualização**.
5. Se começar com `duplicar` → **fluxo de duplicação**.
6. Se começar com `ativar` → **fluxo de ativação/desativação** (ativar).
7. Se começar com `desativar` → **fluxo de ativação/desativação** (desativar).
8. Se começar com `tabelas` → **fluxo de listagem de data tables**.
9. Se começar com `tabela` e contiver `coluna <nome>:<tipo>` após a referência da tabela → **fluxo de criação de coluna**.
10. Se começar com `tabela` e contiver `atualizar` após a referência da tabela → **fluxo de atualização de linhas**.
11. Se começar com `tabela` → **fluxo de exibição de data table**.
12. Se começar com `listar`, ou os argumentos estiverem vazios → **fluxo de listagem de fluxos**.
13. Caso contrário → exibir a seção "Como usar" e encerrar sem chamar a API.

Nos fluxos que recebem `<id ou url do fluxo>`, aceite tanto o ID direto quanto a URL do editor, resolvendo conforme [Extraindo o ID de uma URL](#extraindo-o-id-de-uma-url).

### Extraindo o ID de uma URL

As URLs do editor n8n seguem o padrão `https://<dominio>/workflow/<id>`, podendo ter um sufixo depois do ID (ex: `/executions`). Extraia o segmento imediatamente após `/workflow/` e corte no próximo `/`, `?` ou `#`, o que vier primeiro.

```bash
# Exemplo:
# URL="https://n8n.empresa.com/workflow/abc123/executions"
# WORKFLOW_ID=$(echo "$URL" | sed -E 's#.*/workflow/([^/?#]+).*#\1#')
```

### Resolvendo a referência de uma data table

Nos fluxos de data table, `<id ou nome>` aceita o ID direto ou o nome da tabela:

1. Se o argumento tiver o formato de um ID do n8n (token alfanumérico, sem espaços, ~16 caracteres), use-o diretamente como `TABLE_ID`.
2. Caso contrário, faça `GET ${BASE_URL}/data-tables` e procure no array `data` um item cujo `name` case com o argumento (comparação case-insensitive).
3. Se nenhum ou mais de um item casar, exiba as tabelas disponíveis (`- <name> (ID: <id>)`) e encerre pedindo ao usuário que repita o comando com o ID.

### Descoberta de endpoints

Em caso de dúvida sobre um endpoint ou o schema de um corpo de requisição, consulte o spec OpenAPI da própria instância (somente leitura) antes de tentar chamadas às cegas:

```bash
curl -s "${BASE_URL}/openapi.yml" -H "X-N8N-API-KEY: ${N8N_API_KEY}"
```

### Regras para escrita na API

Válidas para os fluxos de **atualização** e **duplicação**:

- **Payload permitido**: o corpo do `PUT /workflows/{id}` e do `POST /workflows` deve conter **apenas** os campos `name`, `nodes`, `connections` e `settings`. Monte-o a partir do JSON do workflow com `jq '{name, nodes, connections, settings}'`. Os demais campos retornados pelo GET (`id`, `active`, `tags`, `createdAt`, `updatedAt`, `versionId`, `shared`, `isArchived`, `description`, `meta`, `pinData`, `staticData`, etc.) são rejeitados pela API com o erro `request/body must NOT have additional properties`.
- **Se `jq` não estiver disponível** no ambiente, use `node -e` (ou outra ferramenta disponível) para o mesmo filtro:
  ```bash
  node -e "
  const fs=require('fs');
  const w=JSON.parse(fs.readFileSync(process.argv[1],'utf8'));
  fs.writeFileSync(process.argv[2],JSON.stringify({name:w.name,nodes:w.nodes,connections:w.connections,settings:w.settings}));
  " "$WORK_FILE" "$PAYLOAD_FILE"
  ```
  > **Windows/Git Bash**: o Node do Windows não resolve caminhos `/tmp` do Git Bash. Converta os caminhos antes de passá-los ao `node`: `node script.js "$(cygpath -w "$WORK_FILE")"`.
- **Backup obrigatório**: antes de qualquer `PUT`, salve a resposta completa do `GET` em um arquivo temporário (`mktemp --suffix=.json`) e informe o caminho ao usuário no relatório final. A restauração é um `PUT` usando o backup filtrado pelos mesmos 4 campos.
- **Nunca ativar ou desativar** um fluxo como efeito colateral de outra operação — somente pelos comandos explícitos `ativar` e `desativar`. O campo `active` é somente leitura no `PUT`.
- **Confirmação obrigatória**: toda operação de escrita exibe um resumo do que será feito e pede confirmação com `AskUserQuestion` antes de chamar a API. Se o usuário cancelar, encerre sem fazer nada.

---

## Fluxo de listagem de fluxos

### Passo 1 — Extrair filtros opcionais

Analise `$ARGUMENTS` (após remover o prefixo `listar`) e extraia, se presentes:

- `nome`: valor do parâmetro `nome:"..."` ou `nome:...` → filtra pelo campo `name` do workflow
- `ativo`: valor do parâmetro `ativo:true` ou `ativo:false` → filtra pelo campo `active` do workflow

Se nenhum filtro for informado, liste todos os fluxos.

### Passo 2 — Buscar via API

```bash
N8N_API_KEY="${N8N_API_KEY:?Variável N8N_API_KEY não encontrada}"
N8N_URL="${N8N_URL:?Variável N8N_URL não encontrada}"
BASE_URL="${N8N_URL%/}/api/v1"

# Inclua --data-urlencode "name=<valor>" e/ou --data-urlencode "active=<true|false>"
# somente para os filtros que foram informados no Passo 1.
curl -s -G "${BASE_URL}/workflows" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  --data-urlencode "limit=250"
```

### Passo 3 — Exibir o resultado

Para cada item do array `data` da resposta, exibir:

```
- <name> (ID: <id>) — <"Ativo" se active=true, senão "Inativo"> — Atualizado em: <updatedAt>
```

Se `data` vier vazio, informar que nenhum fluxo foi encontrado com os filtros informados. Se a resposta trouxer `nextCursor` preenchido, avisar ao final que há mais resultados além dos exibidos e sugerir refinar a busca com `nome` ou `ativo`.

Se a resposta não tiver o campo `data` (erro da API), siga a seção [Tratamento de erros](#tratamento-de-erros).

---

## Fluxo de nodes de um fluxo

Exibe os nodes de um workflow específico, identificado por ID ou por URL do editor n8n.

### Passo 1 — Resolver o ID do fluxo

Se o argumento (após remover o prefixo `nodos`) for uma URL, extraia o `WORKFLOW_ID` conforme [Extraindo o ID de uma URL](#extraindo-o-id-de-uma-url). Caso contrário, use o argumento diretamente como `WORKFLOW_ID`.

### Passo 2 — Buscar via API

```bash
N8N_API_KEY="${N8N_API_KEY:?Variável N8N_API_KEY não encontrada}"
N8N_URL="${N8N_URL:?Variável N8N_URL não encontrada}"
BASE_URL="${N8N_URL%/}/api/v1"

WORKFLOW_ID="<id resolvido no Passo 1>"

curl -s "${BASE_URL}/workflows/${WORKFLOW_ID}" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}"
```

### Passo 3 — Exibir os nodes

Se a resposta trouxer o campo `nodes`, exibir:

```
Fluxo: <name> (ID: <id>)

Nodes:
- <name> [<type>]<" (desabilitado)" se disabled=true>
```

Se `nodes` vier vazio, informar que o fluxo não possui nodes.

Se a resposta não tiver o campo `nodes` (erro da API), siga a seção [Tratamento de erros](#tratamento-de-erros).

---

## Fluxo de obtenção do JSON

Obtém o JSON completo de um workflow e o salva em arquivo, para inspeção, edição ou backup.

### Passo 1 — Resolver o ID e buscar via API

Resolva o `WORKFLOW_ID` (após remover o prefixo `obter`) e execute:

```bash
N8N_API_KEY="${N8N_API_KEY:?Variável N8N_API_KEY não encontrada}"
N8N_URL="${N8N_URL:?Variável N8N_URL não encontrada}"
BASE_URL="${N8N_URL%/}/api/v1"

WORKFLOW_ID="<id resolvido>"
WORKFLOW_FILE=$(mktemp --suffix=.json)

curl -s "${BASE_URL}/workflows/${WORKFLOW_ID}" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  -o "$WORKFLOW_FILE"
```

Se a resposta for um erro da API (sem o campo `nodes`), siga a seção [Tratamento de erros](#tratamento-de-erros).

### Passo 2 — Exibir o resumo

```
Fluxo: <name> (ID: <id>) — <"Ativo" se active=true, senão "Inativo">
Nodes: <quantidade de itens em nodes>
JSON salvo em: <caminho de WORKFLOW_FILE>
```

---

## Fluxo de atualização

Atualiza os nodes e/ou conexões de um workflow existente a partir de uma descrição em linguagem natural das alterações desejadas.

### Passo 1 — Extrair parâmetros

Analise `$ARGUMENTS` (após remover o prefixo `atualizar`). O primeiro token é o ID ou URL do fluxo; o restante do texto é a **descrição das alterações**. Se a descrição estiver vazia, pergunte ao usuário com `AskUserQuestion` o que deseja alterar.

### Passo 2 — Buscar o fluxo e fazer o backup

```bash
N8N_API_KEY="${N8N_API_KEY:?Variável N8N_API_KEY não encontrada}"
N8N_URL="${N8N_URL:?Variável N8N_URL não encontrada}"
BASE_URL="${N8N_URL%/}/api/v1"

WORKFLOW_ID="<id resolvido no Passo 1>"
BACKUP_FILE=$(mktemp --suffix=.json)

curl -s "${BASE_URL}/workflows/${WORKFLOW_ID}" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  -o "$BACKUP_FILE"
```

Se a resposta for um erro da API, siga a seção [Tratamento de erros](#tratamento-de-erros). Guarde o caminho de `BACKUP_FILE` para informar ao usuário no relatório final.

### Passo 3 — Produzir o novo JSON

Copie o backup para um arquivo de trabalho e edite `nodes` e `connections` conforme a descrição das alterações:

- Ao **adicionar** um node, gere um `id` novo (UUID v4) e uma `position` que não sobreponha nodes existentes, e registre as conexões de entrada/saída correspondentes em `connections` (as chaves de `connections` usam o `name` do node de origem).
- Ao **remover** um node, remova também todas as referências a ele em `connections` (tanto como origem quanto como destino).
- Ao **renomear** um node, atualize todas as referências ao nome antigo em `connections` e nas expressões dos demais nodes (ex: `$('Nome do Node')`).

### Passo 4 — Confirmar com o usuário

Exiba um resumo das mudanças — nodes adicionados, removidos e alterados, conexões alteradas e o que permanece igual — e use `AskUserQuestion` com as opções **Atualizar** e **Cancelar**.

Se o usuário cancelar, encerre sem fazer nada.

### Passo 5 — Atualizar via API

Monte o payload conforme as [Regras para escrita na API](#regras-para-escrita-na-api) e envie:

```bash
WORK_FILE="<arquivo de trabalho do Passo 3>"
PAYLOAD_FILE=$(mktemp --suffix=.json)

jq '{name, nodes, connections, settings}' "$WORK_FILE" > "$PAYLOAD_FILE"

curl -s -X PUT "${BASE_URL}/workflows/${WORKFLOW_ID}" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  -H "Content-Type: application/json" \
  --data-binary "@${PAYLOAD_FILE}"

rm -f "$PAYLOAD_FILE"
```

### Passo 6 — Verificar

Faça um novo `GET` do workflow e confira que as mudanças estão presentes (ex: os nodes esperados existem, os removidos não aparecem mais).

### Passo 7 — Reportar o resultado

- O que foi alterado (nodes e conexões).
- O caminho do backup (`BACKUP_FILE`) e como restaurar: `PUT` com o backup filtrado por `jq '{name, nodes, connections, settings}'`.
- O link do editor: `<N8N_URL>/workflow/<id>`.
- Lembrete de que o fluxo **não** foi ativado nem desativado — o estado `active` permanece o que era.

---

## Fluxo de duplicação

Cria uma cópia de um workflow existente (a cópia é criada inativa).

### Passo 1 — Extrair parâmetros

Analise `$ARGUMENTS` (após remover o prefixo `duplicar`). Extraia o ID ou URL do fluxo de origem e o parâmetro `nome:"..."` ou `nome:...`. Se `nome` não for informado, sugira `<nome original> (cópia)` na confirmação do Passo 3.

### Passo 2 — Buscar o fluxo de origem

Execute o mesmo `GET` do fluxo de obtenção do JSON, salvando a resposta em um arquivo temporário. Se a resposta for um erro da API, siga a seção [Tratamento de erros](#tratamento-de-erros).

### Passo 3 — Confirmar com o usuário

Exiba o nome do fluxo de origem e o nome da cópia, e use `AskUserQuestion` com as opções **Duplicar** e **Cancelar**.

### Passo 4 — Criar a cópia via API

```bash
SOURCE_FILE="<arquivo com o JSON de origem>"
PAYLOAD_FILE=$(mktemp --suffix=.json)

jq --arg nome "<nome da cópia>" '{name: $nome, nodes, connections, settings}' "$SOURCE_FILE" > "$PAYLOAD_FILE"

curl -s -X POST "${BASE_URL}/workflows" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  -H "Content-Type: application/json" \
  --data-binary "@${PAYLOAD_FILE}"

rm -f "$PAYLOAD_FILE"
```

### Passo 5 — Reportar o resultado

Exibir o nome, o ID e o link do editor (`<N8N_URL>/workflow/<id do novo fluxo>`) da cópia criada, informando que ela está **inativa**.

---

## Fluxo de ativação/desativação

Ativa ou desativa um workflow existente.

### Passo 1 — Resolver o ID e buscar o nome

Resolva o `WORKFLOW_ID` (após remover o prefixo `ativar` ou `desativar`) e busque o workflow via `GET` para obter o `name` e o estado atual de `active`. Se o fluxo já estiver no estado desejado, informe isso e encerre sem chamar a API novamente.

### Passo 2 — Confirmar com o usuário

Exiba o nome do fluxo e a ação (ativar ou desativar) e use `AskUserQuestion` com as opções **Confirmar** e **Cancelar**.

### Passo 3 — Ativar ou desativar via API

```bash
# Use /activate para ativar ou /deactivate para desativar
curl -s -X POST "${BASE_URL}/workflows/${WORKFLOW_ID}/activate" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}"
```

### Passo 4 — Reportar o resultado

Informar o nome do fluxo e o novo estado (`Ativo` ou `Inativo`), conforme o campo `active` da resposta.

---

## Fluxo de listagem de data tables

### Passo 1 — Extrair filtro opcional

Analise `$ARGUMENTS` (após remover o prefixo `tabelas`) e extraia, se presente, o parâmetro `nome:"..."` ou `nome:...`.

### Passo 2 — Buscar via API

```bash
N8N_API_KEY="${N8N_API_KEY:?Variável N8N_API_KEY não encontrada}"
N8N_URL="${N8N_URL:?Variável N8N_URL não encontrada}"
BASE_URL="${N8N_URL%/}/api/v1"

curl -s "${BASE_URL}/data-tables" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}"
```

### Passo 3 — Exibir o resultado

Para cada item do array `data` (aplicando localmente o filtro `nome`, se informado, com comparação case-insensitive por substring), exibir:

```
- <name> (ID: <id>) — <quantidade de itens em columns> colunas
```

Se nada casar, informar que nenhuma data table foi encontrada. Se a resposta não tiver o campo `data`, siga a seção [Tratamento de erros](#tratamento-de-erros).

---

## Fluxo de exibição de data table

Exibe as colunas e as linhas de uma data table, identificada por ID ou nome.

### Passo 1 — Resolver a tabela

Resolva o `TABLE_ID` (após remover o prefixo `tabela`) conforme [Resolvendo a referência de uma data table](#resolvendo-a-referência-de-uma-data-table).

### Passo 2 — Buscar via API

```bash
curl -s "${BASE_URL}/data-tables/${TABLE_ID}" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}"

curl -s "${BASE_URL}/data-tables/${TABLE_ID}/rows" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}"
```

### Passo 3 — Exibir o resultado

- Cabeçalho: `Tabela: <name> (ID: <id>)`.
- Colunas (do campo `columns`, ordenadas por `index`): `- <name> (<type>)`.
- Linhas (do campo `data` da resposta de `/rows`): exibir como tabela markdown com as colunas da data table (omitir `createdAt`/`updatedAt`; incluir `id`).

Se `data` vier vazio, informar que a tabela não possui linhas. Se a resposta de `/rows` trouxer `nextCursor` preenchido, avisar que há mais linhas além das exibidas.

---

## Fluxo de criação de coluna

Adiciona uma coluna a uma data table existente.

### Passo 1 — Extrair parâmetros e resolver a tabela

Do comando `tabela <id ou nome> coluna <nome>:<tipo>`, extraia o nome e o tipo da coluna. Tipos aceitos: `string`, `number`, `boolean` e `date`. Se o tipo for inválido ou ausente, informar os tipos aceitos e encerrar. Resolva o `TABLE_ID` conforme [Resolvendo a referência de uma data table](#resolvendo-a-referência-de-uma-data-table).

### Passo 2 — Confirmar com o usuário

Exiba o nome da tabela e a coluna a criar (`<nome> (<tipo>)`) e use `AskUserQuestion` com as opções **Criar** e **Cancelar**. Avise que a criação de coluna **não é coberta por backup** — a reversão é excluir a coluna manualmente (ou via `DELETE /data-tables/{id}/columns/{columnId}`).

### Passo 3 — Criar via API

```bash
curl -s -X POST "${BASE_URL}/data-tables/${TABLE_ID}/columns" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"name": "<nome>", "type": "<tipo>"}'
```

### Passo 4 — Reportar o resultado

Sucesso é HTTP 201 com o JSON da coluna criada (campos `id`, `name`, `type`, `index`). Informar nome, tipo e ID da coluna. HTTP 409 indica que já existe coluna com esse nome. Outros erros: seguir a seção [Tratamento de erros](#tratamento-de-erros).

---

## Fluxo de atualização de linhas

Atualiza os valores de linhas de uma data table a partir de uma descrição em linguagem natural.

### Passo 1 — Extrair parâmetros e resolver a tabela

Do comando `tabela <id ou nome> atualizar <descrição>`, resolva o `TABLE_ID` e interprete a descrição. Se a descrição estiver vazia, pergunte ao usuário com `AskUserQuestion` o que deseja alterar.

### Passo 2 — Backup das linhas

Salve o estado atual das linhas em um arquivo temporário e informe o caminho no relatório final:

```bash
ROWS_BACKUP_FILE=$(mktemp --suffix=.json)

curl -s "${BASE_URL}/data-tables/${TABLE_ID}/rows" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  -o "$ROWS_BACKUP_FILE"
```

### Passo 3 — Montar as atualizações

Para cada grupo de linhas a alterar, monte um corpo de `rows/update`: um `filter` que seleciona as linhas (normalmente por uma coluna chave, ex: `sector`) e um `data` com apenas as colunas a alterar:

```json
{
  "filter": {
    "type": "and",
    "filters": [
      { "columnName": "<coluna>", "condition": "eq", "value": "<valor>" }
    ]
  },
  "data": { "<coluna a alterar>": "<novo valor>" },
  "returnData": true
}
```

### Passo 4 — Confirmar com o usuário

Exiba um resumo — quais linhas serão afetadas (filtro) e quais valores serão gravados — e use `AskUserQuestion` com as opções **Atualizar** e **Cancelar**. Se o usuário cancelar, encerre sem fazer nada.

### Passo 5 — Atualizar via API

Para cada corpo montado no Passo 3:

```bash
curl -s -X PATCH "${BASE_URL}/data-tables/${TABLE_ID}/rows/update" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  -H "Content-Type: application/json" \
  --data-binary "@<arquivo com o corpo>"
```

Com `returnData: true`, a resposta é o array das linhas atualizadas — se vier vazio, o filtro não casou com nenhuma linha; avise o usuário.

### Passo 6 — Verificar e reportar

Faça um novo `GET .../rows` e confira que os valores esperados estão gravados. Reportar:

- As linhas e colunas alteradas.
- O caminho do backup (`ROWS_BACKUP_FILE`) — a restauração é reenviar os valores antigos com `rows/update` filtrando pela mesma chave.

---

## Tratamento de erros

A API do n8n retorna um JSON com o campo `message` quando ocorre um erro. Interprete a resposta assim:

- **Mensagem contendo "unauthorized" ou similar**: informar que não foi possível autenticar — verifique se `N8N_API_KEY` está correta e ainda é válida.
- **HTTP 403 "Forbidden" em uma escrita, com os `GET` funcionando**: a chave de API é válida mas não tem escopo de escrita (ex: `workflow:update`, `workflow:create`, `dataTable:*`) — orientar o usuário a ajustar os escopos da chave em **Settings → API** na instância n8n (ou gerar uma nova chave) e encerrar sem retentar.
- **Fluxo não encontrado** (resposta de erro nos fluxos que usam um ID): informar que o fluxo `<id>` não foi encontrado em `<N8N_URL>` — verifique o ID ou a URL informada.
- **Mensagem contendo "must NOT have additional properties"**: o payload contém campos além dos 4 permitidos (`name`, `nodes`, `connections`, `settings`) — refaça o filtro com `jq`. Se o erro apontar para dentro de `settings`, repita mantendo em `settings` somente chaves aceitas pela API (ex: `executionOrder`) e avise o usuário sobre o que foi descartado.
- **Erro no `PUT` após o backup ter sido feito**: informar que o fluxo original permanece intacto no servidor e que o backup local está disponível em `<caminho do backup>`.
- **Falha de conexão do curl** (sem resposta): informar que não foi possível conectar a `<N8N_URL>` — verifique se a URL está correta e acessível.
- **Outros erros**: exibir o campo `message` da resposta da API.

Em todos os casos, encerrar sem tentar novamente automaticamente.
