# Modelos de cards (HTML)

O Azure DevOps renderiza HTML nos campos de texto. Use os modelos abaixo como base.

## User Story — campo `System.Description`

```html
<p><b>Como</b> [ator inferido], <b>quero</b> [ação principal] <b>para</b> [benefício].</p>
<h3>Critérios de Aceite</h3>
<ul>
  <li>[critério 1 extraído da descrição]</li>
  <li>[critério 2, se houver]</li>
</ul>
```

## Bug — campo `Microsoft.VSTS.TCM.ReproSteps`

> Cards de Bug **não têm** o campo "Description". Use `Microsoft.VSTS.TCM.ReproSteps` para toda a descrição.

```html
<h3>Comportamento Atual</h3>
<p>[o que está acontecendo de errado, conforme descrito]</p>
<h3>Comportamento Esperado</h3>
<p>[o que deveria acontecer]</p>
```

## Task — campo `System.Description`

```html
<h3>Objetivo</h3>
<p>[o que deve ser feito]</p>
<h3>Detalhes</h3>
<p>[informações relevantes da descrição]</p>
```
