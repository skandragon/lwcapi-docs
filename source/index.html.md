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
  "url": "http://...",
  "frequency": 60000,
  "expressions": [
    {
      "id": "tQTQF62tsAZCdyC73Rd0IQgZ5Y0",
      "expression": "nf.cluster,skan-test,:eq,name,totalCpu,:eq,:and,:count,(,someMetric,),:by",
      "frequency": 10000
    }, {
      "id": "asdkqlkfktjrflsjflkwjflkjkd",
      "expression": "nf.cluster,skan-test,:eq,name,totalCpu,:eq,:and,:sum",
      "url": "http://..."
    }
  ]
}
```

Field | Description
----- | -----------
url | The default URL for all expressions, or specific per-expression overrides.
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
data | An array of data which partially or completely provides data for an expression.
tags | tag/value pairs which allow the expression to be processed.
value | A single floating-point value

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

Applications should use Discovery to find all lwcapi instances in their region and stack, and connect to each.

<aside class="notice">
 Note:  This connection should be made to each instance, not to the ELB endpoint.
</aside>

### Query Parameters

Parameter | Description
--------- | -----------
name | A human-readable name for this stream, such as "alert group foo" or some other helpful information.
expr | An Atlas stack expression to subscribe to.
frequency | If expr is provided, request that the client report this often, in milliseconds.

### Stream Contents

The response is a continous stream of housekeeping and expression evaluation data.

All responses are SSE events, beginning with `data:`, an event type, and JSON formatted event type-specific data.
Events are separated by a blank line.

<aside class="notice">
  Note: While all the JSON examples here are multi-line, the data events sent to the stream are all on one line.
</aside>

#### data: hello

```json
{
  "streamId": "yourStreamId",
  "instanceId": "i-3259129e91e",
  "instanceUUID": "55037c3f-600a-4153-8a70-55ef2150bbfe"
}
```

Field | Description
----- | -----------
streamId | The stream ID, echoed from the URL.
instanceId | The instanceId serving this data stream.  This is for debugging.
instanceUUID | The instance's current UUID.  This is for debugging.

#### data: heartbeat

A heartbeat is periodically sent to ensure connections are not idle forever.

```json
{}
```

Currently this is an empty JSON object response.

#### data: subscribe

This event is sent once for each subscription.

```json
{
  "expression": "name,foo,:eq,nf.app,skan,:eq,:and,:avg",
  "frequency": 60000,
  "dataExpressions": [
    {
      "id": "aldjalkdsjalksjdalskdj",
      "expression": "name,foo:eq,nf.app,skan,:eq,:and,:count"
    }, {
      "id": "523ljdlk2j3dlkj23lkjdk",
      "expression": "name,foo:eq,nf.app,skan,:eq,:and,:sum"
    }
  ]
}
```

Field | Description
----- | -----------
requestedExpression | The expression subscribed to.
frequency | The frequency, in milliseconds, this expression will be reported.
expressions | A list of data expressions generated from the requestedExpression.
id | The identifier which each data expression will be identified by.
expression | The data expression itself.

When data arives, it is identified by the `id` field.  See `evaluate` for more information.

#### data: evaluate

...

## Subscribe To Expressions

### HTTP Request

`POST http://<INSTANCE-ENDPOINT>/lwc/api/v1/subscribe`

### Example Request

```json
{
  "streamId": "foo",
  "frequency": 60000,
  "url": "http://...",
  "expressions": [
    {
      "expression": "nf.cluster,skan-test,:eq,name,totalCpu,:eq,:and,:avg",
      "frequency": 10000,
      "url": "http://..."
    }
  ]
}
```

Field | Description
----- | -----------
streamId | The streamId to snd data for these expressions to.
frequency | How often clients are requested to send this data, in milliseconds.  If omitted, a default is used.  If specified at the top level, it supplies a default for all expressions.
url | The URL the client should use to submit data.  This is generally unneeded, and should not be used without consulting the Atlas team.
