---
layout: post
title: "logstash: usando o filtro translate"
description: "exemplo de como utilizar o filtro translate no logstash para enriquecimento de dados"
date: 2021-02-16 14:00:00 -0300
lang: pt-br
ref: "p0005"
categories: [ posts, pt-br ]
tags: [ logstash, elastic ]
---
Quando usamos o logstash para processar os dados antes de enviarmos para o destino final, seja ele o elasticsearch ou alguma outra saída, pode ser interessante realizar algum tipo de enriquecimento nos dados para facilitar o processo de análise, monitoramento e filtragem.

Podemos por exemplo substituir informações de códigos de dispositivos por palavras chaves que vão dar uma visão melhor do que está sendo monitorado ou que tipo de informação estamos vendo.

Um filtro do logstash que possibilita realizar esse enriquecimento nos dados é o filtro `translate`.

### o filtro translate

O filtro `translate` funciona de uma forma bem simples, ele verifica se o valor de um campo existe como chave em um dicionário do tipo chave-valor e caso exista ele pode substituir o valor atual do campo pelo valor do dicionário ou criar um novo campo com esse valor.

![0005-01](/img/posts/0005/0005-01.png)

Como exemplo vamos considerar que temos um dispositivo que monitora o número de pessoas que entraram ou saíram de um local, calcula o número médio de pessoas em determinado momento e envia esses dados para um logstash em um intervalo de tempo configurável.

Considerando o seguinte payload como documento:

```json
{
    "storeId": 1,
    "customerIn": 5,
    "customerOut": 2,
    "customerAvg": 10
}
```

Nesse documento temos os seguintes campos:

- `storeId`**:** um identificador do local onde o dispositivo está instalado
- `customerIn`**:** o número de pessoas que entraram no local
- `customerOut`**:** o número de pessoas que saíram do local
- `customerAvg`**:** o número médio de pessoas no local no momento da coleta

Quando temos uma quantidade pequena de dispositivos pode até ser possível lembrar a localização de cada um através do campo `storeId`, mas a medida que o número aumenta, essa tarefa já se torna mais complicada e ter informações extras sobre o dispositivo é algo extremamente útil.

### aplicando o filtro

O principal componente do filtro `translate` é o seu dicionário, que pode tanto estar presente diretamente na configuração do filtro, quanto estar em um arquivo externo.

Quando utilizamos arquivos externos para o dicionário podemos utilizar um arquivo no formato `YAML`, `CSV` ou `JSON`, onde cada item é um par de chave-valor.

Para esse exemplo vamos considerar que nosso dispositivo está localizado em uma loja de lembranças no _Aeroporto Schiphol_ em Amsterdã, que tem como código as letras **AMS**. 

Queremos criar um novo campo chamado `storeName` a partir do valor do campo `storeId`.

```
translate {
	source => "storeId"
	target => "storeName"
	dictionary => {
		"1" => "AMS"
	}
}
```

Quando um evento onde o campo `storeId` tem o valor igual a **1** passar por esse filtro, um novo campo chamado `storeName` com o valor **AMS** será criado.

Considerando agora que temos um novo dispositivo com o `storeId` igual a **2** instalado em uma outra filial da mesma loja localizada no _Aeroporto do Galeão_ no Rio de Janeiro, que tem como código as letras **GIG**, o dicionário do filtro precisa então ter essa nova informação. 

```
dictionary => {
	"1" => "AMS"
	"2" => "GIG"
}
```

A medida que o número de dispositivos aumenta, a opção de usar um dicionário em um arquivo externo se torna mais interessante, nesse caso precisamos apenas criar o arquivo, salvar em um lugar onde o logstash tenha permissão de leitura e configurar o filtro para ler esse arquivo.

```yaml
"1": "AMS"
"2": "GIG"
"3": "CDG"
"4": "BER"
```

Salvando o dicionário acima como `store.yml`, podemos configurar o filtro `translate` para ler desse arquivo e verificar se houve alguma atualização em um intervalo agendado.

```
translate {
	source => "storeId"
	target => "storeName"
	dictionary_path => "/caminho/para/o/arquivo/store.yml"
	refresh_interval => 300
	fallback => "local desconhecido"
}
```

Na configuração acima a opção `refresh_interval` determina o intervalo em segundos que o logstash deve verificar se houve alteração no arquivo e em caso positivo carregar as alterações em memória, a opção `fallback` é utilizada quando o valor do campo `storeId` não é encontrado no dicionário.

Sendo assim:

`PAYLOAD`

```json
{"storeId": 2, "customerIn": 3, "customerOut":2, "customerAvg": 39 }
```

`OUTPUT`

```
{
	"storeId" => 2,
	"@timestamp" => 2021-02-16T06:23:50.312Z,
	"host" => "elk",
	"customerIn" => 3,
	"customerOut" => 2,
	"customerAvg" => 39,
	"storeName" => "GIG",
	"@version" => "1"
}
```
### substituindo o campo original

Podem haver casos onde queremos substituir o valor do campo original e não criar um novo campo, para isso o campo de origem e destino deve ser o mesmo, utilizamos a opção `override` como `true` e desabilitamos a opção de `fallback`, pois caso o valor buscado não exista no dicionário, conseguimos identificar pelo `storeId` e atualizá-lo.

