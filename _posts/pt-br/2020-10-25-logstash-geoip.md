---
layout: post
title: "logstash: geolocalização com geoip"
description: "exemplo de como utilizar geolocalização com geoip no logstash e elasticsearch"
date: 2020-10-25 01:00:00 -0300
lang: pt-br
ref: "p0004"
categories: [ posts, pt-br ]
tags: [ logstash, elastic ]
---
Um ponto importante no monitoramento de dispositivos, aplicações ou sistemas expostos na internet, é incluir informações sobre a origem dos acessos ou tentativas de acesso que recebemos.

Isso é útil tanto pelo lado da infraestrutura e segurança quanto pelo lado do negócio. Por exemplo, sabendo a origem e quantidade dos acessos, ou tentativas, podemos planejar melhor a distribuição geográfica de servidores, detectar acessos comprometidos ou tentativas de ataque e identificar novas oportunidades de negócios em localidades diferentes.

Uma forma bem simples de se fazer isso quando temos a pilha elastic como uma das ferramentas de monitoramento é utilizar o filtro `geoip` do logstash.

### o filtro geoip

O filtro `geoip` tem uma função bem simples, ele consulta um endereço IP em uma base interna, identifica a geolocalização do mesmo e retorna alguns campos como o país, o código do país, a cidade, as coordenadas geográficas, o continente e alguns outros.

![geoip](/img/posts/0004/0004-01.png)

No caso do filtro `geoip` utilizado pelo logstash, a base interna onde as informações de geolocalização são consultadas é a `GeoLite2`, fornecida pela [maxmind][maxmind], e ao utilizarmos o filtro em um endereço IP é possível obter, além das informações geográficas, as informações referentes ao Sistema Autônomo (_AS_) associadas ao roteamento.

Para utilizar o filtro `geoip` você precisa que o seu evento tenha um campo onde o valor é um endereço IP público e precisa também criar um mapeamento específico no seu índice para armazenar campos com informações de geolocalização.

### aplicando o filtro

Como exemplo para aplicação do filtro `geoip` vou utilizar uma [API simples][go-sysmon] que desenvolvi em Go e que retorna o estado das conexões de uma máquina linux, emulando parte do funcionamento do _netstat_.

Quando consultada a API retorna um documento JSON no seguinte formato.

```json
{
    "srcip":"10.0.1.100",
    "srcport":56954,
    "dstip":"151.101.192.133",
    "dstport":443,
    "status":"ESTABLISHED"
}
```

O campo `srcip` corresponde ao IP local da máquina e o campo `dstip` corresponde ao IP externo da conexão, é esse campo que iremos utilizar com o filtro `geoip`.

Para consumir essa API com o logstash, vou utilizar como `input` o filtro `http_poller`, que basicamente fica fazendo requisições para um _endpoint_ configurado em um intervalo específico de tempo.

```
input {
    http_poller {
        urls => { "api" => "http://10.0.1.100:5000/netstat" } 
        schedule => { "every" => "30s"}
    }
}
```

>  A forma como os dados são recebidos pelo logstash não faz diferença pro filtro `geoip`, você só precisa de um campo com um IP público.

A configuração mais simples do filtro [`geoip`][geoip] exige apenas uma opção obrigatória, `source`, que define qual é o campo com o IP que queremos descobrir a geolocalização.

```
filter {
	geoip {
		source => "dstip"
	}
}
```

Quando o evento passa por esse filtro com sucesso, um novo campo chamado `geoip` é adicionado ao evento.

```
"geoip" => {
    "country_code2" => "IS",
    "continent_code" => "EU",
    "city_name" => "Reykjavik",
    "country_code3" => "IS",
    "region_code" => "1",
    "timezone" => "Atlantic/Reykjavik",
    "region_name" => "Capital Region",
    "location" => {
        "lon" => -21.9466,
        "lat" => 64.1432
    },
    "latitude" => 64.1432,
    "ip" => "31.209.137.10",
    "country_name" => "Iceland",
    "postal_code" => "101",
    "longitude" => -21.9466
}
```

