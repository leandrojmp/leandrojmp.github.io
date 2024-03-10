---
layout: post
title: "elastic stack: fixing timezone issues"
description: "working with different timezones on elasticsearch, logstash and kibana and fixing timezone issues, date offset"
date: 2023-11-19 18:30:00 -0300
lang: en
ref: "p0009"
categories: [ posts, en ]
tags: [ elasticsearch, elastic ]
---
In Elasticsearch all `date` type fields are always stored as _UTC_.

Besides that, when we use Kibana to visualize the data in Elasticsearch, and we are in a different timezone from _UTC_, Kibana per default will convert the date string in _UTC_ to the default browser timezone.

When we index documents where the `date` type fields weren't converted to _UTC_ and do not have any information about the timezone where they were generated, this can lead to some confusion in the data visualization since the non _UTC_ date string will be interpreted as an _UTC_ date string by Elasticsearch and converted to the browser timezone by Kibana.

To avoid this issue we should fix the original date string during ingestion, converting it to _UTC_ or informing the original date string timezone to Elasticsearch.

### Date string without timezone information

As an example, let's consider that we have a date string with the following characteristics.

- It was generated on _UTC -3_ timezone.
- It does not have any information about the timezone where it was generated.

In this case, the date string `2023-11-19T10:30:00` at _UTC -3_ timezone corresponds to the date string `2023-11-19T13:30:00` at _UTC_, an offset of -3 hours.

If we index this date string without providing any information about this offset, this date will be indexed as an _UTC_ date string and when we look at this document on Kibana, on a browser at the _UTC -3_ timezone, another offset of -3 hours will be applied and the date will appear as `Nov 19, 2023 @ 07:30:00.000`.


As an example of this scenario, let's use the following template:

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

And the following document:

```
POST timezone-example/_doc/
{
  "@timestamp": "2023-11-19T10:30:00",
  "dateString": "2023-11-19T10:30:00",
  "message": "date string example"
}
```

On this case, since the field `@timestamp` is a `date` type field, it will be converted by Kibana, which is in _UTC -3_, but the field `dateString` is a `keyword` field, so no convertion will be applied to this field as it is a string.

[![timezone-error](/img/posts/0009/0009-01.png)](/img/posts/0009/0009-01.png){:target="_blank"}

If we use a document where the date string was previously converted to _UTC_, we will not have this issue on Kibana.

```
POST timezone-example/_doc/
{
  "@timestamp": "2023-11-19T13:30:00",
  "dateString": "2023-11-19T10:30:00",
  "message": "date string example"
}
```

[![timezone-fixed](/img/posts/0009/0009-02.png)](/img/posts/0009/0009-02.png){:target="_blank"}

### Date string with timezone

When we have the timezone offset information in the date string, we can see that this issue does not happen as Elasticsearch will correctly convert the date to _UTC_.

```
POST timezone-example/_doc/
{
  "@timestamp": "2023-11-19T10:30:00-03:00",
  "dateString": "2023-11-19T10:30:00-03:00",
  "message": "date string example with timezone"
}
```

[![timezone-fixed](/img/posts/0009/0009-03.png)](/img/posts/0009/0009-03.png){:target="_blank"}

### Setting the timezone offset in the Date String

The best approach to deal with this issue is to have the timezone offset directly in the date string, but this is not always possible, in these cases we need a way to inform Elasticsearch about the timezone offset and how we do that depends on how we are ingesting the data.


#### Ingesting data using Logstash

When we are using Logstash to ingest our data, we can use the [`date`](https://www.elastic.co/guide/en/logstash/current/plugins-filters-date.html) filter with the `timezone` option, which indicates that the date string is in a different timezone from _UTC_.

```
filter {
  date {
    match => ["dateString", "yyyy-MM-dd'T'HH:mm:ss]
    target => "@timestamp"
    timezone => "America/Sao_Paulo
  }
}
```

The previous `date` filter will parse the `dateString` field and validate if the pattern matchs the one configured in the `match` option, it will also convert this date to the timezone specified in the `timezone` option and in the case of a success match it will store the converted _UTC_ value in the `@timestamp` field.

As an output we have:

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

The value for the `timezone` option needs to be in the canonical format (`America/Sao_Paulo`) or in a numerical offset format (`-0300`).

#### Ingesting data using Elasticsearch

When we are sending our data directly to Elasticsearch, we can use an _Ingest Pipeline_ with a `date` _processor_ to informat that the date string is on a different timezone from _UTC_.

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

The above pipeline works in the same way as the `date` filter in Logstash from the previous example, to use this pipeline we can make the following request:

```
POST timezone-example/_doc?pipeline=parse-date
{
  "@timestamp": "2023-11-19T10:30:00",
  "dateString": "2023-11-19T10:30:00",
  "message": "date string with ingest pipeline"
}
```

[![timezone-fixed](/img/posts/0009/0009-04.png)](/img/posts/0009/0009-04.png){:target="_blank"}

And we can also set this _ingest pipeline_ as a _final pipeline_, which will make it be executed for every request to this index, we do that changing the index template.

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

When we have a date string that was generated on a different timezone from _UTC_, it is important that we inform Elasticsearch about this offset.