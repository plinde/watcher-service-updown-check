# watcher-service-updown-check
Inspired by https://discuss.elastic.co/t/website-up-down-status/104758/3

#### Setup

##### 1. Configure Heartbeat

In this example, we'll use Python's built-in HTTP server to demonstrate a simple up/down state of an HTTP service.

>https://www.elastic.co/products/beats/heartbeat


```
heartbeat.yml:
...
# Configure monitors
heartbeat.monitors:
- type: http

  # List or urls to query
  urls: ["http://localhost:8000"]

  # Configure task schedule
  schedule: '@every 10s'

  # Total test connection and data exchange timeout
  timeout: 5s
...
```

>https://github.com/elastic/beats/blob/master/heartbeat/heartbeat.yml#L10-L24

`./heartbeat -e -c ./heartbeat.yml`


We can also configure the HTTP monitor to check for HTTP status code, response payload, etc.

>https://github.com/elastic/beats/blob/master/heartbeat/heartbeat.reference.yml#L196-L206


##### 2. Start the Python HTTP Server on TCP 8000

```
$ python -m SimpleHTTPServer 8000
Serving HTTP on 0.0.0.0 port 8000 ...
```

##### 3. Confirm the up/down monitor is functioning as expected.

We should Heartbeat's stdout writing messages every 10s that resemble this.
```
{"@timestamp":"2017-11-20T14:45:52.934Z","beat":{"hostname":"plinde-mbp.local","name":"plinde-mbp.local",
"version":"5.6.4"},"duration":{"us":835},"host":"localhost","http_rtt":{"us":1148},"ip":"127.0.0.1",
"monitor":"http@http://localhost:8000","port":8000,"resolve_rtt":{"us":226},"response":{"status":200},
"rtt":{"us":1530},"scheme":"http","tcp_connect_rtt":{"us":296},"type":"http","up":true,"url":"http://localhost:8000"}
```

If we stop the Python HTTP server, we should see the heartbeat monitor mention a connection refused on the service.
```
{"@timestamp":"2017-11-20T14:47:45.240Z","beat":{"hostname":"plinde-mbp.local","name":"plinde-mbp.local",
"version":"5.6.4"},"duration":{"us":324},"error":{"message":"Get http://localhost:8000: dial tcp 127.0.0.1:8000: getsockopt: connection refused","type":"io"},"host":"localhost","ip":"127.0.0.1","monitor":"http@http://localhost:8000",
"port":8000,"resolve_rtt":{"us":145},"scheme":"http","tcp_connect_rtt":{"us":107},"type":"http","up":false,"url":"http://localhost:8000"}
```

Start the service again to see the HTTP 200 OK response.
```
{"@timestamp":"2017-11-20T14:47:55.241Z","beat":{"hostname":"plinde-mbp.local","name":"plinde-mbp.local",
"version":"5.6.4"},"duration":{"us":1835},"host":"localhost","http_rtt":{"us":1148},"ip":"127.0.0.1",
"monitor":"http@http://localhost:8000","port":8000,"resolve_rtt":{"us":226},"response":{"status":200},
"rtt":{"us":1530},"scheme":"http","tcp_connect_rtt":{"us":296},"type":"http","up":true,"url":"http://localhost:8000"}
```

##### 4. Configure Elasticsearch output in Heartbeat.

Configure the Elasticsearch output
>https://github.com/elastic/beats/blob/master/heartbeat/heartbeat.yml#L89-L98

Or in the case of Elastic Cloud, the Cloud ID can be used
>https://www.elastic.co/guide/en/cloud/current/ec-cloud-id.html
>https://github.com/elastic/beats/blob/master/heartbeat/heartbeat.yml#L72-L84


```
GET _cat/indices/heartbeat*
health status index                uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   heartbeat-2017.11.20 ArEpxqLeS1S9ocjXHCsa7A   5   1         28            0    158.9kb        158.9kb

```


##### 5. Configure the watch in X-Pack Monitoring (Watcher)

In this example, we'll use a chained Input to execute two Elasticsearch queries; "first", which queries the previously marked state of the service and "second", which queries the most recent datapoint containing the service's latest state from Heartbeat.
>https://www.elastic.co/guide/en/x-pack/current/input-chain.html

We'll want to modify the Actions to fit our environment. Wwe use two webhooks pointing at another Python HTTP Server for easy debugging purposes, but this can be anything (Slack, HipChat, Email, etc).


```
{
  "trigger": {
    "schedule": {
      "interval": "10s"
    }
  },
  "input": {
    "chain": {
      "inputs": [
        {
          "first": {
            "search": {
              "request": {
                "search_type": "query_then_fetch",
                "indices": [
                  "state"
                ],
                "types": [
                  "services"
                ],
                "body": {
                  "query": {
                    "bool": {
                      "must": {
                        "match_all": {}
                      }
                    }
                  }
                }
              }
            }
          }
        },
        {
          "second": {
            "search": {
              "request": {
                "search_type": "query_then_fetch",
                "indices": [
                  "heartbeat-*"
                ],
                "types": [],
                "body": {
                  "query": {
                    "bool": {
                      "must": {
                        "match_all": {}
                      },
                      "filter": {
                        "range": {
                          "@timestamp": {
                            "from": "now-20s",
                            "to": "now"
                          }
                        }
                      }
                    }
                  },
                  "size": 1,
                  "sort": [
                    {
                      "@timestamp": {
                        "order": "desc"
                      }
                    }
                  ],
                  "_source": [
                    "up"
                  ]
                }
              }
            }
          }
        }
      ]
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.second.hits.total": {
        "gt": 0
      }
    }
  },
  "actions": {
    "webbook_sendstate": {
      "throttle_period_in_millis": 10000,
      "webhook": {
        "scheme": "http",
        "host": "notification-service.mydomain.com",
        "port": 12345,
        "method": "get",
        "path": "/services/service1/state/{{ ctx.payload.second.hits.hits.0._source.up }}",
        "params": {},
        "headers": {}
      }
    },
    "webhook_sendstatechange": {
      "throttle_period_in_millis": 10000,
      "condition": {
        "compare": {
          "ctx.payload.second.hits.hits.0._source.up": {
            "not_eq": "{{ctx.payload.first.hits.hits.0._source.up}}"
          }
        }
      },
      "webhook": {
        "scheme": "http",
        "host": "notification-service.mydomain.com",
        "port": 12345,
        "method": "get",
        "path": "/services/service1/state-change/{{ ctx.payload.second.hits.hits.0._source.up }}",
        "params": {},
        "headers": {}
      }
    },
    "index_writebackstate": {
      "condition": {
        "compare": {
          "ctx.payload.second.hits.total": {
            "gt": 0
          }
        }
      },
      "transform": {
        "script": {
          "source": "return [ 'up' : ctx.payload.second.hits.hits.0._source.up ]",
          "lang": "painless"
        }
      },
      "index": {
        "index": "state",
        "doc_type": "services",
        "doc_id": "1"
      }
    }
  }
}
```

#### Limitations
This example is a relatively naive approach which only suppports the monitoring of a single up/down state. However, it should be trivial to modify the input query and index action (index_writebackstate) to target an individual service if there are multiple services being monitored by Heartbeat.