Em caso de falha, a tag `_geoip_lookup_failure` será adicionada.

Podemos alterar esse comportamento padrão utilizando outras opções na configuração do filtro.

- `target`: nome do campo para salvar as informações de geolocalização, o padrão é o campo `geoip`
- `default_database_type`: tem apenas duas opções `City` e `ASN`, a primeira é o padrão e traz informações geográficas, a segunda traz informações do Sistema Autônomo (_AS_) associado.
- `fields`: os campos que serão retornados, por padrão traz todos os disponíveis.
- `tag_on_failure`: o nome da tag que será adicionada ao evento em caso de falha, por padrão é `_geoip_lookup_failure`.

Como a ideia é enriquecer os eventos, vamos utilizar dois filtros `geoip` na sequência, um com a opção `default_database_type` igual a `City` e outro com a mesma opção configurada como `ASN`.

```
filter {
	geoip {
		default_database_type => "City"
		source => "dstip"
		target => "geo"
		tag_on_failure => ["geoip-city-failed"]
	}
	geoip {
		default_database_type => "ASN"
		source => "dstip"
		target => "geo"
		tag_on_failure => ["geoip-asn-failed"]
	}
}
```

### filtrando endereços privados

Quando o campo que vamos utilizar como origem para o filtro `geoip` pode ter também endereços IPs privados, precisamos filtrar esses IPs de alguma forma.

Um jeito simples de evitar que o IPs privados passem pelo filtro `geoip` é utilizar um condicional para adicionar uma _tag_ nos eventos que tem IP privado e um outro condicional para limitar a aplicação do filtro `geoip` somente aos eventos sem essa tag.

Nesse exemplo será preciso filtrar a rede `10.0.1.0/24`, o IP de localhost `127.0.0.1` e o IP de roteamento local `0.0.0.0`.

```
if [dstip] =~ "^10.0.*" or [dstip] =~ "^127.0.*" or [dstip] == "0.0.0.0" { 
	mutate {
		add_tag => ["internal"]
	}
}
```

Agora todos os eventos que tem como destino um IP privado terão a _tag_  `internal` e podemos utilizar essa `tag` para evitar que eles passem pelo filtro de `geoip`.

```
if "internal" not in [tags] {
	geoip {
		default_database_type => "City"
		source => "dstip"
		target => "geo"
		tag_on_failure => ["geoip-city-failed"]
	}
	geoip {
		default_database_type => "ASN"
		source => "dstip"
		target => "geo"
		tag_on_failure => ["geoip-asn-failed"]
	}
}
```

### mapeamento

Antes de podermos analisar os dados no elasticsearch e criarmos mapas no kibana, é preciso criar um mapeamento para indicar os tipos de dados de cada campo.

Embora o elasticsearch consiga identificar e criar um mapeamento na hora da ingestão, isso não é suficiente para os campos de geolocalização, esses campos precisam estar definidos antes da criação do índice.

A parte obrigatória do mapeamento é a que define o campo `geo.location` como sendo do tipo `geo_point`, portanto se utilizarmos um índice chamado `endpoints`, poderíamos apenas criar o índice e aplicar o mapamento para esse campo.

```
PUT /endpoints
PUT /endpoints/_mapping
{
    "properties": {
        "geo": {
            "properties": {
                "location": {
                    "type": "geo_point"
                }
            }
        }
    }
}
```

Dessa forma garantimos que o campo `geo.location` será do tipo `geo_point` e deixaremos que o elasticsearch crie o mapeamento dos outros campos quando for indexar o primeiro documento.

Embora não seja nenhum problema trabalhar assim, o ideal é criar um template para o nosso índice já definindo os tipo para cada campo.

No nosso examplo poderíamos utilizar o seguinte template.

