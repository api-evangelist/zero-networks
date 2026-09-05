---
name: zero-networks-segment-an-asset
description: >-
  Create, review and safely roll back an inbound or outbound segmentation firewall rule for
  a Zero Networks-protected asset — the platform's core write flow. Use when asked to allow
  or block east-west traffic to a server, open a port between two assets, or undo a
  segmentation change.
api: zero-networks-platform
generated: '2026-09-05'
method: generated
source: openapi/zero-networks-platform-openapi.yaml
operations:
  - Assets_Search
  - InboundRules_Create
  - InboundRule_Get
  - InboundRule_Update
  - InboundRule_Delete
  - OutboundRules_Create
  - OutboundRule_Get
  - OutboundRule_Update
  - OutboundRule_Delete
---

# Segment an asset with a Zero Networks rule

Base URL `https://portal.zeronetworks.com/api/v1`. Auth is a console-issued token in the
`Authorization` header — mint it under **Settings > API**. A token whose role is
`API-ReadOnly` (userRole 5) will 403 on every step below except step 1; you need
`API-FullAccess` (userRole 4).

> The published contract's `servers[]` entry says `https://portal.zeronetworks.com/v1/api`.
> That path returns nginx 404. The live base is `/api/v1`. Use `/api/v1`.

## 1. Resolve the asset

`Assets_Search` — `GET /assets/searchId?fqdn=server.domain.local`

Returns `{ "assetId": "..." }`. Every rule field that names a machine wants this opaque id,
not the FQDN. Entity ids are polymorphic: the same id slot accepts an asset, a custom group,
or an encoded IP/subnet, and the response carries no discriminator, so keep track of what
you resolved.

## 2. Build the rule body

`ruleBody` requires exactly six fields — omit any one and you get a 400:

| field | meaning |
|---|---|
| `action` | `1` Allow, `2` Block, `3` Force Block |
| `localEntityId` | the protected asset (the id from step 1) |
| `localProcessesList` | processes on the local side; `[]` for any |
| `portsList` | array of `{ ports, protocolType }`; `protocolType` is the IANA protocol number — `6` TCP, `17` UDP |
| `remoteEntityIdsList` | the peers this rule covers |
| `state` | the rule's enablement state |

Optional but worth setting every time:

- `name` and `description` — this is enterprise firewall policy; unlabeled rules are debt.
- `changeTicket` — first-class field for the ITSM ticket authorizing the change. Use it.
- `expiresAt` — makes the rule self-expiring. **This is the closest thing to an undo the API
  has.** See "Reversibility" below.
- `reviewMode` — stage the rule for human approval instead of enforcing it immediately.
- `srcUsersList` — scope the rule to specific identities rather than to the whole machine.

## 3. Create

- Inbound: `InboundRules_Create` — `POST /protection/rules/inbound`
- Outbound: `OutboundRules_Create` — `POST /protection/rules/outbound`

A `409 Conflict` means a colliding rule already exists. **Do not retry a create that timed
out.** This API has no idempotency mechanism: no `Idempotency-Key` header, no request-key
field, no conditional-request support. A blind retry either creates a duplicate rule or
409s, and you cannot tell from the response which outcome the first attempt produced. On a
timeout, read the rule collection back and reconcile before writing again.

## 4. Verify

`InboundRule_Get` / `OutboundRule_Get` — `GET /protection/rules/{direction}/{ruleId}`

Read back and confirm `action`, `state`, `portsList` and both entity sides match intent
before you report success.

## Reversibility — read this before any write

Every write here has a reversal operation, but **no reversal window is published anywhere**.

| you did | reverse with | window |
|---|---|---|
| `InboundRules_Create` | `InboundRule_Delete` (`DELETE /protection/rules/inbound/{ruleId}`) | not stated |
| `InboundRule_Update` | `InboundRule_Update` with the prior body | not stated |
| `OutboundRules_Create` | `OutboundRule_Delete` | not stated |
| `OutboundRule_Update` | `OutboundRule_Update` with the prior body | not stated |

Two rules follow from that:

1. **Capture the current body with `*_Get` before every update.** There is no version
   history and no prior-state read operation. If you do not save the old body, the update is
   not reversible — you will have nothing to restore.
2. **Prefer `state` (disable) over `DELETE`, and prefer `expiresAt` over both.** The `rule`
   schema carries `deletedAt`/`deletedBy`, which implies rules are soft-deleted server-side,
   but no documented operation reads or restores a deleted rule and no retention period is
   published. Treat `DELETE` as permanent.

Never tell a user a change can be undone "within N days". Zero Networks does not state a
window and inventing one here could cost them an outage or an open port.

## Errors

Uniform envelope on every failure: `{"error": "...", "message": "..."}`, `application/json`,
not RFC 9457. There is no error code registry — you can only branch on the HTTP status.

- `400` malformed body or a missing required field
- `401` `{"error":"unauthorized","message":"jwt authorization not found"}` — no/invalid token
- `403` token role lacks the privilege (likely `API-ReadOnly` attempting a write)
- `404` no such `ruleId`
- `409` colliding rule (create only)
- `500` server error

No `429` is documented and no rate-limit or `Retry-After` header is returned, so there is no
backpressure signal to read. Use conservative exponential backoff and never hot-loop writes.
