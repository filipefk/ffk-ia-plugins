# Mercado do Boca - Plugins para Claude Code

Experimentos com plugins e skills para o [Claude Code](https://claude.ai/code).

---

## O que é um Marketplace de plugins do Claude Code?

O Claude Code possui um sistema de **plugins** que permite estender suas capacidades com novas skills (habilidades) criadas por qualquer pessoa ou equipe. Um **Marketplace** é simplesmente um repositório Git que empacota um conjunto de skills prontas para uso, tornando fácil compartilhá-las e instalá-las com um único comando.

### Como funciona

Cada plugin é composto por arquivos `SKILL.md` — documentos em Markdown que ensinam o Claude Code a executar uma tarefa específica. As skills podem fazer chamadas de API, processar texto, interagir com ferramentas externas e muito mais, tudo orquestrado pelo próprio Claude.

---

## Como instalar no Claude Code

### Pré-requisito

1. Você deve ter o git instalado na máquina: https://git-scm.com/install/
2. Você deve ter acesso ao repositório no GitHub https://github.com/filipefk/ffk-ia-plugins e deve tentar conectar e salvar sua autenticação do git no console, para que o Claude use esta mesma autenticação salva. Ex.: `git ls-remote https://github.com/filipefk/ffk-ia-plugins.git`. Se as credenciais ainda não estiverem salvas, o Git solicitará autenticação.
3. Você deve ter o Claude Code CLI instalado e configurado. O CLI é a versão que roda no console.

Obs.: Se você prefere usar o Claude Code Desktop, não tem problema, mas precisará instalar também o Claude Code CLI, pois alguns comandos ainda não funcionam na versão Desktop.

## Instalação

Para adicionar o Marketplace de plugins, abra o Claude Code CLI e execute o comando abaixo dentro dele:

```/plugins marketplace add filipefk/ffk-ia-plugins```

Com o marketplace adicionado, você pode instalar os plugins individualmente. No momento tem só um plugin com duas skills.
Tanto o marketplace quanto os plugins agora vão aparecer na interface do Claude Code para Windows e podem ser gerenciados por lá (instalar, desabilitar, remover...).
O comando abaixo instala o plugin pelo Claude Code CLI

```/plugin install boca-plugin@mercado-do-boca```

## Atualização automática

Claude Code pode atualizar automaticamente marketplaces e seus plugins instalados na inicialização.
Veja no link abaixo as orientações para habilitar e desabilitar a atualização automática via terminal Claude Code CLI (Não encontrei esta opção no programa do Claude Code para Windows).

[Configure atualizações automáticas](https://code.claude.com/docs/pt/discover-plugins#configure-auto-updates)


## Observações

- Projeto de uso pessoal, sem garantias de suporte ou estabilidade.
- Contribuições não são esperadas, mas sugestões são bem-vindas.
- As skills usan chamadas http com `curl` e devem funcionar no Windows, Linux e macOS.

---

## Skills disponíveis

### `azure-card` — Gerenciar work items no Azure DevOps

Cria ou lê um work item no Azure DevOps a partir de uma descrição em linguagem natural. Suporta criação de hierarquias pai/filho, como uma User Story com Tasks filhas.

#### Obtendo as credenciais do Azure DevOps

Para usar a skill, você precisará da URL base do seu projeto e de um token de acesso pessoal (PAT). Siga os passos abaixo para obtê-los:

1. Abra o board do Azure DevOps no seu navegador.
2. Copie a URL do board e separe tudo que estiver à esquerda de `/_boards`, sem a barra. Exemplo: na URL `https://empresa.visualstudio.com/Projeto/_boards/board/t/Area/Equipe` pegue só até `https://empresa.visualstudio.com/Projeto`. Salve esta URL para usar como `AZURE_URL_BOARD`.
3. Abra o menu **"User Settings"** à direita no topo da página e escolha o item **"Personal access tokens"**.
4. Clique em **"New Token"**.
5. Preencha os parâmetros do token e marque **"Read, write, & manage"** na permissão de **"Work Items"** e clique em **"Create"**.
6. Copie e salve o token gerado para usar como `AZURE_USER_API_KEY`.

#### Exemplos de uso

- *Leia o card número 577 e monte um plano para implementação*

- *Crie um card de User Story descrevendo o problema abaixo, analise o código da pasta local e crie cards filho de bug ou task conforme a análise do problema e do código. A quantidade de cards filho deve ser baseada na possibilidade de distribuir o trabalho para mais de um programador sendo que cada card filho possa ser testado e entregue separadamente sem gerar conflitos. A User Story deve ter a descrição do problema e os cards filhos devem ser mais técnicos, sugerindo possíveis causas e soluções baseadas no código analisado:
[descrição livre do problema]*

#### Variáveis de ambiente necessárias

| Variável | Descrição |
|---|---|
| `AZURE_URL_BOARD` | URL base do seu projeto no Azure DevOps (ex: `https://dev.azure.com/org/projeto`) |
| `AZURE_USER_API_KEY` | Token de acesso pessoal (PAT) do Azure DevOps |

Para adicionar como variáveis de ambiente do sistema:

**Windows (PowerShell)**

Só na sessão atual:

```powershell
$env:AZURE_URL_BOARD = "valor"
```

Persistente (usuário atual):

```powershell
[Environment]::SetEnvironmentVariable("AZURE_URL_BOARD", "valor", "User")
```

Persistente (todo o sistema — requer PowerShell como Administrador):

```powershell
[Environment]::SetEnvironmentVariable("AZURE_URL_BOARD", "valor", "Machine")
```

**macOS / Linux (bash/zsh)**

Só na sessão atual:

```bash
export AZURE_URL_BOARD="valor"
```

Persistente (usuário atual — adiciona ao perfil do shell):

```bash
echo 'export AZURE_URL_BOARD="valor"' >> ~/.zshrc   # ou ~/.bashrc, conforme o shell usado
source ~/.zshrc
```

Repita o mesmo processo para a variável `AZURE_USER_API_KEY`.

Talvez seja necessário encerrar o Claude Code e abrir novamente para que ele enxergue as novas variáveis de ambiente.<br>
Para saber se deu certo, solicite ao Claude Code para que ele execute o comando abaixo e ele vai dizer se a variável está configurada ou não<br>
`AZURE_USER_API_KEY="${AZURE_USER_API_KEY:?Variável AZURE_USER_API_KEY não encontrada}"`

---

### `trello-card` — Gerenciar cards no Trello

Cria, lê, atualiza, move e arquiva cards no Trello a partir de linguagem natural. Também gerencia checklists. Board e coluna são resolvidos automaticamente pelo nome.

#### Obtendo as credenciais do Trello

> **Importante:** a `TRELLO_API_KEY` e o `TRELLO_TOKEN` são vinculados a um Workspace do Trello. Se você utiliza mais de um Workspace, as credenciais só vão funcionar para um deles por vez — para usar a skill em outro Workspace, é necessário trocar as credenciais.<br>
> Se você for convidado como membro em um board, não conseguirá gerar as credenciais para este board. Só o administrador do Workspace consegue gerar estas credenciais.

Para usar a skill, você precisará de uma chave de API `TRELLO_API_KEY` e um token `TRELLO_TOKEN` do Trello. Siga os passos abaixo para obtê-los:

##### 1. Criar um aplicativo no Trello

1. Faça login no Trello e acesse [https://trello.com/power-ups/admin/](https://trello.com/power-ups/admin/).
2. Clique no botão **"Novo"** para criar um novo aplicativo.
3. Preencha os campos solicitados e confirme a criação do aplicativo — o campo "URL de conector Iframe" não é obrigatório.
4. Clique no botão **"Gerar nova chave de API"** que aparece após criar o aplicativo.
5. Copie e salve o valor exibido no campo **"Chave de API"**, que é o que você usará como `TRELLO_API_KEY`.

##### 2. Gerar o token de autenticação

Monte a URL abaixo substituindo os conteúdos entre colchetes pelos dados do seu aplicativo (Remova os colchetes):

```
https://trello.com/1/authorize?expiration=never&scope=read,write&response_type=token&name=[nome+do+app]&key=[Chave de API recém criada]
```

> **Atenção:** no parâmetro `name`, substitua espaços por `+` (ex.: `Meu App` → `Meu+App`).

1. Com o Trello aberto no browser, acesse a URL montada acima
2. Leia as permissões solicitadas e clique em **"Permitir"**
3. Copie e salve o token exibido na página seguinte, que será o que você usará como `TRELLO_TOKEN`

#### Variáveis de ambiente necessárias

| Variável | Descrição |
|---|---|
| `TRELLO_API_KEY` | Chave de API do Trello |
| `TRELLO_TOKEN` | Token de autenticação do Trello |

Para adicionar como variáveis de ambiente do sistema:

**Windows (PowerShell)**

Só na sessão atual:

```powershell
$env:TRELLO_API_KEY = "valor"
```

Persistente (usuário atual):

```powershell
[Environment]::SetEnvironmentVariable("TRELLO_API_KEY", "valor", "User")
```

Persistente (todo o sistema — requer PowerShell como Administrador):

```powershell
[Environment]::SetEnvironmentVariable("TRELLO_API_KEY", "valor", "Machine")
```

**macOS / Linux (bash/zsh)**

Só na sessão atual:

```bash
export TRELLO_API_KEY="valor"
```

Persistente (usuário atual — adiciona ao perfil do shell):

```bash
echo 'export TRELLO_API_KEY="valor"' >> ~/.zshrc   # ou ~/.bashrc, conforme o shell usado
source ~/.zshrc
```

Repita o mesmo processo para a variável `TRELLO_TOKEN`.

Talvez seja necessário encerrar o Claude Code e abrir novamente para que ele enxergue as novas variáveis de ambiente.<br>
Para saber se deu certo, solicite ao Claude Code para que ele execute o comando abaixo e ele vai dizer se a variável está configurada ou não:<br>
`TRELLO_API_KEY="${TRELLO_API_KEY:?Variável TRELLO_API_KEY não encontrada}"`

A API do Trello está documentada no link abaixo:<br>
https://developer.atlassian.com/cloud/trello/rest/api-group-actions/

#### Funcionalidades

| Comando | O que faz |
|---|---|
| `/trello-card <descrição>` | Cria um card inferindo tipo, título e descrição |
| `/trello-card board:"..." coluna:"..." <descrição>` | Cria um card em board e coluna específicos |
| `/trello-card board:"..." coluna:"..." due:"2026-07-01" <descrição>` | Cria um card com data de vencimento |
| `/trello-card board:"..." checklist:"..." <descrição>` | Cria um card com checklist e itens inferidos |
| `/trello-card ler <id ou URL>` | Exibe os dados de um card existente |
| `/trello-card listar board:"..." coluna:"..."` | Lista os cards de uma coluna |
| `/trello-card atualizar <id ou URL> <novo conteúdo>` | Atualiza título e/ou descrição de um card |
| `/trello-card mover <id ou URL> board:"..." coluna:"..."` | Move um card para outra coluna |
| `/trello-card arquivar <id ou URL>` | Arquiva um card |
| `/trello-card checklists <id ou URL>` | Lista os checklists de um card com seus itens |
| `/trello-card checklist-item <id-checklist> <texto>` | Adiciona um item a um checklist existente |

#### Exemplos de uso

- *Leia o card https://trello.com/c/IrWu6L3U/4-nome-do-card e monte um plano para implementação*.

- *Analise o fonte da pasta local e crie um card no Trello com a descrição do problema abaixo. A descrição do card deve ser em termos de funcionalidades e deve ser criado um CheckList no card para as tarefas em termos técnicos a serem executadas:
[descrição livre do problema]*

- *Liste os cards da coluna "Em andamento" do board "Meu Projeto".*

- *Mova o card https://trello.com/c/IrWu6L3U/4-nome-do-card para a coluna "Concluído".*

---

### `n8n` — Consultar e editar workflows do n8n

Consulta e edita workflows do n8n via API REST — lista os fluxos existentes, exibe os nodes de um fluxo, obtém o JSON completo, atualiza nodes e conexões, duplica, ativa e desativa fluxos. Também consulta e edita data tables (colunas e linhas). Toda operação de escrita faz backup prévio e pede confirmação do usuário antes de chamar a API.

#### Obtendo as credenciais do n8n

1. Acesse sua instância n8n e abra **Settings → n8n API**.
2. Clique em **"Create an API key"**, dê um nome à chave e defina os escopos — para usar todos os comandos da skill, inclua os escopos de leitura e escrita de workflows e de data tables.
3. Copie e salve a chave gerada, que será o valor de `N8N_API_KEY` (ela não será exibida novamente).

#### Variáveis de ambiente necessárias

| Variável | Descrição |
|---|---|
| `N8N_API_KEY` | Chave de API do n8n |
| `N8N_URL` | URL base da instância n8n, sem o sufixo `/api/v1` (ex: `https://n8n.empresa.com`) |

Para adicionar como variáveis de ambiente do sistema:

**Windows (PowerShell)**

Só na sessão atual:

```powershell
$env:N8N_API_KEY = "valor"
```

Persistente (usuário atual):

```powershell
[Environment]::SetEnvironmentVariable("N8N_API_KEY", "valor", "User")
```

Persistente (todo o sistema — requer PowerShell como Administrador):

```powershell
[Environment]::SetEnvironmentVariable("N8N_API_KEY", "valor", "Machine")
```

**macOS / Linux (bash/zsh)**

Só na sessão atual:

```bash
export N8N_API_KEY="valor"
```

Persistente (usuário atual — adiciona ao perfil do shell):

```bash
echo 'export N8N_API_KEY="valor"' >> ~/.zshrc   # ou ~/.bashrc, conforme o shell usado
source ~/.zshrc
```

Repita o mesmo processo para a variável `N8N_URL`.

Talvez seja necessário encerrar o Claude Code e abrir novamente para que ele enxergue as novas variáveis de ambiente.<br>
Para saber se deu certo, solicite ao Claude Code para que ele execute o comando abaixo e ele vai dizer se a variável está configurada ou não:<br>
`N8N_API_KEY="${N8N_API_KEY:?Variável N8N_API_KEY não encontrada}"`

A API pública do n8n está documentada no link abaixo:<br>
https://docs.n8n.io/api/

#### Funcionalidades

| Comando | O que faz |
|---|---|
| `/n8n listar` | Lista todos os fluxos da instância |
| `/n8n listar nome:"..."` | Lista os fluxos filtrando pelo nome |
| `/n8n listar ativo:true` | Lista os fluxos filtrando pelo estado (ativo/inativo) |
| `/n8n nodos <id ou URL>` | Exibe os nodes de um fluxo |
| `/n8n obter <id ou URL>` | Obtém o JSON completo de um fluxo e salva em arquivo |
| `/n8n atualizar <id ou URL> <descrição>` | Atualiza nodes e conexões de um fluxo (com backup e confirmação) |
| `/n8n duplicar <id ou URL> nome:"..."` | Cria uma cópia inativa de um fluxo |
| `/n8n ativar <id ou URL>` | Ativa um fluxo |
| `/n8n desativar <id ou URL>` | Desativa um fluxo |
| `/n8n tabelas` | Lista as data tables da instância |
| `/n8n tabela <id ou nome>` | Exibe as colunas e linhas de uma data table |
| `/n8n tabela <id ou nome> coluna <nome>:<tipo>` | Adiciona uma coluna a uma data table |
| `/n8n tabela <id ou nome> atualizar <descrição>` | Atualiza linhas de uma data table (com backup e confirmação) |

#### Exemplos de uso

- *Liste os fluxos ativos da minha instância n8n.*

- *Mostre os nodes do fluxo https://n8n.empresa.com/workflow/abc123.*

- *No fluxo "Envio de e-mails", renomeie o node "HTTP Request" para "Consulta CRM" e ajuste as conexões.*

- *Duplique o fluxo abc123 com o nome "Envio de e-mails (homologação)".*

- *Na data table "Setores", atualize a coluna "responsavel" para "Maria" nas linhas em que "sector" for "Financeiro".*

---

### `weather` — Consultar a previsão do tempo

Busca a previsão do tempo atual e dos próximos dias com base em uma localização informada ou, se não informada, estima a localização a partir do IP público (se estiver usando VPN, vai pegar o IP da VPN).

Utiliza a API gratuita [Open-Meteo](https://open-meteo.com/) — sem necessidade de chave de acesso.

#### Exemplos de uso

- *Como está o clima?*

- *Qual a previsão do tempo para Porto Alegre?*
