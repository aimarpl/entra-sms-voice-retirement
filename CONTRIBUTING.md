# Contributing

Queries supporting the retirement of SMS and voice authentication in
Microsoft Entra ID. Contributions that help organisations find and reduce
that dependency are welcome.

## Scope

In scope: finding users and workloads dependent on SMS or voice, measuring
exposure, tracking migration to phishing-resistant methods, and verifying
completion before the retirement dates.

Out of scope: general-purpose identity queries unrelated to this retirement.

## Query file format

One query per `.kql` file in `queries/`, numbered by the order you would
run them. Each file opens with a metadata header:

```text
// Title: Short description of what it returns
// Platform: Microsoft Sentinel / Log Analytics
// Phase: 1 - Assess | 2 - Migrate | 3 - Verify
// Tags: entra-id, sms, voice, retirement, ...
// Description: What question this answers and when to use it.
// Data sources: SigninLogs
// Parameters: StartTime, EndTime
// Notes: Caveats, permissions, anything that could be misread.
```

Parameters are `let` statements at the top of the query so they can be changed
without reading the body.

## Correctness rules

These exist because each one has produced a wrong answer in practice.

- **Filter for success.** `AuthenticationDetails` holds one entry per
  authentication step, each with its own `succeeded` flag. Without
  `ResultType == "0"` and `tobool(AuthStep.succeeded) == true`, a failed SMS
  prompt counts as SMS usage.
- **Identify users by `UserId`, never `Identity`.** `Identity` is a display
  name: two people sharing one merge into a single row, and a renamed user
  becomes two.
- **Classify every method, then aggregate, then filter.** Narrowing to SMS and
  voice before `summarize` discards the user's other methods and reports
  someone with a passkey as SMS-only. This is the most damaging bug class here,
  because the output is a list of real colleagues to contact.
- **Surface unrecognised methods.** Include an `Unclassified` bucket and
  project it. Entra adds method names over time, and silently treating an
  unknown method as weak produces false positives.
- **`trim(@"\s+", ...)`, not `trim(@" ", ...)`.** The second trims one literal
  space character.
- **Do not claim registration state from sign-in logs.** They show use, not
  registration. Say so in `Notes` where it could be misread.

## Before opening a pull request

- Run the query against a real workspace and check the output is what the
  title claims.
- Confirm any date or scope statement against the
  [Microsoft retirement guidance](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-sms-voice-retirement).
  The February and July populations are easy to mix up.
- Use placeholders (`example.com`, `GroupName`) rather than real tenant values.
  No tenant identifiers, real domains, user names or object IDs.
- Check Markdown formatting: `npx --yes markdownlint-cli2 "**/*.md"`.

## Commits

[Conventional Commits](https://www.conventionalcommits.org/):

- `feat(queries): add conditional access dependency query`
- `fix(queries): correct method list in migration progress`
- `docs: clarify the internal guest retirement date`