```
translate {
	source => "storeId"
	target => "storeId"
	dictionary_path => "/caminho/para/o/arquivo/store.yml"
	refresh_interval => 300
	override => true
}
```

### um translate ou múltiplos translates?

Em alguns casos podemos também querer adicionar múltiplos campos a partir de um mesmo valor, no nosso exemplo poderíamos adicionar além do nome com o código do Aeroporto, a localização geográfica, o tamanho da loja, o país ou qualquer outro tipo de informação que possa ser útil para alguma análise posterior.

Para isso temos basicamente duas abordagens, a primeira consiste em utilizar um filtro `translate` para cada informação que queremos adicionar e a segunda consiste em utilizar um único filtro `translate` onde o valor associado a chave é um documento `json` com os campos que queremos adicionar, posteriormente o campo adicionado pelo filtro `translate` seria expandido utilizando o filtro `json`.

A primeira abordagem é a mais simples, basta utilizar um arquivo para cada informação que queremos adicionar e utilizar múltiplos filtros `translate` no pipeline.

`EXEMPLO`

```
translate {
	source => "storeId"
	target => "storeName"
	dictionary_path => "/caminho/para/o/arquivo/store-name.yml"
	refresh_interval => 300
	fallback => "local desconhecido"
}

translate {
	source => "storeId"
	target => "storeCity"
	dictionary_path => "/caminho/para/o/arquivo/store-city.yml"
	refresh_interval => 300
	fallback => "cidade desconhecida"
}
```

A segunda abordagem não deixa de ser simples, mas tem alguns pequenos detalhes a mais.

Primeiro temos que garantir que o valor no dicionário é um `json` no formato que o filtro `json` do logstash consiga fazer o _parsing_ sem problemas, para isso basta utilizarmos o formato abaixo.

```
"1": '{ "storeName": "AMS", "storeCity": "Amsterdam", "storeCountry": "Netherlands" }'
"2": '{ "storeName": "GIG", "storeCity": "Rio de Janeiro", "storeCountry": "Brazil" }'
"3": '{ "storeName": "CDG", "storeCity": "Paris", "storeCountry": "France" }'
"4": '{ "storeName": "BER", "storeCity": "Berlin", "storeCountry": "Germany" }'
```

Na sequência, caso a opção `fallback` esteja sendo usada, ela também precisa estar em formato `json`, pois o campo de destino do filtro `translate` será validado por um filtro `json` posteriormente no pipeline.

```
filter {
    translate {
        source => "storeId"
        target => "[@metadata][translate]"
        dictionary_path => "/caminho/para/o/arquivo/store.yml"
        fallback => '{"storeName": "local desconhecido"}'
    }
    json {
    	source => "[@metadata][translate]"
    }
}
```

Na configuração acima estamos salvando o resultado do filtro `translate` em um campo temporário chamado `[@metadata][translate]`, os campos `[@metadata]` podem ser criados nos pipelines e utilizados pelos filtros, mas eles não estão presentes no documento final, além disso a opção `fallback` cria apenas o campo `storeName`.

Após o filtro `translate` utilizamos um filtro `json` tendo como origem o campo temporário criado.

`PAYLOAD`

```json
{"storeId": 1, "customerIn": 2, "customerOut":1, "customerAvg": 15 }
```

`OUTPUT`

```
{
	"storeId" => 1,
	"storeCountry" => "Netherlands",
	"customerAvg" => 15,
	"customerIn" => 2,
	"@version" => "1",
	"storeName" => "AMS",
	"@timestamp" => 2021-02-16T07:19:52.808Z,
	"customerOut" => 1,
	"host" => "elk",
	"storeCity" => "Amsterdam"
}
```
Utilizando o filtro `translate` partimos então de um documento simples.
```json
{
	"storeId": 1,
	"customerIn": 2,
	"customerOut":1,
	"customerAvg": 15
}
```
E chegamos em um documento enriquecido com mais informações.
```json
{
	"storeId": 1, 
	"storeName": "AMS",
	"storeCity": "Amsterdam",
	"storeCountry": "Netherlands",
	"customerIn": 2,
	"customerOut":1, 
	"customerAvg": 15
}
```

### conclusão

Podemos ver que o filtro `translate` nos permite enriquecer os documentos de uma forma bem simples, mas ele tem algumas limitações.

Como os dicionários utilizados pelo `translate` são carregados em memória quando o logstash inicia, o tamanho dos dicionários pode impactar na performance do logstash como um todo. 

A [documentação da elastic][translate-limit] informa que o filtro é testado internamente usando dicionários com cerca de 100.000 pares chave-valor, mas além da quantidade de chaves, o tamanho dos valores também influencia.

Se precisarmos utilizar dicionários muito grandes, talvez seja interessante verificar outras formas de realizar esse enriquecimento nos dados, seja fazendo um pré-processamento antes do dado entrar no logstash, seja utilizando outros filtros que realizam consultas externas como o filtro para consultar o `elasticsearch` ou o `memcached`.

Mais informações sobre o filtro `translate` podem ser encontradas na [documentação oficial][doc-translate].

[translate-limit]: https://www.elastic.co/guide/en/logstash/current/plugins-filters-translate.html#_description_151
[doc-translate]: https://www.elastic.co/guide/en/logstash/current/plugins-filters-translate.html