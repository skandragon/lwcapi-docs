---
title: LWCAPI API Reference

language_tabs:
  - json

toc_footers:
  - <a href='https://github.com/tripit/slate'>Documentation Powered by Slate</a>

includes:

search: true
---

# Introduction

atlas-lwcapi is a RESTful stateless service to allow applications which do
not use the Java JVM publish statistics in a streaming manner to Atlas, Mantis,
or any receiver which can process an SSE stream.

# LWCAPI

## Retrieve Client Expressions

Clients should periodically retrieve a list of which expressions are
specifically subscribed by an active listener.

### HTTP Request

`GET http://<ELB-ENDPOINT>/lwc/api/v1/expressions/<CLUSTERID>`

### URL Parameters

Parameter | Description
--------- | -----------
CLUSTERID | The cluster-id for this application.

### Example Response

```json
{
  "destinations": {
    "default": "http://...",
    "publish-proxy": "http://..."
  },
  "frequency": 60000,
  "expressions": [
    {
      "id": "tQTQF62tsAZCdyC73Rd0IQgZ5Y0",
      "frequency": 10000,
      "expression": "nf.cluster,skan-test,:eq,name,totalCpu,:eq,:and,:count,(,someMetric,),:by"
    }, {
      "id": "asdkqlkfktjrflsjflkwjflkjkd",
      "expression": "nf.cluster,skan-test,:eq,name,totalCpu,:eq,:and,:sum",
      "destination": "publish-proxy"
    }
  ]
}
```

Field | Description
----- | -----------
destinations | A key/value list of destinations to send data for expressions.
id | Opaque token used to identify the specific expression requested.  This will be sent with evaluation requests.
frequency | How often this data item should be sent to the evaluation endpoint, in milliseconds.  If an expression does not provide a value, use the top-level value.
expressions | Specific data expressions for the client to send.
destination | The named destination to send data for this expression.  If not specified, use the "default" endpoint.

## Evaluate Expressions

Clients post data periodically to the evaluate endpoint.

### HTTP Request

`POST http://<ELB-ENDPOINT>/lwc/api/v1/evaluate`

<aside class="notice">
 Note: The actual URL specified in the client expression list should be used.
</aside>

### URL Parameters

None

### Example Request

```json
{
  "timestamp": 12345,
  "expressions": [
    {
      "id": "tQTQF62tsAZCdyC73Rd0IQgZ5Y0",
      "data": [
        {"tags": {"someMetric": "value1"}, "value": 123},
        {"tags": {"someMetric": "value2"}, "value": 456}
      ]
    }, {
      "id": "asdkqlkfktjrflsjflkwjflkjkd",
      "data": [
        {"tags": {}, "value": 123}
      ]
    }
  ]
}
```

Data may be partitioned by sending multiple payloads with expression objects.  A single expression object may also be
split into multiple payloads if needed.

Field | Description
----- | -----------
timestamp | The timestamp to apply to all the expressions in this payload.
expressions | Specific data expressions, sent as a unit to evaluation requests.  If this is an empty string, it is a placeholder.
id | Opaque token used to identify the specific expression requested.  This field may be repeated if needed to partition a large return set.

### Example Response

No response body is returned.

## Stream Request

Request an SSE stream of subscribed-to data.

Stream IDs should be durable for a given consumer's lifetime.  Only one connection per ID is allowed to each instance.
If the same ID is re-used, it will cause the older connection to be dropped.

For instance, a Mantis job may choose to use stream ID "myStreamId1" for each connection made.  If a connection is lost, or a
new lwcapi instance is detected, the same ID should be used.  However, if the mantis job were to restart or be replaced by a
newer version, it should use a different ID.

If the `expr` field is provided, it will be automatically subscribed to, using the optionally provided `frequency`.
If more than one expression is needed, addiitonal "Subscribe To Expression" calls may be made to add them.  Note that
this process will need to be repeated on connection loss: Stream Request followed by any additional Subscribe to Expressions
requests.

### HTTP Request

`GET http://<INSTANCE-ENDPOINT>/lwc/api/v1/stream/<ID>?...`

<aside class="notice">
 Note:  This connection should be made to each instance, not to the ELB endpoint.
</aside>

### Query Parameters

Parameter | Description
--------- | -----------
name | A human-readable name for this stream, such as "alert group foo" or some other helpful information.
expr | An Atlas stack expression to subscribe to.
frequency | If expr is provided, request that the client report this often, in milliseconds.

### Example Response

...

## Subscribe To Expressions

### HTTP Request

`GET http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to retrieve

### Example Request

### Example Response
