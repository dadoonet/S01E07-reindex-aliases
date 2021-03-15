# Demo scripts used for Elastic Daily Bytes - Aliases and Reindex

![Aliases and Reindex](images/00-talk.png "Aliases and Reindex")

## Setup

The setup will check that Elasticsearch and Kibana are running and will remove any index named `kibana_sample_data_ecommerce*`, `demo-reindex*`, `demo-alias*`, the index templates `demo-reindex` and `demo-alias` and an ingest pipeline named `demo-reindex`.

It will also add Kibana Canvas slides.

### Run on cloud (recommended)

This specific configuration is used to run the demo on a [cloud instance](https://cloud.elastic.co).
You need to create a `.cloud` local file which contains:

```
CLOUD_ID=the_cloud_id_you_can_read_from_cloud_console
CLOUD_PASSWORD=the_generated_elastic_password
```

Run:

```sh
./setup.sh
```

### Run Locally

Run Elastic Stack:

```sh
docker-compose down -v
docker-compose up
```

And run:

```sh
./setup.sh
```

Open Kibana home, and click on the "Add data" and add the Sample eCommerce orders dataset.

## Demo part

### Alias

Run a search and check the number of documents (`4675`)

```
GET /kibana_sample_data_ecommerce/_search?size=0
```

But what if we want to change the mapping and reindex our data.
Or if we want to rebuild entirely our index from the data source if it needs hours or days?

We want our users to be able to switch from an old index to a new one.

That's the power of aliases.

Create a ecommerce alias:

```
PUT /kibana_sample_data_ecommerce/_alias/ecommerce
```

Run a search and check the number of documents (`4675`)

```
GET /ecommerce/_search?size=0
```

Check the aliases

```
GET /_cat/aliases/ecommerce*?v&h=alias,index&s=index
```

We can also create virtual indices which have a built in filter to see only a subset of a given index. Let's have a look at what our documents are looking like:

```
GET /ecommerce/_search?size=1
```

At the end, we can see the `geoip.continent_name`. Let's have a look at the values we have with a `terms` aggregation:

```
GET /ecommerce/_search
{
  "size": 0,
  "aggs": {
    "continent": {
      "terms": {
        "field": "geoip.continent_name"
      }
    }
  }
}
```

Let's create a virtual index for each continent.

```
PUT /kibana_sample_data_ecommerce/_alias/ecommerce_asia
{
  "filter" : {
    "term" : {
      "geoip.continent_name" : "Asia"
    }
  }
}
PUT /kibana_sample_data_ecommerce/_alias/ecommerce_north_america
{
  "filter" : {
    "term" : {
      "geoip.continent_name" : "North America"
    }
  }
}
PUT /kibana_sample_data_ecommerce/_alias/ecommerce_europe
{
  "filter" : {
    "term" : {
      "geoip.continent_name" : "Europe"
    }
  }
}
PUT /kibana_sample_data_ecommerce/_alias/ecommerce_africa
{
  "filter" : {
    "term" : {
      "geoip.continent_name" : "Africa"
    }
  }
}
PUT /kibana_sample_data_ecommerce/_alias/ecommerce_south_america
{
  "filter" : {
    "term" : {
      "geoip.continent_name" : "South America"
    }
  }
}
```

Check the aliases.

```
GET /_cat/aliases/ecommerce*?v&h=alias,index&s=index
```

Search for the **Europe** ecommerce data using the `ecommerce_europe` virtual index. Check the number of documents (`1172`) and the continents (`Europe`).

```
GET /ecommerce_europe/_search
{
  "size": 0,
  "aggs": {
    "continent": {
      "terms": {
        "field": "geoip.continent_name"
      }
    }
  }
}
```

This is extremly powerful because you can also with security have users in Europe only able to see the European dataset by giving them access to the `ecommerce_europe` alias only.

You can also imagine using aliases for time series data.

```
# PUT /demo-alias-2030-12-25/_alias/demo-alias-2030

# Even better with a composable index template
PUT _index_template/demo-alias-2030
{
  "index_patterns": ["demo-alias-2030-*"],
  "template": {
    "mappings": {
      "properties": {
        "foo": { "type": "keyword" },
        "year": { "type": "integer" }
      }
    },
    "aliases": {
      "demo-alias-2030": {
        "filter": {
          "term": { "year": 2030 }
        }
      }
    }
  }
}

# Create a document for 26 Dec 2030
POST /demo-alias-2030-12-26/_doc
{
  "foo": "bar",
  "year": 2030
}

# Search for data in 2030
GET /demo-alias-2030/_search

# Create an alias for the current day
PUT /demo-alias-2030-12-26/_alias/demo-alias-current_day

# Search for data in current day
GET /demo-alias-current_day/_search

# Create a document for 27 Dec 2030
POST /demo-alias-2030-12-27/_doc
{
  "foo": "bar",
  "year": 2030
}

# Search for data in 2030
GET /demo-alias-2030/_search

# Search for data in current day
GET /demo-alias-current_day/_search

# We obviously need to switch the alias
POST /_aliases
{
  "actions" : [
    { "remove" : { "index" : "demo-alias-2030-12-26", "alias" : "demo-alias-current_day" } },
    { "add" : { "index" : "demo-alias-2030-12-27", "alias" : "demo-alias-current_day" } }
  ]
}

# Search for data in current day
GET /demo-alias-current_day/_search
```

### Reindex

Reindex runs a scroll API and a bulk API using the _source for each hit.
So it does not copy any mapping which needs to be done manually.

```
PUT /demo-reindex-ecommerce
{
  "mappings": {
    "properties": {
      "category": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "currency": {
        "type": "keyword"
      },
      "customer_birth_date": {
        "type": "date"
      },
      "customer_first_name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "customer_full_name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "customer_gender": {
        "type": "keyword"
      },
      "customer_id": {
        "type": "keyword"
      },
      "customer_last_name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "customer_phone": {
        "type": "keyword"
      },
      "day_of_week": {
        "type": "keyword"
      },
      "day_of_week_i": {
        "type": "integer"
      },
      "email": {
        "type": "keyword"
      },
      "event": {
        "properties": {
          "dataset": {
            "type": "keyword"
          }
        }
      },
      "geoip": {
        "properties": {
          "city_name": {
            "type": "keyword"
          },
          "continent_name": {
            "type": "keyword"
          },
          "country_iso_code": {
            "type": "keyword"
          },
          "location": {
            "type": "geo_point"
          },
          "region_name": {
            "type": "keyword"
          }
        }
      },
      "manufacturer": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "order_date": {
        "type": "date"
      },
      "order_id": {
        "type": "keyword"
      },
      "products": {
        "properties": {
          "_id": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          },
          "base_price": {
            "type": "half_float"
          },
          "base_unit_price": {
            "type": "half_float"
          },
          "category": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword"
              }
            }
          },
          "created_on": {
            "type": "date"
          },
          "discount_amount": {
            "type": "half_float"
          },
          "discount_percentage": {
            "type": "half_float"
          },
          "manufacturer": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword"
              }
            }
          },
          "min_price": {
            "type": "half_float"
          },
          "price": {
            "type": "half_float"
          },
          "product_id": {
            "type": "long"
          },
          "product_name": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword"
              }
            },
            "analyzer": "english"
          },
          "quantity": {
            "type": "integer"
          },
          "sku": {
            "type": "keyword"
          },
          "tax_amount": {
            "type": "half_float"
          },
          "taxful_price": {
            "type": "half_float"
          },
          "taxless_price": {
            "type": "half_float"
          },
          "unit_discount_amount": {
            "type": "half_float"
          }
        }
      },
      "sku": {
        "type": "keyword"
      },
      "taxful_total_price": {
        "type": "half_float"
      },
      "taxless_total_price": {
        "type": "half_float"
      },
      "total_quantity": {
        "type": "integer"
      },
      "total_unique_products": {
        "type": "integer"
      },
      "type": {
        "type": "keyword"
      },
      "user": {
        "type": "keyword"
      }
    }
  }
}
```

Reindex ecommerce dataset.

```
POST /_reindex?slices=2
{
  "source": {
    "index": "ecommerce"
  },
  "dest": {
    "index": "demo-reindex-ecommerce"
  }
}

GET /demo-reindex-ecommerce/_search
```

You can use an ingest pipeline to modify the `_source` field on the fly:

```
PUT /_ingest/pipeline/demo-reindex
{
  "processors": [
    {
      "remove": {
        "field": [
          "customer_first_name",
          "customer_full_name",
          "customer_gender",
          "customer_id",
          "customer_last_name",
          "customer_phone",
          "email",
          "geoip",
          "products",
          "user",
          "event",
          "sku"
        ]
      }
    }
  ]
}

# Reindex ecommerce dataset (auto slicing = 1 slice per shard)
DELETE /demo-reindex-ecommerce
POST /_reindex?slices=auto
{
  "source": {
    "index": "ecommerce"
  },
  "dest": {
    "index": "demo-reindex-ecommerce",
    "pipeline": "demo-reindex"
  }
}

GET /demo-reindex-ecommerce/_search
```

Once you are happy, switch the alias `ecommerce` and remove the old index.

```
POST /_aliases
{
  "actions" : [
    { "remove" : { "index" : "kibana_sample_data_ecommerce", "alias" : "ecommerce" } },
    { "add" : { "index" : "demo-reindex-ecommerce", "alias" : "ecommerce" } }
  ]
}

GET /ecommerce/_search

DELETE kibana_sample_data_ecommerce
```
