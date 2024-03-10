---
layout: post
title: "logstash: using the memcached filter"
description: "example of how to use the memcached filter in logstash instead of the translate filter"
date: 2021-04-10 17:00:00 -0300
lang: en
ref: "p0007"
categories: [ posts, en ]
tags: [ logstash, elastic ]
---
In the process of [data enrichment][post-translate] using logstash it is pretty common to use the `translate` filter to add new fields in the documents based on the value of an existing field, to do that we use a `key-value` dictionary and if the key that we are searching exists in the dictionary, the value of this key will be added as a new field.

Although the [`translate`][logstash-translate] filter is very flexible, it has some limitations that could make its use difficult or not possible at all, like for example when we have a very large dictionary or when we have multiple logstash instances running the same pipeline.

As an alternative we can store the dictionary data in a [`memcached`][memcached] instance and make queries using the [`memcached`][logstash-memcached] filter in logstash.

### memcached

_Memcached_ is a `key-value` in-memory database which is commonly used to speed-up web pages, create _cache_ of data and help with the database load of some systems.

It is an open source tool, it is very simple to install and configure and also has managed versions provided by the most common cloud services.

We can insert and query data in _memcached_ using some of the many libraries that exist for a variety of languages, in the case of _logstash_ we do that using the [_memcached_][logstash-memcached] filter.

### the memcached filter

With the [_memcached_][logstash-memcached] filter in logstash we can insert data in a memcached instance using the **SET** option and query data using the **GET** option.

`SET`

To insert a key in _memcached_ we can use the following configuration.

```
filter {
	memcached {
		hosts => ["10.0.1.21"]
		set => {
			"origem" => "exemplo"
		}
	}
}
```

In the option `hosts` we can specify the memcached instance address, which in this example is the `10.0.1.21` server using the default port, `11211`.

In the `set` block we define the field that has the value that we want to store and the name of the key that will store this value, in this case the key that will be created in `memcached` is named `exemplo` and it will have the value from the field `origem`.

`GET`

To query keys in memcached we use the following configuration.

```
filter {
	memcached {
		hosts => ["10.0.1.21"]
		get => {
			"exemplo" => "destino"
		}
	}
}
```

In the `get` block we need to do the opposite that we did in the `set` block, we need to specify the key that we want to search and the name of the field to store the value if we have a positive match, in this case we will look up for the key `exemplo` and store the value in the field `destino`.

### pipeline example

The following pipeline creates a key named `exemplo` inside _memcached_ with the value of the field `origem` and in the sequence we make a query for the same key and store the value in a new field called `destino`.

```
input {
    generator {
        message => "valor exemplo"
        count => 1
    }
}
filter {
    mutate {
        rename => { "message" => "origem" }
    }
    memcached {
        hosts=> ["10.0.1.21"]
        set => {
            "origem" => "exemplo"
        }
    }
    memcached {
        hosts=> ["10.0.1.21"]
        get => {
            "exemplo" => "destino"
        }
    }
}
output {
    stdout {}
}
```

Running this pipeline returns the following output.

```
{
          "host" => "elk",
       "destino" => "valor exemplo",
    "@timestamp" => 2021-04-10T18:30:09.412Z,
      "sequence" => 0,
        "origem" => "valor exemplo",
      "@version" => "1"
}
```
We can see that the value of the field `destino` is the same value of the field `origem`, which was stored in _memcached_.

To confirm that the value is in _memcached_ we can use the ruby library `dalli`, the same one used by the logstash filter, to query the key.

![memcached-ruby](/img/posts/0007/0007-01.gif)

### memcached or translate?

The process of data enrichment using the [`translate`][logstash-translate] filter is very similar to when we use the [`memcached`][logstash-memcached] filter, the choice between one or another is entirelly based in the infrastructure used and the use cases.

One use case where is an advantage to use the _memcached_ filter is when we have a large dictionary, with hundreds of **MB**. If we use the _translate_ filter we probably would need to increase the size of the heap memory of the logstash process and the startup and reload times could cause a great impact in the ingestion process.

Using _memcached_ in this case would allow us to make the process of loading and updating the dictionary completely independent of the logstash process.

Other two use cases where we could use _memcached_ instead of _translate_ are:
- when we have more than one logstash instance/node running the same pipeline and using the same dictionary.
- when we need to create and/or update the dictionary from external tools.

In the first case, when we have more than one logstash instance doing the data enrichment process, using the _translate_ filter implies in having to update the same dictionary file in every instance or create a network share with this file. If instead we use _memcached_ we would need to create and update the dictionary file in one place and would avoid having to create a network share.

![memcached-cluster-logstash](/img/posts/0007/0007-02.jpg)

In the second case, when the dictionary used in the data enrichment process is created or updated by external tools like `python` or `ruby` scripts, using the _translate_ filter would also make necessary to grant read and write permissions to the `.yml` file with the dictionary, which could not be possible in some cases and which would add some security issues.

With _memcached_ we would only need to allow the external tool to connect to the memcached server.

![memcached-external-tool](/img/posts/0007/0007-03.jpg)

In both previous cases, using _memcached_ makes the entire data enrichment process more scalable.

The documentation for the _memcached_ filter can be found in [in link][logstash-memcached].

> This post was originally written in Portuguese and later translated into English, some example field names and messages were kept in Portuguese.

[post-translate]: https://web.leandrojmp.com/posts/en/2021/02/logstash-translate
[memcached]: http://memcached.org
[logstash-memcached]: https://www.elastic.co/guide/en/logstash/current/plugins-filters-memcached.html
[logstash-translate]: https://www.elastic.co/guide/en/logstash/current/plugins-filters-translate.html