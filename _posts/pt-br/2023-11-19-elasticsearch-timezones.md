---
layout: post
title: "elastic stack: corrigindo erros de timezone"
description: "trabalhando com diferentes timezones no elasticsearch, logstash e kibana e resolvendo problemas de timezone, atraso na data"
date: 2023-11-19 18:30:00 -0300
lang: pt-br
ref: "p0009"
categories: [ posts, pt-br ]
tags: [ elasticsearch, elastic ]
---
No Elasticsearch todos os campos do tipo `date` são sempre armazenados em _UTC_.

Além disso, quando usamos o Kibana para visualizar os dados no Elasticsearch, e estamos em uma timezone diferente de _UTC_, o Kibana por padrão irá converter a string de data em _UTC_ para a timezone padrão do navegador.

Quando indexamos documentos onde os campos do tipo `date` não foram convertidos para _UTC_ e não possuem informação sobre a timezone na qual foram gerados, podemos causar uma confusão na hora de visualizar esses dados, já que uma string de data não _UTC_ será interpretada como estando em _UTC_ pelo Elasticsearch e convertida para a timezone do navegador pelo Kibana.

Para evitar esse problema devemos ajustar a string de data original durante a ingestão, convertendo para _UTC_ ou informando a timezone original da string de data.

### String de data sem timezone

Como exemplo vamos considerar que temos uma string de data com as seguintes características.

- foi gerada na timezone _UTC -3_.
- não possui informação sobre a timezone na qual foi gerada.

Nesse caso a string de data `2023-11-19T10:30:00` na timezone _UTC -3_ corresponde a string de data `2023-11-19T13:30:00` em _UTC_, um offset de -3 horas.

Se indexarmos essa string de data sem informar esse offset, ela será indexada como já estando em _UTC_ e ao visualizarmos esse documento no Kibana, em um navegador na timezone _UTC -3_, a string de data original irá sofrer outro offset de -3 horas, sendo exibida como `Nov 19, 2023 @ 07:30:00.000`.

Para exemplificar essa situação usaramos o seguinte template:

```
PUT _index_template/timezone-example
{
  "index_patterns": ["timezone-example"],
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "dateString": {
          "type": "keyword"
        },
        "message": {
          "type": "text"
        }
      }
    }
  }
}
```

E o seguinte documento:

```
POST timezone-example/_doc/
{
  "@timestamp": "2023-11-19T10:30:00",
  "dateString": "2023-11-19T10:30:00",
  "message": "date string example"
}
```

Nesse caso o campo `@timestamp` por ser do tipo `date` será convertido pelo Kibana que está na timezone `UTC-3`, mas o campo `dateString` por ser do tipo `keyword` será tratado como uma string e não será convertido.

[![timezone-error](/img/posts/0009/0009-01.png)](/img/posts/0009/0009-01.png){:target="_blank"}

Usando agora um documento com a string de data convertida para _UTC_, não teremos esse problema no Kibana.

```
POST timezone-example/_doc/
{
  "@timestamp": "2023-11-19T13:30:00",
  "dateString": "2023-11-19T10:30:00",
  "message": "date string example"
}
```

[![timezone-fixed](/img/posts/0009/0009-02.png)](/img/posts/0009/0009-02.png){:target="_blank"}

### String de data com timezone

Quando temos a informação da timezone na String de data podemos verificar que esse problema não ocorre, já que o Elasticsearch converte a data corretamente para _UTC_.

```
POST timezone-example/_doc/
{
  "@timestamp": "2023-11-19T10:30:00-03:00",
  "dateString": "2023-11-19T10:30:00-03:00",
  "message": "date string example with timezone"
}
```

[![timezone-fixed](/img/posts/0009/0009-03.png)](/img/posts/0009/0009-03.png){:target="_blank"}

### Informando a timezone da string de data

A melhor forma de resolver esse problema é informar a timezone diretamente na string de data, mas nem sempre isso possível, nesses casos precisamos informar ao Elasticsearch que a string de data possui um offset de timezone e como isso será feito depende de como a ingestão é feita.


#### Ingestão via Logstash
Quando utilizamos o Logstash podemos usar o filtro [`date`](https://www.elastic.co/guide/en/logstash/current/plugins-filters-date.html) com a opção `timezone`, indicando que a string de data está em uma timezone diferente de _UTC_.

```
filter {
  date {
    match => ["dateString", "yyyy-MM-dd'T'HH:mm:ss]
    target => "@timestamp"
    timezone => "America/Sao_Paulo
  }
}
```

O filtro `date` acima irá fazer o parse do campo `dateString` e validar se o padrão corresponde ao especificado e considerar que a string de data foi gerada na timezone `America/Sao_Paulo`, em caso de sucesso o valor convertido para _UTC_ será armazenado no campo `@timestamp`.

Como exemplo de output temos:

```
{
    "event" => {
      "original" => "2023-11-19T10:30:00"
    },
    "@version" => "1",
    "@timestamp" => 2023-11-19T13:30:00.000Z,
    "dateString" => "2023-11-19T10:30:00"
}

```

O valor para a opção `timezone` precisa estar no formato canônico (`America/Sao_Paulo`) ou no formato numérico (`-0300`).

#### Ingestão via Elasticsearch

Quando estamos enviando os dados diretamente ao Elasticsearch podemos utilizar um _Ingest Pipeline_ com um `date` _processor_ para informar que a string de data está em uma timezone diferente de _UTC_.

```
PUT _ingest/pipeline/parse-date
{
  "description": "parse date field",
  "processors": [
    {
      "date" : {
        "field" : "dateString",
        "target_field" : "@timestamp",
        "formats" : ["yyyy-MM-dd'T'HH:mm:ss"],
        "timezone": "America/Sao_Paulo"
      }
    }
  ]
}
```

O pipeline acima funciona da mesma forma que o filtro `date` do Logstash do exemplo anterior, para utilizar esse pipeline podemos fazer o seguinte request:

```
POST timezone-example/_doc?pipeline=parse-date
{
  "@timestamp": "2023-11-19T10:30:00",
  "dateString": "2023-11-19T10:30:00",
  "message": "date string with ingest pipeline"
}
```

[![timezone-fixed](/img/posts/0009/0009-04.png)](/img/posts/0009/0009-04.png){:target="_blank"}

Podemos também configurar esse _ingest pipeline_ como um _final pipeline_, fazendo com que ele seja sempre executado para qualquer request para esse índice, fazemos isso alterando o template.

```
PUT _index_template/timezone-example
{
  "index_patterns": ["timezone-example"],
  "template": {
    "mappings": {
    "properties": {
        "@timestamp": {
          "type": "date"
        },
        "dateString": {
          "type": "keyword"
        },
        "message": {
          "type": "text"
        }
      }
    },
    "settings": {
      "index.final_pipeline": "parse-date"
    }
  }
}
```

Quando temos uma string de data que foi gerada em uma timezone diferente de _UTC_, é necessário informarmos de alguma forma o offset em relação a _UTC_.