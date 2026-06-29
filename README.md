# Mercado do Boca - Plugins para Claude Code

Experimentos com plugins e skills para o [Claude Code](https://claude.ai/code).

## Instalação

Este plugin está publicado no marketplace `mercado-do-boca`. Para aparecer o marketplace no Claude Code, execute a linha de comando abaixo dentro do terminal do Claude Code CLI (Não vai funcionar no programa do Claude Code para Windows):

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

---

## Skills disponíveis

### `azure-card`

Cria ou lê work items no Azure DevOps a partir de descrições em linguagem natural. Suporta criação de hierarquias pai/filho (ex.: User Story com Tasks filhas).
Utiliza scripts PowerShell para interagir com a API do Azure DevOps, então, tem que ter o PowerShell instalado e configurado no ambiente onde o Claude Code está rodando.

#### Obtendo as credenciais do Azure DevOps

Para usar a skill, você precisará da URL base do seu projeto e de um token de acesso pessoal (PAT). Siga os passos abaixo para obtê-los:

1. Abra o board do Azure DevOps no seu navegador.
2. Copie a URL do board e separe tudo que estiver à esquerda de `/_boards`, sem a barra. Exemplo: na URL `https://empresa.visualstudio.com/Projeto/_boards/board/t/Area/Equipe` pegue só até `https://empresa.visualstudio.com/Projeto`. Salve esta URL para usar como `AZURE_URL_BOARD`.
3. Abra o menu **"User Settings"** à direita no topo da página e escolha o item **"Personal access tokens"**.
4. Clique em **"New Token"**.
5. Preencha os parâmetros do token e marque **"Read, write, & manage"** na permissão de **"Work Items"** e clique em **"Create"**.
6. Copie e salve o token gerado para usar como `AZURE_USER_API_KEY`.

#### Variáveis de ambiente necessárias

| Variável | Descrição |
|---|---|
| `AZURE_URL_BOARD` | URL base do board no Azure DevOps |
| `AZURE_USER_API_KEY` | Token de API do usuário |

Configure-as no Claude Code com os comandos abaixo ou adicione manualmente como variáveis de ambiente do computador:

```bash
claude config set env.AZURE_URL_BOARD "https://dev.azure.com/sua-org/seu-projeto"
claude config set env.AZURE_USER_API_KEY "seu-token-aqui"
```

**Exemplo de uso:**

Crie um card de User Story para "Implementar login" com duas tasks filhas: "Criar endpoint de autenticação" e "Desenvolver interface de login". Analise o fonte da pasta local para extrair os detalhes.

Crie um card de User Story descrevendo o problema abaixo, analise o código da pasta local e crie cards filho de bug ou task conforme a análise do problema e do código. A quantidade de cards filho deve ser baseada na possibilidade de distribuir o trabalho para mais de um programador sendo que cada card filho possa ser testado e entregue separadamente sem gerar conflitos. A User Story deve ter a descrição do problema e os cards filhos devem ser mais técnicos, sugerindo possíveis causas e soluções baseadas no código analisado:
[descrição livre do problema]

---

### `trello-card`

Cria, lê, atualiza, move, arquiva e lista cards no Trello a partir de linguagem natural. Também gerencia checklists. Board e coluna são resolvidos por nome.

#### Obtendo as credenciais do Trello

Para usar a skill, você precisará de uma chave de API `TRELLO_API_KEY` e um token `TRELLO_TOKEN` do Trello. Siga os passos abaixo para obtê-los:

##### 1. Criar um aplicativo no Trello

1. Faça login no Trello e acesse [https://trello.com/power-ups/admin/](https://trello.com/power-ups/admin/).
2. Clique no botão **"Novo"** para criar um novo aplicativo.
3. Preencha os campos solicitados e confirme a criação do aplicativo — o campo "URL de conector Iframe" não é obrigatório.
4. Clique no botão **"Gerar nova chave de API"** que aparece após criar o aplicativo.
5. Copie e salve o valor exibido no campo **"Chave de API"**, que é o que você usará como `TRELLO_API_KEY`.

##### 2. Gerar o token de autenticação

Monte a URL abaixo substituindo os conteúdos entre colchetes pelos dados do seu aplicativo:

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
| `TRELLO_API_KEY` | API Key da conta Trello (obtida em https://trello.com/app-key) |
| `TRELLO_TOKEN` | Token de acesso OAuth do Trello |

Configure-as no Claude Code com os comandos abaixo ou adicione manualmente como variáveis de ambiente do computador:

```bash
claude config set env.TRELLO_API_KEY "sua-chave-aqui"
claude config set env.TRELLO_TOKEN "seu-token-aqui"
```

**Exemplos de uso:**

```
/trello-card board:"Meu Projeto" coluna:"To Do" Implementar tela de login
/trello-card board:"Meu Projeto" coluna:"To Do" due:"2026-07-01" Corrigir bug no checkout
/trello-card board:"Meu Projeto" checklist:"Tarefas" Nova feature com subtarefas
/trello-card ler https://trello.com/c/IrWu6L3U/4-nome-do-card
/trello-card listar board:"Meu Projeto" coluna:"Em andamento"
/trello-card atualizar <id ou URL> nova descrição
/trello-card mover <id ou URL> board:"Meu Projeto" coluna:"Concluído"
/trello-card arquivar <id ou URL>
/trello-card checklists <id ou URL>
/trello-card checklist-item <id-checklist> Texto do novo item
```

---

### `weather`

Busca a previsão do tempo com base em uma localização informada ou, se não informada, estima a localização a partir do IP público.
Faz chamadas GET em serviços gratuítos de previsão do tempo, optenção do IP externo e identificação da localização usando o IP obtido. Se estiver usando VPN vai pegar o IP da VPN.

Utiliza a API gratuita [Open-Meteo](https://open-meteo.com/) — sem necessidade de chave de acesso.

**Exemplo de uso:**

Como está o clima?

Qual a previsão do tempo para Porto Alegre?
