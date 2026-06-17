---
name: weather
description: Busca a previsão do tempo com base em uma localização. Se a localização não for fornecida, busca o IP externo e busca a localização deste IP.
---

# Weather

Esta skill busca a previsão do tempo atual e dos próximos dias usando a API gratuita e sem necessidade de chave da [Open-Meteo](https://open-meteo.com/).
A localização da previsão pode ser informada pelo usuário, ou se não informada pode ser obtida a partir do endereço IP.

## Quando usar

Use esta skill sempre que o usuário pedir informações sobre clima/tempo/previsão.

## Passo a passo

### 0. Localização não informada (fallback por IP)

Se o usuário não informar nenhuma localização (ex.: "qual o clima aqui?", "qual o clima na minha localização?"), descubra a localização automaticamente a partir do IP público:

1. Use `WebFetch` para obter o IP público:

   ```
   GET https://checkip.amazonaws.com
   ```

   A resposta é apenas o endereço IP em texto puro.

2. Use `WebFetch` para geolocalizar esse IP:

   ```
   GET https://ipwho.is/{ip}
   ```

   A resposta traz `city`, `region`, `country`, `latitude` e `longitude`. Use `latitude`/`longitude` diretamente no passo 2 (pule o geocoding) e use `city`/`region`/`country` para nomear a localização na resposta final.

3. Avise o usuário, de forma breve, que a localização foi estimada pelo IP e pode não ser exata (ex.: "Estimei sua localização como {cidade} a partir do seu IP — me diga a cidade correta se eu errei.").

Se o usuário informar uma localização explicitamente, ignore este passo e siga para o passo 1.

### 1. Resolver a localização (geocoding)

Se o usuário já forneceu coordenadas (latitude e longitude) ou elas já foram obtidas no passo 0, pule para o passo 2.

Caso contrário, use a ferramenta `WebFetch` para consultar a API de geocodificação da Open-Meteo, convertendo o texto da localização (cidade, bairro, CEP, etc.) em latitude/longitude:

```
GET https://geocoding-api.open-meteo.com/v1/search?name={localizacao}&count=5&language=pt&format=json
```

- `{localizacao}` deve ser a localização informada pelo usuário, com espaços codificados (`%20` ou `+`).
- A resposta traz um array `results`, cada item com `name`, `latitude`, `longitude`, `country`, `admin1` (estado/região), etc.
- Se houver mais de um resultado plausível (ex.: cidades com o mesmo nome em países diferentes), pergunte ao usuário qual delas é a correta antes de continuar, mostrando país/estado de cada opção.
- Se não houver resultados, informe ao usuário que a localização não foi encontrada e peça para reformular (ex.: incluir o estado ou país).

### 2. Buscar a previsão do tempo

Com `latitude` e `longitude` em mãos, use `WebFetch` para consultar a API de previsão:

```
GET https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&current=temperature_2m,relative_humidity_2m,apparent_temperature,precipitation,weather_code,wind_speed_10m&daily=weather_code,temperature_2m_max,temperature_2m_min,precipitation_probability_max&timezone=auto&forecast_days=5
```

Isso retorna:
- `current`: condições no momento (temperatura, sensação térmica, umidade, precipitação, código do tempo, vento).
- `daily`: previsão para os próximos 5 dias (máx/mín de temperatura, probabilidade de precipitação, código do tempo).

### 3. Traduzir os códigos de tempo (weather_code)

A API retorna códigos numéricos (WMO Weather Codes). Traduza-os para descrições em português usando esta tabela:

| Código | Descrição |
|---|---|
| 0 | Céu limpo |
| 1, 2, 3 | Poucas nuvens a nublado |
| 45, 48 | Neblina |
| 51, 53, 55 | Garoa (leve, moderada, intensa) |
| 56, 57 | Garoa congelante |
| 61, 63, 65 | Chuva (leve, moderada, forte) |
| 66, 67 | Chuva congelante |
| 71, 73, 75 | Neve (leve, moderada, forte) |
| 77 | Grãos de neve |
| 80, 81, 82 | Pancadas de chuva (leves, moderadas, violentas) |
| 85, 86 | Pancadas de neve |
| 95 | Tempestade |
| 96, 99 | Tempestade com granizo |

### 4. Apresentar o resultado

Responda ao usuário em português, de forma clara e objetiva, incluindo:

- Nome da localização encontrada (cidade, estado/país).
- Condição atual (descrição + temperatura + sensação térmica).
- Umidade e vento, se relevante.
- Resumo da previsão para os próximos dias (mín/máx e condição), em formato de lista ou tabela.

Exemplo de saída:

```
**Previsão para São Paulo, SP, Brasil**

Agora: Nublado, 22°C (sensação 23°C), umidade 68%, vento 12 km/h

Próximos dias:
- Hoje: 18°C – 25°C, poucas nuvens
- Amanhã: 17°C – 24°C, chuva leve
- ...
```

## Observações

- A API da Open-Meteo é gratuita, não exige chave de API e não tem limites agressivos de uso — ideal para esta skill.
- Sempre que possível, prefira coordenadas precisas vindas do geocoding para evitar ambiguidade.
- Se o usuário pedir explicitamente por unidades imperiais (Fahrenheit, mph), adicione `&temperature_unit=fahrenheit&wind_speed_unit=mph` à URL de previsão.
- A localização por IP (`checkip.amazonaws.com` + `ipwho.is`) é uma estimativa baseada na geolocalização do IP público de saída — pode divergir da localização física real do usuário (ex.: VPN, proxy, rede corporativa). Sempre sinalize isso ao usuário e permita que ele corrija informando a cidade real.
