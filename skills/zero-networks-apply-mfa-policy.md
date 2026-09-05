---
name: zero-networks-apply-mfa-policy
description: >-
  Put just-in-time multi-factor authentication in front of a network protocol — RDP, SSH,
  SMB, WinRM — by creating an inbound or outbound Zero Networks reactive (MFA) policy. Use
  when asked to require MFA before a privileged port opens, protect an admin or service
  account path, or grant time-boxed user access to an asset.
api: zero-networks-platform
generated: '2026-09-05'
method: generated
source: openapi/zero-networks-platform-openapi.yaml
operations:
  - Assets_Search
  - MFAInboundPolicies_Create
  - MFAInboundPolicies_Get
  - MFAInboundPolicies_Update
  - MFAInboundPolicies_Delete
  - MFAOutboundPolicies_Create
  - MFAOutboundPolicies_Get
  - MFAOutboundPolicies_Update
  - MFAOutboundPolicies_Delete
  - InternalAccessPolicy_Create
  - InternalAccessPolicy_Get
  - InternalAccessPolicy_Update
  - InternalAccessPolicy_Delete
---

# Apply MFA to a protocol with Zero Networks

Base URL `https://portal.zeronetworks.com/api/v1`, `Authorization: <console token>`,
`API-FullAccess` role required for writes.

The contract calls these **reactive policies**; the console and SDKs call them **MFA
policies**. Same object. This is the capability that lets Zero Networks put MFA in front of
protocols that were never designed for it.

## 1. Resolve the destination asset

`Assets_Search` — `GET /assets/searchId?fqdn=dc01.corp.local` → `{ "assetId": "..." }`

## 2. Build the policy body

`reactivePolicyInboundBody` requires **thirteen** fields. Send all of them; a partial body
400s.

`dstEntityInfo`, `dstPort`, `dstProcessNames`, `fallbackToLoggedOnUser`, `mfaMethods`,
`overrideBuiltins`, `protocolType`, `ruleDuration`, `srcEntityInfos`, `srcProcessNames`,
`srcUserInfos`, `state`, `additionalPortsList`

Enums you must get right (all integers, from the contract):

- `protocolType` — `6` TCP, `17` UDP. Only these two.
- `mfaMethods` — `1` SMS, `2` Email, `3` Duo, `4` Browser, `5` No MFA, `6` Microsoft
  Authenticator, `7` Okta.
  **`5` means no MFA.** Setting `mfaMethods: 5` on a policy meant to enforce MFA silently
  produces the opposite of what was asked. Never pass 5 unless the user explicitly asked for
  a no-MFA policy.
- `ruleDuration` — how long the port stays open after a successful challenge:
  `1` hour, `2` day, `3` week, `4` month, `5` **never expires**, `6` 4 hours, `7` 12 hours,
  `8` 8 hours.
  **`5` means the grant never expires.** It is the least safe value in the enum and it is
  not the largest number — do not reach for it by assuming the enum is ordered.

Optional and worth using: `name`, `description`, `changeTicket`,
`excludedSrcEntityInfos`, `excludedSrcProcesses`, `restrictLoginToOriginatingUser`,
`useOccasionalMfa`, `useDefaultIdp`.

## 3. Create

- Inbound (protect a destination): `MFAInboundPolicies_Create` — `POST /protection/reactive-policies/inbound`
- Outbound (constrain a source): `MFAOutboundPolicies_Create` — `POST /protection/reactive-policies/outbound`

Neither declares a 409 and neither is idempotent-keyed. A retried create after a timeout may
produce a second policy. Read the collection back instead of retrying blind.

## 4. Verify

`MFAInboundPolicies_Get` / `MFAOutboundPolicies_Get` —
`GET /protection/reactive-policies/{direction}/{reactivePolicyId}`

Confirm `mfaMethods`, `ruleDuration`, `state`, `dstPort` and `protocolType` before reporting
success. Getting `mfaMethods` or `ruleDuration` wrong is an access-control failure, not a
cosmetic one, so verify rather than assume the write took.

## Related: user-to-asset access without MFA

`InternalAccessPolicy_*` (`/protection/internal-access/policies`) grants named users access
to one destination asset on named ports for a `ruleDuration`. Required fields: `dstAssetId`,
`dstPortsList`, `dstProcessNamesList`, `name`, `ruleDuration`, `srcUserIdsList`. Same
`ruleDuration` enum, same caveat about `5`.

## Reversibility

| you did | reverse with | window |
|---|---|---|
| `MFAInboundPolicies_Create` | `MFAInboundPolicies_Delete` | not stated |
| `MFAOutboundPolicies_Create` | `MFAOutboundPolicies_Delete` | not stated |
| `InternalAccessPolicy_Create` | `InternalAccessPolicy_Delete` | not stated |
| any `*_Update` | the same `*_Update` with the prior body | not stated |

`*_Get` before every update, or the update is not reversible. Prefer flipping `state` to
deleting. No restore endpoint and no retention period is published — do not promise one.

## Errors

`{"error": "...", "message": "..."}` on `400`, `401`, `403`, `404`, `500`. No error code
registry, no `429`, no rate-limit headers.
