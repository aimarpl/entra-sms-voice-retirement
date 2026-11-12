# Entra SMS and voice retirement

[![KQL](https://img.shields.io/badge/KQL-Microsoft%20Sentinel-0078D4?logo=microsoftazure&logoColor=white)](https://learn.microsoft.com/kusto/query/)
[![CI](https://github.com/aimarpl/entra-sms-voice-retirement/actions/workflows/quality.yml/badge.svg)](https://github.com/aimarpl/entra-sms-voice-retirement/actions/workflows/quality.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

KQL queries for finding and reducing dependency on SMS and voice
authentication in Microsoft Entra ID, ahead of their retirement.

Microsoft-provided SMS and voice delivery is being withdrawn. Users whose only
MFA method is SMS or voice face a **blocking** passkey registration prompt at
their retirement date, and there is no opt out from that enforcement.

## Timeline

| Date | What happens |
| ------ | -------------- |
| 1 September 2026 | Users enabled for SMS or voice are auto-enabled for passkeys and nudged to register. |
| 1 February 2027 | Microsoft-provided SMS and voice retired for all users **except** Global Administrators and external users. Internal guests are included in this date. |
| 1 July 2027 | Retired for Global Administrators and external users. |

Two details that are easy to get wrong:

- **Internal guest users follow the February date**, not July. Only external
  users and Global Administrators get the later deadline.
- **SSPR is in scope.** A user who signs in with a passkey but whose only
  password-reset method is a phone still loses self-service reset.

If you need a telephony channel for regulatory reasons, a provider can be
configured through Microsoft Security Store; see the Microsoft guidance below.

## Queries

| # | Query | Phase | Answers |
| --- | ------- | ------- | --------- |
| 01 | [users-at-risk](queries/01-users-at-risk.kql) | Assess | Who faces a blocking prompt at retirement? |
| 02 | [tenant-exposure](queries/02-tenant-exposure.kql) | Assess | How big is the problem, and is this tenant in scope at all? |
| 03 | [privileged-and-external-users](queries/03-privileged-and-external-users.kql) | Assess | Who falls under the July date, and are any admins affected? |
| 05 | [sspr-phone-dependency](queries/05-sspr-phone-dependency.kql) | Assess | Who loses self-service password reset? |
| 06 | [exception-group-sms-voice-only](queries/06-exception-group-sms-voice-only.kql) | Migrate | Who must stay in the SMS and voice exception group, and when and where did they last use it? |
| 04 | [migration-progress](queries/04-migration-progress.kql) | Verify | Is the migration actually moving? |

Start with **02** to size the problem, then **01** for the worklist. When you
switch SMS and voice off for everyone except an exception group, **06** lists
the users who must stay in that group: their only successful MFA in the last
90 days was SMS or voice.

## How to run

Microsoft Sentinel or Log Analytics, against a workspace receiving Entra ID
sign-in logs. In the Entra admin center, the diagnostic settings must export
`SignInLogs` (and `AuditLogs` for query 05).

Each query declares its parameters as `let` statements at the top. Adjust the
time range there; the default is the last 30 days, or 90 days for queries 04
and 06. A 90-day range needs at least 90 days of `SigninLogs` retention in the
workspace.

Query 06 reads the account's country from `IdentityInfo` when Microsoft
Sentinel UEBA is enabled. Without it the query still runs, and only the
sign-in location country is filled in.

Queries are plain `.kql` files. Paste one into the query window and run it.

## Reading the results correctly

**Sign-in logs show what people used, not what they registered.** A user who
holds an unused Authenticator registration is safe but still appears in query
01. Before contacting anyone, confirm against registration data:

- The **Authentication methods activity** report in the Entra admin center.
- Microsoft's [SMS and voice usage analyzer](https://github.com/microsoft/entra-sms-voice-usage-analyzer),
  which reads registration rather than usage.

Use these queries to find and prioritise candidates, and registration data to
confirm before acting. The two answer different questions and you need both.

**`UnclassifiedMethods` and `NeedsReview` are not cosmetic.** Entra adds method
names over time. A non-empty value means the classification lists need
extending, and that the at-risk figure may be wrong in either direction until
they are. Never read an unclassified method as confirmation of risk.

**Query 03 cannot read directory roles.** `SigninLogs` has no role column, so
Global Administrators are inferred from the applications used. Confirm actual
role membership in Entra or through Microsoft Graph.

## Microsoft guidance

- [Passkeys by default and retirement of Microsoft-provided SMS and voice authentication](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-sms-voice-retirement)
- [Frequently asked questions](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-sms-voice-retirement-faq)
- [Plan a passkey deployment](https://learn.microsoft.com/en-us/entra/identity/authentication/how-to-deploy-phishing-resistant-passwordless-authentication)
- [Choose a telephony provider](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-phone-providers)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE)
