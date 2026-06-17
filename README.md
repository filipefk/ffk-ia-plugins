# Mercado do Boca - Plugins para Claude Code

Experimentos com plugins e skills para o [Claude Code](https://claude.ai/code).

## Instalação

Este plugin está publicado no marketplace `mercado-do-boca`. Para aparecer o marketplace no Claude Code, execute a linha de comando abaixo dentro do terminal do Claude Code CLI (Não vai funcionar no programa do Claude Code para Windows):

```/plugins marketplace add filipefk/ffk-ia-plugins```

Com o marketplace adicionado, você pode instalar as skills individualmente:

```/plugin install boca-plugin@mercado-do-boca```

## Observações

- Projeto de uso pessoal, sem garantias de suporte ou estabilidade.
- Contribuições não são esperadas, mas sugestões são bem-vindas.

---

## Skills disponíveis

### `azure-card`

Cria ou lê work items no Azure DevOps a partir de descrições em linguagem natural. Suporta criação de hierarquias pai/filho (ex.: User Story com Tasks filhas).
Utiliza scripts PowerShell para interagir com a API do Azure DevOps, então, tem que ter o PowerShell instalado e configurado no ambiente onde o Claude Code está rodando.

Requer as seguintes variáveis de ambiente configuradas:

| Variável | Descrição |
|---|---|
| `AZURE_URL_BOARD` | URL base do board no Azure DevOps |
| `AZURE_USER_API_KEY` | Token de API do usuário |

**Exemplo de uso:**

Crie um card de User Story para "Implementar login" com duas tasks filhas: "Criar endpoint de autenticação" e "Desenvolver interface de login". Analise o fonte da pasta local para extrair os detalhes.

Crie um card de User Story descrevendo o problema abaixo, analise o código da pasta local e crie cards filho de bug ou task conforme a análise do problema e do código. A quantidade de cards filho deve ser baseada na possibilidade de distribuir o trabalho para mais de um programador sendo que cada card filho possa ser testado e entregue separadamente sem gerar conflitos. A User Story deve ter a descrição do problema e os cards filhos devem ser mais técnicos, sugerindo possíveis causas e soluções baseadas no código analisado:
[descrição livre do problema]

---

### `weather`

Busca a previsão do tempo com base em uma localização informada ou, se não informada, estima a localização a partir do IP público.
Faz chamadas GET em serviços gratuítos de previsão do tempo, optenção do IP externo e identificação da localização usando o IP obtido. Se estiver usando VPN vai pegar o IP da VPN.

Utiliza a API gratuita [Open-Meteo](https://open-meteo.com/) — sem necessidade de chave de acesso.

**Exemplo de uso:**

Como está o clima?

Qual a previsão do tempo para Porto Alegre?