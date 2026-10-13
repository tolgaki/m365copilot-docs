---
title: Work IQ MCP overview
description: Learn how to use Work IQ through the Model Context Protocol (MCP) to give AI agents tool-based access to Microsoft 365 data.
author: jasonjoh
ms.author: jasonjoh
ms.topic: overview
ms.localizationpriority: medium
ms.date: 10/12/2026
---

# Work IQ MCP overview

The Work IQ MCP server exposes Microsoft 365 intelligence capabilities to AI agents through the [Model Context Protocol (MCP)](https://modelcontextprotocol.io). It provides generic tools to access Microsoft 365 entities, retrieve grounding evidence, invoke Microsoft 365 Copilot for natural-language reasoning, and discover API schemas through a single MCP endpoint. Tool availability depends on the supported release and the capabilities enabled for the caller.

> [!TIP]
> Explore the Work IQ MCP tools in the [Interactive Demo](https://aka.ms/copilot.dev?page=%2Fwork-iq%2Fmcp&server=workiq-remote). You can inspect tool schemas, load examples, and preview responses in Demo mode.

## Design principles

Work IQ MCP is built on the following design principles:

- **Fewer tools, more paths.** Generic entity tools operate on resource paths. New workloads can reuse these tools by adding paths instead of introducing a tool for every API operation.
- **Introspection over enumeration.** Agents ask for schemas at runtime (`get_schema`) rather than loading thousands of type definitions into context.
- **Policy over scopes.** Four broad OAuth permissions gate capability; fine-grained access control is enforced per path, method, and tenant policy.

## Tool categories

The tools covered by this documentation are organized into four categories:

| Category | Tools | Description |
|----------|-------|-------------|
| Entity tools | `fetch`, `fetch_blob`, `create_entity`, `update_entity`, `delete_entity`, `do_action`, `call_function` | Read entities and binary content, perform CRUD operations, invoke actions, and call read-only functions |
| Copilot tools | `ask`, `list_agents` | Invoke Microsoft 365 Copilot for natural-language intelligence and discover available agents |
| Retrieval tools | `retrieve` | Retrieve read-only grounding evidence and source metadata when available to the caller |
| Schema tools | `get_schema`, `search_paths` | Discover available API paths and retrieve OpenAPI schemas at runtime |

For detailed information about each tool, including parameters, see [Work IQ MCP tool reference](tool-reference.md).

Use MCP `tools/list` to discover the tools and input schemas exposed by your connected server. Follow any `nextCursor` values to obtain the complete inventory. Don't assume that every release or tenant exposes every tool in the table. In particular, call `retrieve` only when the server advertises it. If it isn't available, use `ask` for a Copilot answer or the entity tools for supported resource reads.

## How tools work

Entity tools work with **relative resource paths**. The path shows the resource, and the tool acts as the verb. Copilot and retrieval tools accept natural-language input:

```text
fetch /me/messages                         → read my emails
do_action /me/sendMail                     → send an email
create_entity /me/events                   → create a calendar event
fetch /me/chats/{id}/messages              → read Teams chat messages
call_function /me/calendarView/delta        → track calendar changes
do_action /search/query                    → submit a search request
ask "What deals closed this quarter?"      → invoke Copilot agent
retrieve "Find the Project Atlas plan"     → retrieve grounding evidence, when available
```

Microsoft Graph `/search/query` is a POST operation. Use `do_action` with the request body described by the [search API](/graph/api/search-query), not `call_function`, which uses GET for Graph functions. Path access remains subject to tenant policy.

Use `retrieve` when you need source evidence to ground an answer, and `ask` when you want a Copilot or agent answer. Retrieval source-selection options and availability are described in the [tool reference](tool-reference.md#retrieve).

## Authentication

The Work IQ MCP server uses Microsoft Entra ID for authentication. MCP clients automatically discover the authentication configuration through the `/.well-known/oauth-protected-resource` endpoint. For details on required permissions, see [Work IQ API permissions reference](../permissions.md).

> [!NOTE]
> The Work IQ service principal is created automatically in the tenant when Work IQ is used for the first time. In some corner cases, such as configuring Work IQ MCP policy in the Microsoft 365 admin center before any MCP use in the tenant, the service principal might not exist yet and policy configuration can fail. Tenant administrators can provision it by following the steps in [Enable your tenant for Work IQ](../enable-work-iq.md).

## Related content

- [Work IQ API overview](../api-overview.md)
- [Work IQ MCP tool reference](tool-reference.md)
- [Work IQ API permissions reference](../permissions.md)
