# Configuração da credencial OAuth 2.0 (setup único, feito pelo usuário)

Esta skill não executa este fluxo sozinha — ele é feito manualmente uma única vez pelo usuário no navegador, fora do Claude Code. Depois de concluído, as três variáveis de ambiente ficam configuradas e a skill funciona sem intervenção.

## Por que uma credencial nova (e não a do n8n)

O Client ID/Secret usado no n8n é válido, mas o OAuth client cadastrado no Google Cloud só aceita os redirect URIs que estiverem na lista dele (provavelmente só o callback do n8n). Para gerar um refresh token novo pra esta skill é preciso um redirect URI diferente. Criar uma credencial dedicada evita qualquer alteração na credencial de produção do n8n.

## Passo 1 — Criar o OAuth Client no Google Cloud Console

1. Acesse https://console.cloud.google.com/apis/credentials no mesmo projeto usado pelo n8n (ou em outro, se preferir isolar).
2. Se ainda não houver, configure a "Tela de consentimento OAuth" (OAuth consent screen) com o escopo `https://www.googleapis.com/auth/drive.readonly`.
3. Em "Criar credenciais" → "ID do cliente OAuth", escolha o tipo **"Aplicativo da Web"** (Web application).
   > Não use "Aplicativo para computador" (Desktop app): esse tipo só aceita redirect URIs de loopback automáticos e não permite cadastrar o redirect do OAuth Playground, causando o erro `redirect_uri_mismatch`.
4. Em **"URIs de redirecionamento autorizados"**, adicione: `https://developers.google.com/oauthplayground`
5. Anote o **Client ID** e o **Client Secret** gerados.

## Passo 2 — Habilitar a Google Drive API

Em https://console.cloud.google.com/apis/library, busque "Google Drive API" e clique em "Ativar" (se ainda não estiver ativa para o projeto).

## Passo 3 — Gerar o refresh token via OAuth Playground

1. Acesse https://developers.google.com/oauthplayground
2. Clique no ícone de engrenagem (canto superior direito) e marque **"Use your own OAuth credentials"**. Informe o Client ID e Client Secret do Passo 1.
3. Na lista de escopos à esquerda, informe manualmente: `https://www.googleapis.com/auth/drive.readonly`
4. Clique em **"Authorize APIs"** e faça login com a conta Google dona dos arquivos/pastas do Drive que serão acessados.
5. Na tela seguinte ("Step 2"), clique em **"Exchange authorization code for tokens"**.
6. Copie o valor de **Refresh token** exibido.

## Passo 4 — Configurar as variáveis de ambiente

Defina as três variáveis no ambiente onde o Claude Code roda (ex: `.env`, variáveis de sistema do Windows, ou perfil do PowerShell):

```
GOOGLE_CLIENT_ID=<client id do passo 1>
GOOGLE_CLIENT_SECRET=<client secret do passo 1>
GOOGLE_REFRESH_TOKEN=<refresh token do passo 3>
```

## Observações

- O refresh token não expira por tempo, mas é revogado se o usuário remover o acesso do app em https://myaccount.google.com/permissions, se ficar 6 meses sem uso, ou se o OAuth consent screen estiver em modo "Testing" e o usuário for removido da lista de testadores.
- Caso o app esteja em modo "Testing" (comum para uso pessoal/interno), o refresh token expira em 7 dias. Para uso contínuo, publique o app na tela de consentimento (não precisa de verificação do Google se o escopo for só `drive.readonly` e o uso for restrito).
- Para acessar outras pastas/arquivos, a conta autorizada no Passo 3 precisa ter permissão de leitura sobre eles no Drive.
