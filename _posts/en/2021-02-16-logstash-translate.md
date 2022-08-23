---
layout: post
title: "logstash: using the translate filter"
description: "example of how to enrich data using the translate filter in logstash"
date: 2021-02-16 14:00:00 -0300
lang: en
ref: "p0005"
categories: [ posts, en ]
tags: [ logstash, elastic ]
cover: /img/posts/covers/0005.png
---
When we use logstash to process data before sending it to the final destination, be it elasticsearch or some other output, it may be useful to perform some kind of data enrichment to help with the analysis, monitoring or filtering.

For example, we can replace some device code with a string that will give a better view of what is being monitored or what type of information we are seeing.

A logstash filter that allows us to do this data enrichment is the `translate` filter.

### the translate filter

The `translate` filter works in a very simple way, it checks if the value of a field exists as a key in a given key-value dictionary and if it exists it can replace the current value of the field with the dictionary value or create a new field with the dictionary value.

![0005-01](/img/posts/0005/0005-01.png)

As an example, let's consider that we have a device that monitors the number of people entering or leaving a location, calculates the average number of people at any given time and sends this data to a logstash according to a configurable schedule.

Considering the following payload as a document:

```json
{
    "storeId": 1,
    "customerIn": 5,
    "customerOut": 2,
    "customerAvg": 10
}
```

In this document we have the following fields:

- `storeId`**:** the id of the location where the device is installed
- `customerIn`**:** the number of customers entering the location
- `customerOut`**:** the number of customers leaving the location
- `customerAvg`**:** the average number of customers in the location when the data was acquired

When we have a small number of devices it may be possible to remember the location of each one using the `storeId` field, but as this number increases, this task becomes more complicated and having extra information about the devices is extremely useful .

### applying the filter

The main component of the `translate` filter is its dictionary, which can either be directly in the filter configuration or in an external file.

When we have our dictionary in an external file, we can use a `YAML`, `CSV` or `JSON` file, where each item is a key-value pair.

For this example, let's consider that our device is located in a gift shop store at the _Schiphol Airport_ in Amsterdam, which has the code **AMS**.

We want to create a new field named `storeName` based on the value of the field `storeId`.

```
translate {
	source => "storeId"
	target => "storeName"
	dictionary => {
		"1" => "AMS"
	}
}
```

When an event where the field `storeId` has the value equal to **1** pass through this filter, a new field named `storeName` with the value **AMS** will be created.

Let's consider now that we have a new device where the `storeId` field has the value equal to **2** in another store of the same gift shop chain located at the _Galeão Airport_ in Rio de Janeiro, which has the code **GIG**, the filter dictionary needs to be updated with this new information.

```
dictionary => {
	"1" => "AMS"
	"2" => "GIG"
}
```

As the number of devices grows, having the dictionary in an external file becomes more practical, in this case we just need to create the file, save it in a path where logstash has read permissions and configure the filter to read this file.

```yaml
"1": "AMS"
"2": "GIG"
"3": "CDG"
"4": "BER"
```

Saving the above dictionary as `store.yml`, we can configure the `translate` filter to read this file and check for updates at a scheduled interval.

```
translate {
	source => "storeId"
	target => "storeName"
	dictionary_path => "/path/to/the/file/store.yml"
	refresh_interval => 300
	fallback => "unknown place"
}
```

In the above configuration the option `refresh_interval` sets the interval in seconds that logstash should check the file for changes and load the changes into memory, the option `fallback` is used when the value of the field `storeId` isn't found in the dictionary.

As an example we have a `payload` and an `output`.

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
### replacing the original field

There may be cases where we want to replace the value of the original field instead of creating a new field, to do that we need to use the same field as source and destination, set the `override` option as `true` and disable the `fallback` option, because if the search value does not exist in the dictionary, we can use the `storeId` original value to identify the device and update the dictionary.

```
translate {
	source => "storeId"
	target => "storeId"
	dictionary_path => "/path/to/the/file/store.yml"
	refresh_interval => 300
	override => true
}
```

### one translate or multiple translates? 

In some cases we may also want to add multiple fields based on the same original value, in our example besides the name of the store we could also add the location, the size of the store, the country or any other kind of information that could be useful for a posterior analysis.

We can approach this use case in two ways, the first one is to use one `translate` filter for each new information that we want to add, the second one is to use a single `translate` filter where the value in the key-value pair on the dictionary is a `json` document with all the fields that we want to add, later we would use the `json` filter to parse this field.

The first way is very simple, we just need to use one dictionary file for each information that we want to add and use multiple `translate` filters in our pipeline.

`EXAMPLE`

```
translate {
	source => "storeId"
	target => "storeName"
	dictionary_path => "/path/to/the/file/store-name.yml"
	refresh_interval => 300
	fallback => "unknown place"
}

translate {
	source => "storeId"
	target => "storeCity"
	dictionary_path => "/path/to/the/file/store-city.yml"
	refresh_interval => 300
	fallback => "unknown city"
}
```

The second way is also very simple, but it has some small details.

First we need to make sure that the value in the dictionary is a `json` on a format that the `json` filter in logstash will be able to parse without any problems, to do that we just need to use the format below.

```
"1": '{ "storeName": "AMS", "storeCity": "Amsterdam", "storeCountry": "Netherlands" }'
"2": '{ "storeName": "GIG", "storeCity": "Rio de Janeiro", "storeCountry": "Brazil" }'
"3": '{ "storeName": "CDG", "storeCity": "Paris", "storeCountry": "France" }'
"4": '{ "storeName": "BER", "storeCity": "Berlin", "storeCountry": "Germany" }'
```

Next, if the option `fallback` is being used, we also need to make sure that its value is a `json` in the same format, because the destination field from the `translate` filter will be validated by a `json` filter later in the pipeline.

```
filter {
    translate {
        source => "storeId"
        target => "[@metadata][translate]"
        dictionary_path => "/path/to/the/file/store.yml"
        fallback => '{"storeName": "unknown place"}'
    }
    json {
    	source => "[@metadata][translate]"
    }
}
```

In the above configuration we are storing the result of the `translate` filter in a temporary field named `[@metadata][translate]`. The `[@metadata]` fields can be created and used by the filters in the pipeline, but they are not present in the final document, we also have the `fallback` option creating only the field `storeName`.

After the `translate` filter we use a `json` filter where the source is the temporary field.

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
Using the `translate` filter we went from have a simple document.

```json
{
	"storeId": 1,
	"customerIn": 2,
	"customerOut":1,
	"customerAvg": 15
}
```
To have an enriched document with more information.
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

### conclusion

We can see that the `translate` filter allows us to enrich the documents in a simple way, but it has some limitations.

Since the dictionaries used by the `translate` filter are loaded into memory when logstash starts, the size of the dictionaries can impact in the overall performance of the system.

The [elastic documentation][translate-limit] says that the filter is tested internally using dictionaries with 100.000 key-values pairs, but besides the number of keys, the size of the values could also have an impact in the performance.

If we need to use very large dictionaries, maybe it is more interesting to see if there are other ways to do the data enrichment process, be that with some pre-processing before logstash or be that using other filters to make external queries like the `elasticsearch` filter or the `memcached` filter.

More information about the `translate` filter can be found in the [official documentation][doc-translate].

[translate-limit]: https://www.elastic.co/guide/en/logstash/current/plugins-filters-translate.html#_description_151
[doc-translate]: https://www.elastic.co/guide/en/logstash/current/plugins-filters-translate.html