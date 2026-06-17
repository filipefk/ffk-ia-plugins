# Regras de criação de cards

## Tipo do work item

Inferir com base na descrição:

| Tipo | Quando usar |
|------|-------------|
| `Bug` | Problema, falha, erro ou comportamento incorreto |
| `User Story` | Funcionalidade do ponto de vista do usuário ou negócio |
| `Task` | Demais casos: tarefa técnica, melhoria, ajuste, etc. |

## Título

Frase curta e objetiva (máximo 80 caracteres) resumindo a essência da descrição, seguida obrigatoriamente do sufixo ` (Criado pelo Claude Code)`.

Exemplo: `Corrigir login inválido (Criado pelo Claude Code)`.

## Campo de descrição por tipo

- **User Story** e **Task**: usar o campo `System.Description`
- **Bug**: usar o campo `Microsoft.VSTS.TCM.ReproSteps` (campo "Repro Steps")
  - Cards de Bug **não possuem** o campo "Description" no Azure DevOps
  - Todo o conteúdo descritivo do bug deve ir em `Microsoft.VSTS.TCM.ReproSteps`

## Formato HTML

O Azure DevOps renderiza HTML nos campos de texto. Use HTML simples conforme os modelos em [card-models.md](card-models.md).

## Confirmação antes de criar

Sempre exibir o card montado para confirmação usando `AskUserQuestion` com as opções **Criar card** e **Cancelar** antes de chamar a API. Se cancelar, encerrar sem fazer nada.

## Exibição do card para confirmação

```
Tipo:    <tipo>
Título:  <título>

Descrição:
<conteúdo em texto legível, sem tags HTML>
```
