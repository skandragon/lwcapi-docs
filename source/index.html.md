---
title: API Reference

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

Each retured expression has one or more data expressions.  A client should
compare data at hand to these data expressions, and submit them for
evaluation based on the requested frequency.

### HTTP Request

`GET http://<ELB-ENDPOINT>/lwc/api/v1/expressions/<CLUSTERID>`

### URL Parameters

Parameter | Description
--------- | -----------
CLUSTERID | The cluster-id for this application.

### Example Response

```json
[
  {
    "id": "tQTQF62tsAZCdyC73Rd0IQgZ5Y0",
    "frequency": 60000,
    "dataExpressions": [
      "nf.cluster,skan-test,:eq,name,totalCpu,:eq,:and,:count,(,nf.node,),:by",
      "nf.cluster,skan-test,:eq,name,totalCpu,:eq,:and,:sum,(,nf.node,),:by"
    ]
  }
]
```

Field | Description
----- | -----------
id | Opaque token used to identify the specific expression requested.  This will be sent with evaluation requests.
frequency | How often this data item should be sent to the evaluation endpoint, in milliseconds.
dataExpressions | Specific data expressions, sent as a unit to evaluation requests.  If this is an empty string, it is a placeholder.

It is recommended that a client support specific frequencies that include
at least 10, 30, and 60 seconds.

## Evaluate Expressions

Clients post data periodically to the evaluate endpoint.

### HTTP Request

`POST http://<ELB-ENDPOINT>/lwc/api/v1/evaluate`

### URL Parameters

None

### Example Request

```json
[
  {
    "timestamp": 12345,
    "id": "tQTQF62tsAZCdyC73Rd0IQgZ5Y0",
    "dataExpressions": [
      {"tags": {"nf.app": "skan"}, "value": 123},
      {"tags": {"nf.app": "skan"}, "value": 456}
    ]
  }
]
```

The request body contains one or more expressions.

Field | Description
----- | -----------
id | Opaque token used to identify the specific expression requested.  This will be sent with evaluation requests.
frequency | How often this data item should be sent to the evaluation endpoint, in milliseconds.
dataExpressions | Specific data expressions, sent as a unit to evaluation requests.  If this is an empty string, it is a placeholder.

### Example Response

No response body is returned.

## Stream Request

### HTTP Request

`GET http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to retrieve

### Example Request

### Example Response

## Subscribe To Expressions

### HTTP Request

`GET http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to retrieve

### Example Request

### Example Response
