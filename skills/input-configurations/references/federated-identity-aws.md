# Federated Identity for AWS integrations

Complete guide for enabling Federated Identity (Cloud Connectors) on an
agentless-eligible AWS integration package in elastic/integrations. Captures
the pattern proven on the `aws` package (elastic/integrations#19828) and
applicable to any AWS integration.

- Kibana IaCP broker route: `POST /internal/fleet/iac_provider/render_template`
- Rollout tracking: elastic/ingest-dev#8812 (per-package), elastic/ingest-dev#8804 (skill origin)

Template generation is IaC Provider (IaCP)-first: the `cloud-iac-provisioner`
service renders IAM role templates at onboard time from `provider_permissions`
declarations. The static `federated-identity-aws.yml` CFT in elastic/cloudbeat
is the **fallback** — Kibana opens it via `iac_template_url` when the IaCP
render fails (422/502). Both are fed from the same declarations.

> **Warning when reading the `aws` package as a reference:** its
> `identity_federation` var_groups option carries an `iac_template_url`
> pointing at the legacy GuardDuty-only CFT
> (`cloudformation-cloud-connectors-guardduty-*.yml`). Do not copy that URL —
> new packages point at the incremental `federated-identity-aws` template.

---

## §0 — Pre-flight checks

Run these checks before touching any file. Their output determines which
sections apply.

### §0.1 Fetch the manifest

```bash
cat packages/<PACKAGE>/manifest.yml
```

Record the current values:

| Field | Federation requirement | Why |
|-------|----------------------|-----|
| `format_version` | `>= 3.6.4` | `var_groups` requires 3.6.1; `provider_permissions` requires 3.6.4 |
| `conditions.kibana.version` | `^9.4.0` or higher | `auth.aws` requires Agent 9.4.0+ |
| `conditions.agent.version` | `^9.4.0` or higher | same |

Values that fall short will be bumped in §1.1 — note them now rather than
treating them as blockers here.

> **Escalation: the bump would abandon users on a supported stack.** If the
> current Kibana constraint covers a stack line that `^9.4.0` drops (e.g.
> `^8.16.5 || ^9.0.0`), do NOT bump silently — that is a product decision.
> Stop and report it, then suggest the branching strategy from
> elastic/ingest-dev#8788:
>
> 1. Bump the package **major version** on `main`.
> 2. Cut a long-running `backport-<package>-<N>.x` branch from the last
>    release commit, keeping the old floor for old-stack users.
> 3. EPR version routing does the rest: old stacks resolve the backport
>    line, new stacks resolve `main`.
> 4. Split the PRs: spec bump first, federation change second.
>
> Requires sign-off from every team in CODEOWNERS before any constraint change.

> **Expect validators to fire on old files.** A `format_version` jump to 3.6.x
> activates stricter pipeline validation (processor `tag` and `on_failure`
> format). Apply only the `format_version` bump first (§1.1), run
> `elastic-package lint` to size the cleanup — on `aws_securityhub` the
> 3.5.0 → 3.6.4 bump surfaced 514 pre-existing pipeline violations. Land those
> as a separate mechanical hygiene PR (precedent: elastic/integrations#19824
> for the `aws` package), then rebase the federation change on top.

### §0.2 Classify inputs (federation vs agentless)

Federation and agentless are not the same gate. Classify every input in each
policy template into exactly one bucket:

| Bucket | Meaning | Examples | Later action |
|--------|---------|----------|--------------|
| **Federation-eligible** | Can run agentless **and** supports Identity Federation (`auth.aws` / Cloud Connectors) | `cel`, `httpjson`, `aws/metrics` (any `*metrics` variant), `aws-cloudwatch` | Remain visible under Identity Federation |
| **Agentless, not federation** | Can run agentless with other credential methods, but **not** with Identity Federation | `aws-s3` | §2 `hide_in_var_group_options` — do **not** pin to `deployment_modes: ["default"]` |
| **Not agentless** | Cannot run agentless at all | `awsfargate/metrics`, and any other unlisted type that hard-requires a local agent/runtime | §1.4b `deployment_modes: ["default"]` |

If **no** inputs are federation-eligible, stop. The report must name each
blocking input type and the upstream dependency that would unblock it:

> `<package>` cannot be federated: its only input is `aws-s3`, which does not
> support Identity Federation (`auth.aws` / Cloud Connectors). Blocked until
> `aws-s3` gains federation support; no package-level change can work around
> that. (The input may still run agentless with access keys — that does not
> make the package federation-eligible.)

### §0.3 Audit the credential vars

AWS auth vars are declared at different levels depending on the package:

- **Package level** (`vars:`) — e.g. the `aws` package
- **Input level** (`policy_templates[].inputs[].vars`) — e.g. `aws_securityhub`

Record which shape this package uses. Then classify every existing credential
var:

| Bucket | Vars | Fate |
|--------|------|------|
| **Federation-required** | `role_arn`, `external_id` | Add any missing (§1.2). `supports_identity_federation` is always added new. |
| **Agentless-compatible** | `access_key_id`, `secret_access_key` | Kept; form the `direct_access_key` option (§1.3). |
| **Agent-only** | `session_token`, `shared_credential_file`, `credential_profile_name` | Kept; grouped into options with `hide_in_deployment_modes: [agentless]` (§1.3). |
| **Auxiliary** | `assume_role_duration`, `proxy_url`, `ssl`, ... | Left outside `var_groups` — tuning knobs, not credential selectors. |

**If no AWS credential vars exist at all**, verify whether the input supports
`auth.aws`:

1. Check the beats module / agent implementation. An input reading a local
   endpoint (e.g. `awsfargate/metrics` reading the ECS task metadata endpoint)
   makes no AWS API calls — credential vars would be dead configuration.
2. If the input supports AWS auth but the package never exposed vars: **ask
   the user** whether to add the minimum required set:
   - `role_arn`, `external_id`, `supports_identity_federation`
     (the `identity_federation` option)
   - `access_key_id`, `secret_access_key` (the `direct_access_key` option —
     the only other option visible in agentless mode)

   On yes: add these per §1.2, emit only those two var_group options in §1.3,
   and add the full `auth.aws` block to the stream templates per §4.
3. If the input does not support AWS auth: stop with an actionable report and
   flag the package for removal from the rollout scope.

> **Unverified assumption:** the package-spec validator accepts input-level var
> references in `var_groups`, but Fleet UI rendering of this has not been
> end-to-end verified. If the UI misbehaves, fall back to hoisting the auth
> vars to package level.

### §0.4 Check for existing structures

- If `var_groups:` exists **and contains an `identity_federation` option**,
  skip §1.3.
- If `supports_identity_federation` already exists as a var, skip adding it
  in §1.2.
- If `deployment_modes.agentless.enabled: true` already exists on the target
  policy templates, skip §1.4.

Check for expected content, not just key presence.

### §0.5 Find tests that assert the behavior you are about to remove

```bash
grep -rl "access_key\|Authorization\|X-Amz" packages/<pkg>/data_stream/<stream>/_dev/test/
```

- **`scripts/`**: script tests asserting a hand-rolled credential gate's error
  message will break when `auth.aws` removes that gate. Precedent:
  `aws/config`'s `missing_credentials` test asserted "access_key_id and
  secret_access_key required" — a message the migration deletes. Rewrite to
  the new contract: the unauthenticated request fails at the AWS API and the
  program surfaces its own stable error wrapper as an error event (assert the
  wrapper prefix, not the environment-dependent AWS exception text), with no
  data events.
- **`policy/`**: rendered-policy snapshots embedding old hbs output
  (manual `Authorization`/`X-Amz-Date` transforms) go stale on hbs changes —
  regenerate them.

---

## §1 — Package `manifest.yml` changes

Schema authority for `var_groups`, `provider_permissions`, and the
`format_version` / conditions floors is `package-spec` ->
`references/var-groups-and-provider-permissions.md`. This section is the
AWS Federated Identity procedure that applies those constructs.

**Order matters:** bump `format_version` / conditions first (§1.1), then add
vars and `var_groups`. `var_groups` needs `>= 3.6.1` and
`provider_permissions` needs `>= 3.6.4` — adding them on a `3.4.2` package
fails validation.

### §1.1 Bump `format_version` and conditions

Do this **before** adding `var_groups` or `provider_permissions`.

```yaml
format_version: "3.6.4"
```

```yaml
conditions:
  kibana:
    version: "^9.4.0"
  # auth.aws support in CEL and HTTPJSON inputs requires Elastic Agent 9.4.0+.
  agent:
    version: "^9.4.0"
```

Run `elastic-package lint` immediately after this bump to surface pre-existing
violations before making any other changes. If lint fails on files this change
does not own, land a separate hygiene PR first (§0.1), then continue.

### §1.2 Add missing federation vars

Add alongside the existing auth vars (same level as identified in §0.3),
after the last auth var.

Always add:

```yaml
  - name: supports_identity_federation
    type: bool
    title: Supports Identity Federation
    multi: false
    required: false
    show_user: false
```

If `role_arn` or `external_id` are missing, add them too:

```yaml
  - name: role_arn
    type: text
    title: Role ARN
    multi: false
    required: false
    show_user: false
  - name: external_id
    type: text
    title: External ID
    multi: false
    required: false
    show_user: false
    secret: true
    description: External ID to use when assuming a role in another account, see [the AWS documentation for use of external IDs](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html)
```

### §1.3 Add `var_groups`

At the package level, before `policy_templates:` (after the `vars:` block,
or directly before `policy_templates:` if there are no package-level vars):

```yaml
var_groups:
  - name: credential_type
    required: true
    title: Setup Access
    selector_title: Preferred method
    options:
      - name: identity_federation
        title: Identity Federation
        vars: [role_arn, external_id, supports_identity_federation]
        hide_in_deployment_modes: [default]
        provider: aws
        iac_template_url: https://console.aws.amazon.com/cloudformation/home#/stacks/quickcreate?templateURL=https://elastic-cspm-cft.s3.eu-central-1.amazonaws.com/cloudformation-federated-identity-aws-<KIBANA_FLOOR_MINOR>.yml&param_ElasticResourceId=RESOURCE_ID
      - name: direct_access_key
        title: Direct Access Keys
        vars: [access_key_id, secret_access_key]
      - name: temporary_access_key
        title: Temporary Access Keys
        vars: [access_key_id, secret_access_key, session_token]
        hide_in_deployment_modes: [agentless]
      - name: assume_role
        title: Assume Role
        vars: [role_arn]
        hide_in_deployment_modes: [agentless]
      - name: assume_role_external_id
        title: Assume Role with External ID
        vars: [role_arn, external_id]
        hide_in_deployment_modes: [agentless]
      - name: shared_credentials
        title: Shared Credentials
        vars: [shared_credential_file, credential_profile_name]
        hide_in_deployment_modes: [agentless]
```

Build the options list from the §0.3 audit — emit only options whose vars the
package actually declares. Every **agent-only** option carries
`hide_in_deployment_modes: [agentless]`. **Auxiliary** vars stay outside
`var_groups`. If the package has extra credential vars that don't match any
canonical option, propose a new agent-only option and flag it for reviewer
attention.

**`iac_template_url`:** replace `<KIBANA_FLOOR_MINOR>` with the package's
Kibana floor minor (e.g. `9.4.0`), matching the `aws` package convention.
Keep `RESOURCE_ID` literal — Kibana substitutes it. The template is published
by cloudbeat's `publish_cft.sh` as
`cloudformation-federated-identity-aws-<version>.yml`. Kibana renders via
IaCP first; when that fails (422/502) it falls back to opening this
quick-create URL.

> The fallback only works once the template is published to S3. Until then the
> URL 404s on fallback — the integrations PR must note that dependency.

Package-spec validator rules:
- Every name in `options[].vars` must exist as a var at any level.
- Vars referenced by a var_group must not have `required: true`.

### §1.4 Enable agentless deployment (if not already enabled)

Skip if `deployment_modes.agentless.enabled: true` already exists (§0.4).

**a. Enable on each target policy template:**

```yaml
    deployment_modes:
      default:
        enabled: true
      agentless:
        enabled: true
        release: beta             # or ga; evaluated in Kibana 9.5.0+
        organization: <org>       # e.g. security
        division: engineering
        team: <owning-team>       # from CODEOWNERS
```

Optional fields on the agentless block: `is_default: true` (make agentless
the default mode when both are offered) and `resources.requests.{memory,cpu}`
if the workload needs more than the platform default.

**b. Restrict inputs that cannot run agentless** (the **Not agentless** bucket
from §0.2) to agent-based mode:

```yaml
      - type: awsfargate/metrics
        title: Collect AWS Fargate metrics
        description: Collecting metrics from the local ECS task metadata endpoint.
        deployment_modes: ["default"]
```

Do **not** use this pin for **Agentless, not federation** inputs such as
`aws-s3` — those stay available in agentless mode with non-federation
credentials and are gated only via §2 `hide_in_var_group_options`.

Enabling agentless is a product decision — the owning team commits to
operating fully-managed deployments. Call it out explicitly in the PR
description and get the owning team's sign-off before shipping.

---

## §2 — Policy template input gating

For inputs in the **Agentless, not federation** bucket from §0.2 (e.g. `aws-s3`
used with access keys in a mixed-input policy template), add
`hide_in_var_group_options` under the input so it is hidden when Identity
Federation is selected:

```yaml
      - type: aws-s3
        title: Collect <Package> logs from AWS S3 or SQS
        description: Collecting <Package> logs via AWS S3/SQS.
        hide_in_var_group_options:
          credential_type: [identity_federation]
```

Leave **Federation-eligible** inputs (`cel`, `httpjson`, `aws/metrics`,
`aws-cloudwatch`) without this key — they must remain visible when Identity
Federation is selected.

Two distinct gates — don't conflate them:

| Situation (§0.2 bucket) | Mechanism |
|-------------------------|-----------|
| Agentless, not federation (e.g. `aws-s3`) | `hide_in_var_group_options` with `credential_type: [identity_federation]` |
| Not agentless (e.g. `awsfargate/metrics`) | `deployment_modes: ["default"]` on the input (§1.4b) |

An input pinned to `["default"]` never sees the Identity Federation option
(it's hidden in default mode), so it does not also need the
`hide_in_var_group_options` gate.

---

## §3 — Declare `provider_permissions`

Schema and floors: `package-spec` ->
`references/var-groups-and-provider-permissions.md`. Below is how to
populate them for Federated Identity.

The IaCP renders the IAM role template from these — single source of truth
for what the generated role can do. Requires `format_version >= 3.6.4`.

Declare at the **narrowest level that needs it** (package, policy_template,
input, or data_stream). Entries across levels are accumulated and deduplicated.

```yaml
# Input-level example:
      - type: cel
        title: Collect AWS Security Hub logs via API
        description: Collecting AWS Security Hub logs via API.
        provider_permissions:
          - provider: aws
            description: Security Hub read access for findings collection.
            permissions:
              - name: securityhub:GetFindings
```

Optional `roles` attach managed policies alongside inline permissions:

```yaml
        provider_permissions:
          - provider: aws
            description: Read-only security auditing.
            roles:
              - name: SecurityAudit
                id: arn:aws:iam::aws:policy/SecurityAudit
            permissions:
              - name: securityhub:GetFindings
```

Determine actions from the input's actual API calls (agent stream template,
CEL program, or beats module docs) — do not guess. Prefer the minimal
read-only set. Watch for actions whose IAM authorization name differs from the
API operation name — these are the silent killers:

> `aws_securityhub` calls `GetFindingsV2` but the IAM action is
> `securityhub:GetFindings`. A reviewer who "corrects" it to
> `securityhub:GetFindingsV2` breaks the role silently.

Verify against the AWS API Reference and cite the primary source in §9.1.

### §3.1 Mirror into the static CFT (elastic/cloudbeat)

The static `deploy/cloudformation/federated-identity-aws.yml` grows
incrementally — one block per federated integration, never granting ahead of
a declaration. Submit as a separate PR (one per integration) that links to the
integrations PR.

For each `provider_permissions` entry with `provider: aws`:

**`permissions` → inline policy resource** (CamelCase `Elastic` prefix):

```yaml
  ElasticAwsSecurityHub:
    Type: AWS::IAM::Policy
    Properties:
      PolicyName: ElasticAwsSecurityHub
      Roles:
        - !Ref ElasticFederatedIdentityRole
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          # <package>: <description from provider_permissions>
          - Sid: ElasticAwsSecurityHubRead
            Effect: Allow
            Action:
              - securityhub:GetFindings
            Resource: '*'
```

**`roles` → append each `id` ARN to `ManagedPolicyArns`** with a comment
naming the declaring package.

Validate:

```bash
cfn-lint deploy/cloudformation/federated-identity-aws.yml
# or via pre-commit:
pre-commit run cfn-python-lint --files deploy/cloudformation/federated-identity-aws.yml
```

Apply this step in the local cloudbeat checkout only — the integrations PR
must not depend on the unpublished template (the fallback URL 404s until
cloudbeat ships and publishes).

---

## §4 — Agent stream template (`*.yml.hbs`) changes

For each eligible input at `data_stream/<name>/agent/stream/<input>.yml.hbs`:

```handlebars
auth.aws:
{{#if access_key_id}}
  access_key_id: {{access_key_id}}
{{/if}}
{{#if secret_access_key}}
  secret_access_key: {{secret_access_key}}
{{/if}}
{{#if session_token}}
  session_token: {{session_token}}
{{/if}}
{{#if shared_credential_file}}
  shared_credential_file: {{shared_credential_file}}
{{/if}}
{{#if credential_profile_name}}
  credential_profile_name: {{credential_profile_name}}
{{/if}}
{{#if role_arn}}
  role_arn: {{role_arn}}
{{/if}}
{{#if external_id}}
  external_id: {{external_id}}
{{/if}}
{{#if assume_role_duration}}
  assume_role.duration: {{assume_role_duration}}
{{/if}}
{{#if assume_role_expiry_window}}
  assume_role.expiry_window: {{assume_role_expiry_window}}
{{/if}}
{{#if supports_identity_federation}}
  use_cloud_connectors: {{supports_identity_federation}}
{{/if}}
```

Apply the minimum change:

- If `auth.aws:` already exists, append only the final
  `{{#if supports_identity_federation}}` block.
- If there is no `auth.aws:` block, add the full block above, trimming
  `{{#if}}` clauses for vars the package doesn't declare.

Regenerate any `policy/` snapshots in `_dev/test/` identified in §0.5.

---

## §5 — Kibana policy-group membership (conditional)

`CLOUD_CONNECTOR_PERMISSION_ALLOWLIST` in
`x-pack/platform/plugins/shared/fleet/common/constants/cloud_connector.ts`
controls connector **sharing**, not rendering — a package absent from it still
renders fine, standalone, with its own cloud connector. What it controls is
connector sharing: integrations in the same policy group share one cloud
connector, so the rendered template must grant the whole group's permissions
at once.

**New package should have its own standalone connector** (the default): no
Kibana change needed. Skip this section.

**New package should share a connector with an existing group:** open a Kibana
PR adding an entry to the relevant group, e.g. `aws_global_policy_group`:

```typescript
aws_global_policy_group: [
  { provider: AWS_CLOUD_PROVIDER, package: 'aws', policyTemplate: 'guardduty' },
  { provider: AWS_CLOUD_PROVIDER, package: '<package>', policyTemplate: '<policy_template>' },
],
```

If in doubt, ship standalone first — group membership can be added later.

---

## §6 — Changelog entry

Bump the package **minor version** and prepend to `changelog.yml`:

```yaml
- version: "X.Y.0"
  changes:
    - description: Enable Identity Federation (Cloud Connectors) authentication for agentless deployments.
      type: enhancement
      link: https://github.com/elastic/integrations/pull/<PR_NUMBER>
```

Update `version:` in `manifest.yml` to match. Add a separate changelog entry
if §1.4 enabled agentless for the first time.

---

## §7 — Validation checklist

- [ ] `elastic-package lint` — no errors
- [ ] `elastic-package build` — builds cleanly
- [ ] Fleet UI agentless path shows the Identity Federation option
- [ ] Fleet UI default path hides the Identity Federation option
- [ ] Ineligible inputs do not appear when Identity Federation is selected
- [ ] On serverless, onboard via Identity Federation: IaCP renders the IAM role template, user deploys, data arrives
- [ ] Deployed role's policy matches `provider_permissions` (no missing actions at collection time)

---

## §8 — PR checklists

**elastic/integrations PR:**
- [ ] Title: `[<package>] Enable Identity Federation for agentless deployments`
- [ ] Body links to the rollout tracking issue (elastic/ingest-dev#8812)
- [ ] Agentless enablement (if new) called out explicitly
- [ ] Dependency on cloudbeat CFT publishing documented
- [ ] `changelog.yml` entry added with correct version bump
- [ ] CODEOWNERS confirmed

**elastic/kibana PR** (only if joining a policy group, §5):
- [ ] Adds package to `CLOUD_CONNECTOR_PERMISSION_ALLOWLIST`
- [ ] Links to the integrations PR

**elastic/cloudbeat PR** (§3.1 CFT mirror):
- [ ] One PR per integration, stacked on the incremental-baseline PR
- [ ] Links to the integrations PR whose `provider_permissions` it mirrors
- [ ] `cfn-lint` passes

---

## §9 — Human validation handoff (always generate this)

Produce a checklist tailored to the target package. Research package-specific
answers — never emit generic placeholders.

### §9.1 Permission claims to verify

For every IAM action in §3:

- The action, the API call it authorizes, and the **primary-source citation**
  (AWS API Reference page).
- Any action whose name differs from the API operation name, flagged
  prominently. These are the silent killers — e.g. Security Hub's
  `GetFindingsV2` operation authorizes as `securityhub:GetFindings`; a
  reviewer who "corrects" it to `securityhub:GetFindingsV2` breaks the role.
- Whether the set is least-privilege: no action the collector doesn't call,
  no `*` wildcards, `Resource: '*'` justified (most read/list APIs are not
  resource-scopable — say so explicitly if that's the justification).

### §9.2 Cross-PR consistency to verify

- IAM actions in `provider_permissions`, the cloudbeat CFT, and the
  IaCP-rendered template are **identical sets** — drift means fallback and
  primary paths grant different permissions.
- `iac_template_url` version segment matches the package's Kibana floor minor.
- Merge order documented: hygiene PR → integrations PR; CFT baseline → CFT
  per-integration PR.

### §9.3 End-to-end test plan

Embed this as an `## E2E test plan` section in the integrations PR body —
reviewers and testers must see it without hunting through comments, and the
hold-merge gate references it. State what the run uniquely proves (e.g. that
the `auth.aws` signer handles the program's request shape against real AWS —
mocks don't verify signatures), and end with a regression line covering the
legacy credential paths the change touches. Cover:

1. **Prerequisite AWS resources** — what must exist in the test account.
2. **Event generation** — fastest trigger for ingestible records.
3. **Latency expectations** — how long before declaring failure.
4. **Where to look** — exact data stream and one field to sanity-check.
5. **Failure probes** — what auth failure looks like vs. authorized-but-empty.

Worked example for `aws_securityhub`:

> 1. Enable Security Hub; enable AWS Foundational Security Best Practices.
> 2. Fastest data: automated control checks within ~2h of enablement;
>    deliberately noncompliant resources (e.g. unencrypted S3 bucket) accelerate
>    real findings. GuardDuty sample findings also flow in if GuardDuty
>    integration is on.
> 3. Budget 2h before concluding failure.
> 4. `logs-aws_securityhub.finding-*`; check `aws_securityhub.finding.class_uid`.
> 5. Auth failure: agent CEL log shows HTTP 403 with `AccessDeniedException` on
>    `POST /findingsv2`; authorized-but-empty returns HTTP 200 `"Findings": []`.

### §9.4 Open assumptions

Restate anything marked unverified during the run (e.g. the Fleet UI
input-level var_groups rendering from §0.3) so the human knows what hasn't
been proven end-to-end.
