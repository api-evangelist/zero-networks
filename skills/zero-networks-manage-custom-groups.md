---
name: zero-networks-manage-custom-groups
description: >-
  Build and maintain the custom groups Zero Networks rules target — create a group, add or
  remove asset and identity members in bulk, and keep group-scoped policy correct. Use when
  asked to group servers or identities for segmentation, or to change who or what a rule
  applies to without editing the rule.
api: zero-networks-platform
generated: '2026-09-05'
method: generated
source: openapi/zero-networks-platform-openapi.yaml
operations:
  - Assets_Search
  - CustomGroups_Create
  - CustomGroups_Get
  - CustomGroups_Update
  - CustomGroups_Delete
  - CustomGroupMembers_List
  - CustomGroupsMembers_Add
  - CustomGroupsMembers_Delete
---

# Manage Zero Networks custom groups

Base URL `https://portal.zeronetworks.com/api/v1`, `Authorization: <console token>`.

A custom group is a named set of assets or identities that segmentation rules can target
instead of naming machines one by one. Changing a group's membership changes the reach of
every rule pointed at it — which makes this the highest-leverage and most dangerous small
write in the API.

## Create a group

`CustomGroups_Create` — `POST /groups/custom`

`customGroupBody` requires only `name`. Optional: `description`, `membersId` (seed the
membership at creation), `conditions` (dynamic membership).

`409 Conflict` on a name collision. Not idempotent — do not retry a timed-out create; list
and reconcile.

## Read a group

`CustomGroups_Get` — `GET /groups/custom/{groupId}`

Returns `createdAt`/`updatedAt`/`addedBy` audit fields, `directMembersCount`, and
`hasProtectionPolicy`. **Check `hasProtectionPolicy` before changing anything.** `true`
means live policy depends on this group and your membership edit will change what that
policy allows or blocks.

## Members

- List: `CustomGroupMembers_List` — `GET /groups/custom/{groupId}/members`
- Add: `CustomGroupsMembers_Add` — `PUT /groups/custom/{groupId}/members`
- Remove: `CustomGroupsMembers_Delete` — `DELETE /groups/custom/{groupId}/members`

All three speak `membersId`, an array — membership is manipulated in bulk, never one member
per call. Resolve machine members with `Assets_Search` first.

`CustomGroupMembers_List` returns a bare `membersId` array with **no cursor, offset or limit
parameter**. There is no pagination on this endpoint. On a large group, assume you receive
everything in one response and size your handling accordingly; if you see a suspiciously
round count, do not assume it is a page — the contract offers no way to ask for the next one.

## The order that matters

To change what a rule covers:

1. `CustomGroups_Get` — check `hasProtectionPolicy`.
2. `CustomGroupMembers_List` — **save the current `membersId` array.** This is your only
   restore point.
3. `CustomGroupsMembers_Add` or `..._Delete` with the delta.
4. `CustomGroupMembers_List` again and diff against intent.

## Reversibility

| you did | reverse with | window |
|---|---|---|
| `CustomGroupsMembers_Add` | `CustomGroupsMembers_Delete` with the same `membersId` | not stated |
| `CustomGroupsMembers_Delete` | `CustomGroupsMembers_Add` with the saved `membersId` | not stated |
| `CustomGroups_Update` | `CustomGroups_Update` with the prior body | not stated |
| `CustomGroups_Create` | `CustomGroups_Delete` | not stated |

Member removal is only reversible if you captured the list in step 2 — the API keeps no
membership history you can read back. `CustomGroups_Delete` on a group with
`hasProtectionPolicy: true` changes live firewall policy; confirm with a human first.

## Errors

`{"error": "...", "message": "..."}` on `400`, `401`, `403`, `409` (create only), `500`.
Note that the member operations do **not** declare a `404` in the contract even though they
address a `{groupId}` — a bad group id may surface as a `400` or `500` rather than a clean
`404`. Do not depend on `404` to detect a missing group; `CustomGroups_Get` first.