```
PUT _template/endpoints
{
    "order" : 0,
    "version" : 1,
    "index_patterns" : [ "endpoints" ],
    "settings" : {
      "index" : {
        "mapping" : {
          "ignore_malformed" : "true"
        },
        "refresh_interval" : "5s",
        "number_of_shards" : "1",
        "number_of_replicas" : "0"
      }
    },
    "mappings" : {
        "properties" : {
            "@timestamp" : {
                "format" : "strict_date_optional_time||epoch_millis",
                "type" : "date"
            },
            "@version" : { "type" : "keyword" },
            "status" : { "type" : "keyword" },
            "srcip" : { "type" : "ip" },
            "srcport" : { "type" : "keyword" },
            "dstip" : { "type" : "ip" },
            "dstport" : { "type" : "keyword" },
            "geo" : {
                "properties" : {
                    "as_org" : { "type" : "keyword" },
                    "asn" : { "type" : "keyword" },
                    "country_code2" : { "type" : "keyword" },
                    "country_code3" : { "type" : "keyword" },
                    "country_name" : { "type" : "keyword" },
                    "continent_code" : { "type" : "keyword" },
                    "city_name" : { "type" : "keyword" },
                    "region_code" : { "type" : "keyword" },
                    "region_name" : { "type" : "keyword" },
                    "postal_code" : { "type" : "keyword" },
                    "ip" : { "type" : "ip" },
                    "location" : { "type" : "geo_point" },
                    "latitude" : { "type" : "float" },
                    "longitude" : { "type" : "float" },
                    "timezone" : { "type" : "keyword" }
                }
            },
            "message" : { "type" : "text" },
            "tags" : { "type" : "keyword"}
        }
    }
}
```

> Como o template só é aplicado durante a criação do índice, caso o índice tenha sido criado antes, é preciso deletá-lo com o request `DELETE endpoints`

Com o template criado, podemos adicionar o output para o elasticsearch no pipeline.

```
output {
    elasticsearch {
        hosts => ["http://elk:9200"]
        index => "endpoints"
    }
}
```

### visualizando os dados e criando mapas

Assim que iniciamos o logstash com o pipeline configurado, os dados começarão a ser coletados e enviados para o elasticsearch, após a criação do _index pattern_, podemos visualizar os dados já com a gelocalização no kibana.

![kibana discovery](/img/posts/0004/0004-02.png)

Enquanto o nosso evento inicial trazia apenas a informação do IP de destino, utilizando o filtro de `geoip` conseguimos enriquecer o documento adicionando informações relacionadas a esse IP de destino.

O campo `geo.country_name` é um exemplo de informação gerada pelo filtro `geoip` com a opção `default_database_type` como `City` e o campo `geo.as_org` é um exemplo de informação gerada quando usamos `default_database_type` como `ASN`, esse é o motivo para termos utilizado dois filtros `geoip` no pipeline.

Podemos ainda visualizar graficamente esses dados utilizando a ferramenta _Maps_ dentro kibana.

![mapa geoip](/img/posts/0004/0004-03.gif)

Se antes no nosso monitoramento tínhamos somente a informação do IP de destino, agora temos uma ideia de onde está esse destino e podemos visualizá-lo em um mapa.

![mapa zoom](/img/posts/0004/0004-04.png)

Um ponto importante que devemos considerar é que a precisão das coordenadas obtidas pelo filtro `geoip` não é exata e pode variar de acordo com a localização geográfica, sendo mais precisas em alguns países, o tipo de conexão, se é um IP de uma conexão fixa ou móvel, e o nível de zoom.

Esse [link][precision] no site da [maxmind][maxmind] permite ter uma estimativa da precisão em cada caso.

> O resultado do filtro geoip nunca deve ser considerado como exato e pode não corresponder com a realidade.

Para mais informações sobre o filtro `geoip`, você pode consultar a [documentação oficial][geoip] da elastic.

[maxmind]: https://dev.maxmind.com/geoip/geoip2/geolite2/
[go-sysmon]: https://github.com/leandrojmp/go-sysmon
[http_poller]: https://www.elastic.co/guide/en/logstash/current/plugins-inputs-http_poller.html
[geoip]: https://www.elastic.co/guide/en/logstash/current/plugins-filters-geoip.html

[precision]: https://www.maxmind.com/en/geoip2-city-accuracy-comparison