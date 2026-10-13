---
title: Work IQ MCP tool reference
description: Reference documentation for all tools exposed by the Work IQ MCP server, including parameters and usage examples.
author: jasonjoh
ms.author: jasonjoh
ms.topic: reference
ms.localizationpriority: medium
ms.date: 10/12/2026
---

<!-- markdownlint-disable MD024 -->
<!-- cSpell:ignore bizchat bbkr enddatetime eykd SNWE startdatetime Uplg FdvcmtJUSE -->

# Work IQ MCP tool reference

This article describes the tools exposed by the Work IQ MCP server through the [Model Context Protocol (MCP)](https://modelcontextprotocol.io), including their purposes, parameters, and usage examples.

Tool availability and advertised schemas can vary by supported release and caller. Use MCP `tools/list`, following any `nextCursor` values, to discover the complete toolset for your connection. Use the advertised `inputSchema` when constructing a request. The `retrieve` tool and some discovery options are available only to enabled audiences; their presence in this reference doesn't guarantee availability in every tenant.

Work IQ MCP uses Microsoft Entra authentication and the signed-in user's Microsoft 365 permissions when its tools access Microsoft Graph resource paths. A separate tenant policy layer evaluates each tool request and blocks mutation operations by default. Administrators can enable supported create, update, delete, and action requests through tenant policy. For details, see [Policy governance for Work IQ MCP](policy-governance-mcp.md).

Unless an example explicitly includes `content` or `structuredContent`, its response shows the payload inside `structuredContent`, not the complete MCP tool result. Text-only responses are identified in their tool sections. Check the tool result's `isError` flag before treating a response as successful; an error doesn't necessarily use the success payload's shape.

## Entity tools

Entity tools provide CRUD operations and actions on Microsoft 365 resources. These tools operate on relative resource paths that map to [Microsoft Graph](/graph/overview) endpoints.

For `create_entity`, `update_entity`, and `do_action`, a body such as `"jsonBody": {"subject": "Hello world!"}` can also be supplied as `"jsonBody": "{\"subject\":\"Hello world!\"}"`. Both forms represent the same entity or action body. Use the form permitted by your client's advertised input schema.

### fetch

Reads one or more entities by resource path. Supports fetching multiple paths in parallel.

#### Parameters

| Parameter    | Type         | Required | Description                                                                                 |
|--------------|--------------|----------|---------------------------------------------------------------------------------------------|
| `entityUrls` | string array | Yes      | The relative resource paths to fetch (for example, `/me/messages`, `/me/events/{event-id}`) |
| `agentId`    | string       | No       | Reserved for future use.                                                                    |

#### Response

A successful response contains a `results` property that is an array of objects with the following properties. There's one object for each resource path in the `entityUrls` parameter of the request.

| Property     | Type    | Description                                  |
|--------------|---------|----------------------------------------------|
| `data`       | object  | The JSON object returned by Microsoft Graph. |
| `statusCode` | integer | The HTTP status code for the response.       |

> [!NOTE]
>
> - Per-tenant policy limits apply to collection results. If you don't specify a value, a default `$top` of 25 is injected, with a maximum cap of 100.
> - Chat messages are capped at 10 per request.
> - The `$skip` and `$skiptoken` query parameters are blocked.

#### Example

##### Request

[▶ Open this example in the Interactive Demo](https://aka.ms/copilot.dev?page=%2Fwork-iq%2Fmcp&server=workiq-remote&tool=fetch&args=%7B%22entityUrls%22%3A%5B%22%2Fme%2Fmessages%22%5D%7D)

```json
{
  "method": "tools/call",
  "params": {
    "name": "fetch",
    "arguments": {
      "entityUrls": [
        "/me/messages"
      ]
    }
  }
}
```

##### Response

```json
{
  "results": [
    {
      "data": {
        "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users('0c6bc4a2-337b-4672-9bcd-6aa53af1b02a')/messages(id,subject,from,receivedDateTime,isRead,importance)",
        "value": [
          {
            "@odata.etag": "W/\"CQAAABYAAADltuSvpI1YRKDFG38ihSmWAABepTg8\"",
            "id": "AAMkADk0...",
            "receivedDateTime": "2026-05-31T15:40:44Z",
            "subject": "Your weekly PIM digest for Contoso",
            "importance": "normal",
            "isRead": false,
            "from": {
              "emailAddress": {
                "name": "Microsoft Security",
                "address": "MSSecurity-noreply@microsoft.com"
              }
            }
          },
          {
            "@odata.etag": "W/\"CQAAABYAAADltuSvpI1YRKDFG38ihSmWAAA/eZg0\"",
            "id": "AAMkADk0...",
            "receivedDateTime": "2026-05-26T07:39:09Z",
            "subject": "Microsoft Entra ID Protection Weekly Digest",
            "importance": "normal",
            "isRead": false,
            "from": {
              "emailAddress": {
                "name": "Microsoft Security",
                "address": "MSSecurity-noreply@microsoft.com"
              }
            }
          }
        ],
        "@odata.nextLink": "https://graph.microsoft.com/v1.0/me/messages?%24select=id%2csubject%2cfrom%2creceivedDateTime%2cisRead%2cimportance&%24top=10&%24skip=10"
      },
      "statusCode": 200
    }
  ]
}
```

---

### fetch_blob

Fetches binary content, such as documents, images, and Office files from a relative Work IQ path. The content is returned as a Base64-encoded string with metadata.

#### Parameters

| Parameter | Type   | Required | Description                                                                                                                                 |
|-----------|--------|----------|---------------------------------------------------------------------------------------------------------------------------------------------|
| `path`    | string | Yes      | The relative path to the binary content (for example, `/drives/{drive-id}/items/{item-id}/content`)                                         |
| `format`  | string | No       | File conversion format (for example, `pdf`). Supported only by drive-content endpoints that accept the Microsoft Graph `$format` parameter. |
| `agentId` | string | No       | Reserved for future use.                                                                                                                    |

#### Response

A successful response exposes the following properties through the MCP `structuredContent` field.

| Property        | Type    | Description                            |
|-----------------|---------|----------------------------------------|
| `statusCode`    | integer | The HTTP status code for the response. |
| `contentType`   | string  | The MIME type of the returned content. |
| `blobName`      | string  | The file name, when available.         |
| `sizeBytes`     | integer | The size of the raw content in bytes.  |
| `base64Content` | string  | The binary content encoded as Base64.  |

> [!NOTE]
>
> - The default raw-file limit is 4 MB and can be configured per path.
> - Clients must Base64-decode `base64Content` to recover the original bytes.
> - The blob payload isn't duplicated in the MCP text content field. Clients must read it from `structuredContent`.
> - A file exceeding the configured limit returns an MCP tool error containing the configured limit; it doesn't return blob content.

#### Example

##### Request

[▶ Open this example in the Interactive Demo](https://aka.ms/copilot.dev?page=%2Fwork-iq%2Fmcp&server=workiq-remote&tool=fetch_blob&args=%7B%22path%22%3A%22%2Fme%2Fdrive%2Fitems%2F01ABCDEF%2Fcontent%22%7D&json=false&mode=demo)

```json
{
  "method": "tools/call",
  "params": {
    "name": "fetch_blob",
    "arguments": {
      "path": "/me/drive/items/01ABCDEF/content"
    }
  }
}
```

##### Response

```json
{
  "structuredContent": {
    "statusCode": 200,
    "contentType": "text/plain",
    "blobName": "notes.txt",
    "sizeBytes": 14,
    "base64Content": "SGVsbG8sIFdvcmtJUSE="
  }
}
```

---

### create_entity

Creates a new entity in a collection.

#### Parameters

| Parameter   | Type   | Required | Description                                                                                                              |
|-------------|--------|----------|--------------------------------------------------------------------------------------------------------------------------|
| `parentUrl` | string | Yes      | The relative resource path for the collection (for example, `/me/events`, `/me/messages`).                               |
| `jsonBody`  | object or string | Yes | The entity data to create, matching the resource schema. Supply a JSON object or a JSON-encoded string. |
| `agentId`   | string | No       | Reserved for future use.                                                                                                 |

#### Response

A successful response contains an object in the `structuredContent` property with the following properties.

| Property     | Type    | Description                                  |
|--------------|---------|----------------------------------------------|
| `data`       | object  | The JSON object returned by Microsoft Graph. |
| `statusCode` | integer | The HTTP status code for the response.       |

#### Example

##### Request

[▶ Open this example in the Interactive Demo](https://aka.ms/copilot.dev?page=%2Fwork-iq%2Fmcp&server=workiq-remote&tool=create_entity&args=%7B%22parentUrl%22%3A%22%2Fme%2Fmessages%22%2C%22jsonBody%22%3A%7B%22subject%22%3A%22Hello+world%21%22%7D%7D)

```json
{
  "method": "tools/call",
  "params": {
    "name": "create_entity",
    "arguments": {
      "parentUrl": "/me/messages",
      "jsonBody": {
        "subject": "Hello world!"
      }
    }
  }
}
```

##### Response

```json
{
  "statusCode": 201,
  "data": {
    "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users('0c6bc4a2-337b-4672-9bcd-6aa53af1b02a')/messages/$entity",
    "@odata.etag": "W/\"CQAAABYAAADltuSvpI1YRKDFG38ihSmWAABlFT7g\"",
    "id": "AAMkADk0...",
    "createdDateTime": "2026-06-01T17:32:39Z",
    "lastModifiedDateTime": "2026-06-01T17:32:39Z",
    "changeKey": "CQAAABYAAADltuSvpI1YRKDFG38ihSmWAABlFT7g",
    "categories": [],
    "receivedDateTime": "2026-06-01T17:32:39Z",
    "sentDateTime": "2026-06-01T17:32:39Z",
    "hasAttachments": false,
    "internetMessageId": "<CH7PR03MB79063B75F29F929523CCC68FAE152@CH7PR03MB7906.namprd03.prod.outlook.com>",
    "subject": "Hello world!",
    "bodyPreview": "",
    "importance": "normal",
    "parentFolderId": "AQMkADk0...",
    "conversationId": "AAQkADk0...",
    "conversationIndex": "AQHc8eykdVXKjf4ytUio+dQPqWG95A==",
    "isDeliveryReceiptRequested": false,
    "isReadReceiptRequested": false,
    "isRead": true,
    "isDraft": true,
    "inferenceClassification": "focused",
    "body": {
      "contentType": "text",
      "content": ""
    },
    "toRecipients": [],
    "ccRecipients": [],
    "bccRecipients": [],
    "replyTo": [],
    "flag": {
      "flagStatus": "notFlagged"
    }
  }
}
```

---

### update_entity

Updates an existing entity.

#### Parameters

| Parameter   | Type   | Required | Description                                                                                                              |
|-------------|--------|----------|--------------------------------------------------------------------------------------------------------------------------|
| `entityUrl` | string | Yes      | The relative path to the entity (for example, `/me/messages/{message-id}`).                                              |
| `jsonBody`  | object or string | Yes | The entity data to update, matching the resource schema. Supply a JSON object or a JSON-encoded string. |
| `headers` | object | No | Optional request headers, when advertised. If the operation schema declares `If-Match`, supply the `@odata.etag` from the latest read of the same entity. |
| `agentId`   | string | No       | Reserved for future use.                                                                                                 |

#### Response

A successful response contains an object in the `structuredContent` property with the following properties.

| Property     | Type    | Description                                  |
|--------------|---------|----------------------------------------------|
| `data`       | object  | The JSON object returned by Microsoft Graph. |
| `statusCode` | integer | The HTTP status code for the response.       |

#### Example

##### Request

[▶ Open this example in the Interactive Demo](https://aka.ms/copilot.dev?page=%2Fwork-iq%2Fmcp&server=workiq-remote&tool=update_entity&args=%7B%22entityUrl%22%3A%22%2Fme%2Fmessages%2FAAMkADk0...%22%2C%22jsonBody%22%3A%7B%22subject%22%3A%22Updated%3A+Hello+world%21%22%7D%7D)

```json
{
  "method": "tools/call",
  "params": {
    "name": "update_entity",
    "arguments": {
      "entityUrl": "/me/messages/AAMkADk0...",
      "jsonBody": {
        "subject": "Updated: Hello world!"
      }
    }
  }
}
```

##### Response

```json
{
  "statusCode": 200,
  "data": {
    "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users('0c6bc4a2-337b-4672-9bcd-6aa53af1b02a')/messages/$entity",
    "@odata.etag": "W/\"CQAAABYAAADltuSvpI1YRKDFG38ihSmWAABlFT7g\"",
    "id": "AAMkADk0...",
    "createdDateTime": "2026-06-01T17:32:39Z",
    "lastModifiedDateTime": "2026-06-01T17:32:39Z",
    "changeKey": "CQAAABYAAADltuSvpI1YRKDFG38ihSmWAABlFT7g",
    "categories": [],
    "receivedDateTime": "2026-06-01T17:32:39Z",
    "sentDateTime": "2026-06-01T17:32:39Z",
    "hasAttachments": false,
    "internetMessageId": "<CH7PR03MB79063B75F29F929523CCC68FAE152@CH7PR03MB7906.namprd03.prod.outlook.com>",
    "subject": "Updated: Hello world!",
    "bodyPreview": "",
    "importance": "normal",
    "parentFolderId": "AQMkADk0...",
    "conversationId": "AAQkADk0...",
    "conversationIndex": "AQHc8eykdVXKjf4ytUio+dQPqWG95A==",
    "isDeliveryReceiptRequested": false,
    "isReadReceiptRequested": false,
    "isRead": true,
    "isDraft": true,
    "inferenceClassification": "focused",
    "body": {
      "contentType": "text",
      "content": ""
    },
    "toRecipients": [],
    "ccRecipients": [],
    "bccRecipients": [],
    "replyTo": [],
    "flag": {
      "flagStatus": "notFlagged"
    }
  }
}
```

---

### delete_entity

Deletes an entity.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entityUrl` | string | Yes | The relative resource path to the entity to delete (for example, `/me/messages/{id}`) |
| `headers` | object | No | Optional request headers, when advertised. If the operation schema declares `If-Match`, supply the `@odata.etag` from the latest read of the same entity. |
| `agentId` | string | No | Reserved for future use; currently ignored. This parameter doesn't select an agent for deletion. |

#### Response

A successful response contains a `statusCode` property set to 204 in the `structuredContent` field.

#### Example

##### Request

[▶ Open this example in the Interactive Demo](https://aka.ms/copilot.dev?page=%2Fwork-iq%2Fmcp&server=workiq-remote&tool=delete_entity&args=%7B%22entityUrl%22%3A%22%2Fme%2Fmessages%2FAAMkADk0MDkyMzM3LTlmNzAtNDhmM....%22%7D)

```json
{
  "method": "tools/call",
  "params": {
    "name": "delete_entity",
    "arguments": {
      "entityUrl": "/me/messages/AAMkADk0MDkyMzM3LTlmNzAtNDhmM...."
    }
  }
}
```

##### Response

```json
{
  "statusCode": 204
}
```

---

### do_action

Executes a side-effect action such as sending mail, copying, or moving items.

#### Parameters

| Parameter   | Type   | Required | Description                                                                                                |
|-------------|--------|----------|------------------------------------------------------------------------------------------------------------|
| `actionUrl` | string | Yes      | The relative path for the action (for example, `/me/messages/{message-id}/send`).                          |
| `jsonBody`  | object or string | No | The body for the action, if required by its schema. Supply a JSON object or a JSON-encoded string. |
| `agentId`   | string | No       | Reserved for future use.                                                                                   |

Some advertised schemas require the `jsonBody` argument even when the action doesn't require a body. In that case, supply `null`. The example below uses an empty object.

#### Response

A successful response contains an object in the `structuredContent` property with the following properties.

| Property     | Type    | Description                                          |
|--------------|---------|------------------------------------------------------|
| `data`       | object  | The JSON object returned by Microsoft Graph, if any. |
| `statusCode` | integer | The HTTP status code for the response.               |

#### Example

##### Request

[▶ Open this example in the Interactive Demo](https://aka.ms/copilot.dev?page=%2Fwork-iq%2Fmcp&server=workiq-remote&tool=do_action&args=%7B%22actionUrl%22%3A%22%2Fme%2Fmessages%2FAAMkADk0...%2Fsend%22%2C%22jsonBody%22%3A%7B%7D%7D)

```json
{
  "method": "tools/call",
  "params": {
    "name": "do_action",
    "arguments": {
      "actionUrl": "/me/messages/AAMkADk0.../send",
      "jsonBody": {}
    }
  }
}
```

##### Response

```json
{
  "statusCode": 202
}
```

---

### call_function

Calls a read-only function to compute derived data, such as calendar deltas or reminders.

Graph functions use GET. For a POST operation such as `/search/query`, use `do_action` with the operation's request body instead.

#### Parameters

| Parameter     | Type   | Required | Description                                                           |
|---------------|--------|----------|-----------------------------------------------------------------------|
| `functionUrl` | string | Yes | The relative path for a supported read-only function (for example, `/me/calendarView/delta`), including required query parameters. |
| `jsonBody` | object or string | No | Optional structured parameters for a provider function. Available only when advertised for the caller; supply a JSON object or JSON-encoded string according to the function schema. |
| `agentId`     | string | No       | Reserved for future use.                                              |

#### Response

A successful response contains an object in the `structuredContent` property with the following properties.

| Property     | Type    | Description                                  |
|--------------|---------|----------------------------------------------|
| `data`       | object  | The JSON object returned by Microsoft Graph. |
| `statusCode` | integer | The HTTP status code for the response.       |

#### Example

##### Request

[▶ Open this example in the Interactive Demo](https://aka.ms/copilot.dev?page=%2Fwork-iq%2Fmcp&server=workiq-remote&tool=call_function&args=%7B%22functionUrl%22%3A%22%2Fme%2FcalendarView%2Fdelta%3FstartDateTime%3D2026-06-01T00%3A00%3A00Z%26endDateTime%3D2026-06-08T00%3A00%3A00Z%22%7D&json=false&mode=demo)

```json
{
  "method": "tools/call",
  "params": {
    "name": "call_function",
    "arguments": {
      "functionUrl": "/me/calendarView/delta?startDateTime=2026-06-01T00:00:00Z&endDateTime=2026-06-08T00:00:00Z"
    }
  }
}
```

##### Response

The following illustrative payload shows an event in the requested time range. The delta token is abbreviated.

```json
{
  "data": {
    "value": [
      {
        "@odata.etag": "W/\"5bbkr6SNWESgxRt/IoUplgAAKcCTsQ==\"",
        "id": "AAMkADk0...",
        "subject": "Sales and Marketing Sync",
        "isAllDay": false,
        "start": {
          "dateTime": "2026-06-07T12:30:00.0000000",
          "timeZone": "UTC"
        },
        "end": {
          "dateTime": "2026-06-07T13:00:00.0000000",
          "timeZone": "UTC"
        },
        "location": {
          "displayName": "Sales and Marketing / General",
          "locationType": "default",
          "uniqueId": "Sales and Marketing / General",
          "uniqueIdType": "private"
        },
        "organizer": {
          "emailAddress": {
            "name": "Sales and Marketing",
            "address": "SalesandMarketing@contoso.com"
          }
        }
      }
    ],
    "@odata.deltaLink": "https://graph.microsoft.com/v1.0/me/calendarView/delta?$deltatoken=opaque-token"
  },
  "statusCode": 200
}
```

---

## Copilot tools

Copilot tools provide natural-language intelligence by invoking Microsoft 365 Copilot and discovering available agents.

### ask

Asks Microsoft 365 Copilot (or a specific agent) a natural-language question about the user's data. When you provide an `agentId`, the request goes to that specific agent. When you omit it, the request goes to the built-in Microsoft 365 Copilot agent.

#### Parameters

| Parameter        | Type         | Required | Description                                                                                                            |
|------------------|--------------|----------|------------------------------------------------------------------------------------------------------------------------|
| `question`       | string       | Yes      | The natural-language question to ask                                                                                   |
| `agentId`        | string       | No       | The ID of a specific agent to route the question to. If omitted, defaults to the built-in Microsoft 365 Copilot agent. |
| `fileUrls`       | string array | No       | An array of OneDrive or SharePoint file URLs to use as context.                                                        |
| `conversationId` | string       | No       | A conversation ID from an existing conversation. If provided, this request continues the existing conversation.        |
| `timeZone`       | string       | No       | An IANA time zone identifier matching the user's current UTC offset. If omitted, time are returned in UTC.             |

#### Response

A successful response returns the answer as plain text in an MCP `content` text block and as `structuredContent.answer`. The structured content includes the following properties.

| Property         | Type   | Description                                                                                                       |
|------------------|--------|-------------------------------------------------------------------------------------------------------------------|
| `answer` | string | The answer from Microsoft 365 Copilot or the specified agent. |
| `conversationId` | string | The conversation ID, when available. Provide it in subsequent requests to continue the conversation. |
| `application/vnd.ms-workiq.reference` | object | Citation references keyed by reference ID, when available. References can include target links and sensitivity metadata. |

Errors use a different envelope: `isError` is `true`, and a `content` text block can contain JSON with an `error` field and, when available, a `conversationId`. Don't parse an error as an answer or expect `structuredContent.answer` to be present.

#### Example

##### Request

[▶ Open this example in the Interactive Demo](https://aka.ms/copilot.dev?page=%2Fwork-iq%2Fmcp&server=workiq-remote&tool=ask&args=%7B%22question%22%3A%22Do+I+have+any+meetings+today%3F%22%7D)

```json
{
  "method": "tools/call",
  "params": {
    "name": "ask",
    "arguments": {
      "question": "Do I have any meetings today?"
    }
  }
}
```

##### Response

```json
{
  "isError": false,
  "content": [
    {
      "type": "text",
      "text": "You have **4 meetings scheduled today**."
    }
  ],
  "structuredContent": {
    "answer": "You have **4 meetings scheduled today**.",
    "conversationId": "cc7d6f5202cb4c5b814b120440e48ede"
  }
}
```

For example, an empty question can produce this error tool result:

```json
{
  "isError": true,
  "content": [
    {
      "type": "text",
      "text": "{\"error\":\"Parameter 'question' is required.\"}"
    }
  ]
}
```

---

### list_agents

Lists available agents that you can use with the `ask` tool. Returns a JSON array of available agents, and the built-in Microsoft 365 Copilot agent is always included.

#### Parameters

This tool takes no parameters.

#### Response

A successful response contains a JSON string in the `text` property of a [TextContent](https://modelcontextprotocol.io/specification/latest/schema#textcontent) object. The string includes the following properties.

| Property   | Type   | Description                          |
|------------|--------|--------------------------------------|
| `agentId`  | string | The unique identifier for the agent. |
| `name`     | string | The display name of the agent.       |
| `provider` | string | The publisher of the agent.          |

#### Example

##### Request

[▶ Open this example in the Interactive Demo](https://aka.ms/copilot.dev?page=%2Fwork-iq%2Fmcp&server=workiq-remote&tool=list_agents&args=%7B%7D)

```json
{
  "method": "tools/call",
  "params": {
    "name": "list_agents",
    "arguments": {}
  }
}
```

##### Response

```json
[
  {
    "agentId": "bizchat-as-gpt-scenario",
    "name": "Microsoft Copilot",
    "provider": "Microsoft"
  }
]
```

---

## Retrieval tools

Retrieval tools return source evidence for grounding an answer rather than a synthesized Copilot answer.

### retrieve

Searches Microsoft 365 and connected enterprise sources and returns read-only grounding evidence with source metadata. Use `retrieve` to gather evidence for your own answer, and `ask` to obtain a Copilot or agent answer.

> [!NOTE]
> Call `retrieve` only when it appears in your connection's MCP `tools/list` response. Availability and the input and output variants depend on the supported release and capabilities enabled for the caller.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string or string array | Yes | Nonempty natural-language search input. Use a single string when the advertised schema requires a string. Legacy schemas accept an array of queries. Use the shape advertised by your connection, not both interchangeably. |
| `strategy` | string | No | `grounding` (default) searches indexed Microsoft 365 content. `copilot` includes federated connectors and other connected sources. Choose according to the sources needed; other values are rejected. |
| `agentId` | string | No | The agent to target. Defaults to `bizchat-as-gpt-scenario`. |
| `capabilities` | object array | No | Optional source-selection allow-list using declarative agent capability objects. Omit it or use an empty array to search all sources available to the selected agent. A nonempty list can't be combined with a non-default `agentId`. |
| `includeDeveloperCard` | boolean | No | Request orchestration diagnostics. Defaults to `false`. |

The supported capability names are `People`, `Meetings`, `OneDriveAndSharePoint`, `Email`, `TeamsMessages`, `Dataverse`, and `GraphConnectors`. Capability objects can include source-specific scope fields, such as SharePoint URLs, email folders, Teams message URLs, or connector connection IDs. Use the advertised schema for those fields. `Dataverse` and `GraphConnectors` require the `copilot` strategy; they aren't supported with `grounding`.

#### Response

The MCP result stores its retrieval payload in `structuredContent["application/vnd.ms-workiq.retrieval"]`. Depending on the output variant, the payload includes:

| Property | Type | Description |
|----------|------|-------------|
| `markdown` | string | Grounding text with citation markers in the top-level-markdown variant. Match its `[^id]` markers to `retrievalHits[].id`. |
| `retrievalHits` | object array | Source metadata for the top-level-markdown variant, including identifiers, URLs, and sensitivity information when available. |
| `results` | object array | Per-source evidence in the structured-results variant. Ground on each `results[].markdown` value and use metadata from that same result for attribution. |
| `resultCount` | integer | The result count reported by the retrieval response. |
| `stoppedReason` | string | The terminal state: `completed`, `cancelled`, or `error`. |
| `conversationId` | string | Conversation context, when returned by the enabled output variant. |

The result can also include grounding text in MCP `content`; some output variants leave `content` empty. Read `structuredContent` rather than assume that evidence is always duplicated in text. Check `isError` and `stoppedReason` before treating results as complete. Errors can use a text-only JSON error envelope instead of the retrieval payload. Preserve source links, citation identifiers, content-protection decisions, and sensitivity metadata when passing evidence to another consumer.

##### Content protection

Each `retrievalHits[]` or `results[]` source can include a `contentProtection` string. Use source content only when its decision is recognized as allowing use by the supported response contract. Missing, failed, blocked, or unknown decisions aren't permission to use the content. Don't infer an allow decision from an unfamiliar value, a sensitivity label, `isError: false`, or `stoppedReason: "completed"`.

For the top-level-markdown variant, correlate citation markers with their source hits before using the text. Don't use text from a source whose protection decision doesn't allow use. If you can't separate that text from the combined markdown, don't use the combined markdown. For the structured-results variant, apply the check to each source before using its `markdown`.

#### Example

##### Request

This request uses the single-string query variant and restricts retrieval to indexed OneDrive and SharePoint content.

```json
{
  "method": "tools/call",
  "params": {
    "name": "retrieve",
    "arguments": {
      "query": "Find the Project Atlas rollout plan",
      "strategy": "grounding",
      "capabilities": [
        {
          "name": "OneDriveAndSharePoint"
        }
      ]
    }
  }
}
```

For a connection that advertises the legacy array variant, use `"query": ["Find the Project Atlas rollout plan"]` instead.

##### Response

The following illustrative tool result uses the top-level-markdown variant and shows a failed content-protection evaluation. The citation marker matches the source's `id`, but the client must not use this source or its associated markdown even though the tool completed successfully.

```json
{
  "isError": false,
  "content": [
    {
      "type": "text",
      "text": "Project Atlas rollout plan [^h1]."
    }
  ],
  "structuredContent": {
    "application/vnd.ms-workiq.retrieval": {
      "markdown": "Project Atlas rollout plan [^h1].",
      "retrievalHits": [
        {
          "id": "h1",
          "webUrl": "https://contoso.sharepoint.com/sites/Atlas/Shared%20Documents/Rollout.docx",
          "contentProtection": "evaluationFailed"
        }
      ],
      "resultCount": 1,
      "stoppedReason": "completed"
    }
  }
}
```

If a request returns no matching evidence, that doesn't establish that a document doesn't exist.

---

## Schema tools

Schema tools enable runtime discovery of available API paths and their OpenAPI schemas. These tools allow agents to discover what operations are available without loading large schema definitions into context upfront.

### get_schema

Retrieves the OpenAPI schema for a specific operation, identified by path and operation type.

> [!NOTE]
> Currently, only Microsoft Graph v1.0 schemas are available.

#### Parameters

| Parameter       | Type   | Required | Description                                                                                                                                                                               |
|-----------------|--------|----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `path` | string | No | The relative API path to get the schema for (for example, `/me/messages`). Supply a path and matching operation type to discover a specific operation. |
| `operationType` | string | Yes | The operation type: `fetch`, `create`, `update`, or `action`. Some enabled audiences also advertise `function`; use it only when present in your connection's schema. |
| `format` | string | No | `cddl` ([Concise Data Definition Language](https://www.rfc-editor.org/rfc/rfc8610)), `jsonschema` ([JSON Schema](https://json-schema.org/)), or `typescript` (TypeScript definitions). Defaults to `cddl`. |
| `agentId` | string | No | Reserved for future use; currently ignored. |

`operationIds` and `backend` aren't inputs in the current advertised contract. Use the parameter names exposed by MCP `tools/list`, not identifiers from older examples.

#### Response

A successful response contains schema text in an MCP `content` text block. The format depends on `format`; omitting it returns CDDL rather than JSON Schema.

#### Example

##### Request

[▶ Open this example in the Interactive Demo](https://aka.ms/copilot.dev?page=%2Fwork-iq%2Fmcp&server=workiq-remote&tool=get_schema&args=%7B%22path%22%3A%22%2Fme%2Fmessages%22%2C%22operationType%22%3A%22fetch%22%7D)

```json
{
  "method": "tools/call",
  "params": {
    "name": "get_schema",
    "arguments": {
      "path": "/me/messages",
      "operationType": "fetch"
    }
  }
}
```

##### Response

The following excerpt shows CDDL returned in the text block. Additional definitions are omitted.

```cddl
microsoft-graph-messageCollectionResponse = {
  BaseCollectionPaginationCountResponse,
  ? "value": [* microsoft-graph-message]
}

BaseCollectionPaginationCountResponse = {
  ? "@odata.count": (int / null),
  ? "@odata.nextLink": (tstr / null)
}
```

---

### search_paths

Searches available API paths and supported operations. Use this tool to discover resource paths before calling entity tools.

The advertised input contract is rollout-dependent. The legacy variant requires `filter`, a regex or path-keyword filter. The query variant requires `query`, a natural-language description or path prefix. Use the variant in your connection's MCP `tools/list` response. Legacy support hasn't been removed; don't assume that every connection accepts the query variant.

> [!NOTE]
> Currently, only Microsoft Graph v1.0 paths are available.

#### Parameters

| Parameter | Type   | Required | Description                                                                                            |
|-----------|--------|----------|--------------------------------------------------------------------------------------------------------|
| `filter` | string | In the legacy variant | A regex or path-keyword filter (for example, `messages`, `messages\|reply`, or `/me/people`). |
| `query` | string | In the query variant | A natural-language resource and operation description (for example, `recent emails` or `reply to email`), or a path prefix such as `/me/people`. |

These rows describe alternative advertised schemas, not two required inputs for one request. Supply only the input advertised by your connection; don't send both. `backend` isn't an input in either current variant.

#### Response

A successful response contains a `paths` property in the `structuredContent` field. This property is an array of objects with the following properties.

| Property     | Type         | Description                                                                              |
|--------------|--------------|------------------------------------------------------------------------------------------|
| `path`       | string       | The relative API path.                                                                   |
| `operations` | string array | The supported operations for the path, such as `fetch`, `create`, `update`, `delete`, and `action`. Enabled audiences can also receive `function`. |

#### Example

##### Request

This example uses the legacy `filter` variant.

[▶ Open this example in the Interactive Demo](https://aka.ms/copilot.dev?page=%2Fwork-iq%2Fmcp&server=workiq-remote&tool=search_paths&args=%7B%22filter%22%3A%22messages%22%7D)

```json
{
  "method": "tools/call",
  "params": {
    "name": "search_paths",
    "arguments": {
      "filter": "messages"
    }
  }
}
```

For a connection that advertises `query`, use the following request instead:

```json
{
  "method": "tools/call",
  "params": {
    "name": "search_paths",
    "arguments": {
      "query": "/me/messages"
    }
  }
}
```

##### Response

The following excerpt shows paths returned for `/me/messages`. Other matching paths are omitted.

```json
{
  "paths": [
    {
      "path": "/me/messages",
      "operations": [
        "fetch",
        "create"
      ]
    },
    {
      "path": "/me/messages/{message-id}",
      "operations": [
        "fetch",
        "update",
        "delete"
      ]
    },
    {
      "path": "/me/messages/{message-id}/send",
      "operations": [
        "action"
      ]
    }
  ]
}
```

---

## Allowed resource paths

Entity tools work with relative resource paths. By default, the following path prefixes are allowed:

- `/me/`
- `/users/`
- `/sites/`

The following path segments are blocked:

- `/authentication/`
- `/servicePrincipals/`

> [!NOTE]
> The complete list of allowed and blocked paths depends on per-tenant policy configuration and might vary.

## Error handling

When a tool call fails, the Work IQ MCP server returns errors with the following information:

- HTTP status code from the downstream service
- `Retry-After` headers for throttling responses (passed through from Microsoft Graph)
- Request correlation IDs for troubleshooting

The server doesn't perform automatic retries. The MCP client receives errors and makes client-side retry decisions.

## Related content

- [Work IQ MCP overview](overview.md)
- [Work IQ API overview](../api-overview.md)
- [Work IQ API permissions reference](../permissions.md)
