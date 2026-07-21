---
name: google-sheets
description: Acessa o Google Sheets via chamadas curl na API REST v4. Por enquanto suporta listar as abas de uma planilha e ler os valores de um intervalo, a partir de uma URL ou ID. Requer GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET e GOOGLE_REFRESH_TOKEN configurados (mesmas credenciais da skill google-drive — veja ../google-drive/references/oauth-setup.md).
---

## Como usar

```
/google-sheets abas <url ou ID da planilha>
/google-sheets ler <url ou ID da planilha> [intervalo]
```

Exemplos de `intervalo` (notação A1): `Vendas`, `Vendas!A1:D20`, `A1:C10` (usa a primeira aba).

## Referências

- [Configuração da credencial OAuth 2.0 e obtenção do refresh token](../google-drive/references/oauth-setup.md) — mesmo processo usado pela skill `google-drive`.

## Variáveis de ambiente

| Variável               | Descrição                                                        |
|-------------------------|-------------------------------------------------------------------|
| `GOOGLE_CLIENT_ID`      | Client ID do OAuth 2.0 (tipo "Aplicativo para computador")        |
| `GOOGLE_CLIENT_SECRET`  | Client Secret correspondente                                      |
| `GOOGLE_REFRESH_TOKEN`  | Refresh token obtido uma única vez (ver referência de setup)       |

Se alguma dessas variáveis não estiver definida, informe ao usuário e aponte para [../google-drive/references/oauth-setup.md](../google-drive/references/oauth-setup.md).

**Sobre o escopo:** o escopo `drive.readonly` (usado pela skill `google-drive`) já é aceito pela API do Sheets para leitura de valores, então o mesmo refresh token normalmente funciona sem nenhuma alteração. Se as chamadas abaixo retornarem erro de permissão (403), oriente o usuário a regenerar o refresh token no OAuth Playground incluindo também o escopo `https://www.googleapis.com/auth/spreadsheets.readonly`.

## Instruções de execução

Ao ser invocada, verifique o conteúdo de `$ARGUMENTS`:

1. Se começar com `abas` → **fluxo de listagem de abas**.
2. Se começar com `ler` → **fluxo de leitura de intervalo**.
3. Caso contrário → informe que, por enquanto, a skill só suporta os comandos `abas` e `ler` e mostre os exemplos de uso.

---

## Passo comum — Obter um access token

Antes de qualquer chamada à API do Sheets, troque o refresh token por um access token (válido por ~1 hora):

```bash
GOOGLE_CLIENT_ID="${GOOGLE_CLIENT_ID:?Variável GOOGLE_CLIENT_ID não encontrada}"
GOOGLE_CLIENT_SECRET="${GOOGLE_CLIENT_SECRET:?Variável GOOGLE_CLIENT_SECRET não encontrada}"
GOOGLE_REFRESH_TOKEN="${GOOGLE_REFRESH_TOKEN:?Variável GOOGLE_REFRESH_TOKEN não encontrada}"

TOKEN_RESPONSE=$(curl -s -X POST "https://oauth2.googleapis.com/token" \
  -d client_id="${GOOGLE_CLIENT_ID}" \
  -d client_secret="${GOOGLE_CLIENT_SECRET}" \
  -d refresh_token="${GOOGLE_REFRESH_TOKEN}" \
  -d grant_type=refresh_token)

ACCESS_TOKEN=$(echo "$TOKEN_RESPONSE" | grep -o '"access_token"[[:space:]]*:[[:space:]]*"[^"]*"' | cut -d'"' -f4)
```

Se `ACCESS_TOKEN` vier vazio, exiba o conteúdo de `$TOKEN_RESPONSE` como erro e encerre (provavelmente o refresh token expirou ou foi revogado — oriente o usuário a refazer o passo de autorização em [../google-drive/references/oauth-setup.md](../google-drive/references/oauth-setup.md)).

---

## Passo comum — Extrair o ID da planilha

Analise o argumento recebido (após remover o prefixo `abas`/`ler`). O primeiro token (até o próximo espaço) é a URL ou ID da planilha; o restante da string, se houver, é o intervalo (apenas no fluxo `ler`).

Se for uma URL do Sheets, extraia o ID:

- `https://docs.google.com/spreadsheets/d/<ID>/edit` → `<ID>`
- `https://docs.google.com/spreadsheets/d/<ID>/edit?usp=sharing` → `<ID>` (ignorar querystring)
- `https://docs.google.com/spreadsheets/d/<ID>/edit#gid=123456` → `<ID>` (ignorar o `#gid=...`)

Se não for uma URL, trate o token inteiro como o próprio ID da planilha.

Se nenhum argumento foi informado, use `AskUserQuestion` para solicitar a URL ou ID da planilha.

---

## Fluxo de listagem de abas

### Passo 1 — Extrair o ID da planilha

Ver "Passo comum — Extrair o ID da planilha" acima.

### Passo 2 — Obter o access token

Executar o "Passo comum — Obter um access token" acima.

### Passo 3 — Buscar as abas

```bash
SPREADSHEET_ID="<id extraído no passo 1>"

curl -s -G "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}" \
  --data-urlencode "fields=properties(title),sheets(properties(sheetId,title,gridProperties))" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

### Passo 4 — Exibir o resultado

Se a resposta contiver um campo `error`, exiba a mensagem de erro (ex: planilha não encontrada, sem permissão de acesso) e encerre.

Caso contrário, exiba o título da planilha (`properties.title`) e, para cada item em `sheets`, exibir:

```
- <properties.title> — <gridProperties.rowCount> linhas x <gridProperties.columnCount> colunas
```

---

## Fluxo de leitura de intervalo

### Passo 1 — Extrair o ID da planilha e o intervalo

Ver "Passo comum — Extrair o ID da planilha" acima para o ID.

Se um intervalo foi informado como segundo token do argumento, use-o diretamente (notação A1, ex: `Vendas!A1:D20` ou apenas `Vendas`).

Se nenhum intervalo foi informado, execute o "Passo 3" do fluxo de listagem de abas para descobrir o nome da primeira aba e use esse nome como intervalo (retorna todos os dados preenchidos daquela aba).

### Passo 2 — Obter o access token

Executar o "Passo comum — Obter um access token" acima (pode ser reaproveitado se já obtido no passo anterior).

### Passo 3 — Ler os valores

```bash
SPREADSHEET_ID="<id extraído no passo 1>"
RANGE="<intervalo definido no passo 1>"

curl -s -G "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}/values:batchGet" \
  --data-urlencode "ranges=${RANGE}" \
  --data-urlencode "majorDimension=ROWS" \
  --data-urlencode "valueRenderOption=FORMATTED_VALUE" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

### Passo 4 — Exibir o resultado

Se a resposta contiver um campo `error`, exiba a mensagem de erro (ex: intervalo inválido, planilha não encontrada, sem permissão de acesso) e encerre.

Caso contrário, pegue `valueRanges[0].values` (lista de linhas, cada uma uma lista de células) e monte uma tabela markdown com o conteúdo:

- Se a primeira linha parecer um cabeçalho (valores distintos, texto curto), use-a como cabeçalho da tabela.
- Linhas mais curtas que as demais têm células vazias implícitas no final — preencha com vazio ao montar a tabela.

Se `values` estiver ausente ou vazio, informe que o intervalo não possui dados.
