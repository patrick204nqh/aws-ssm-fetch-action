# aws-ssm-fetch

GitHub Actions composite action: assume an AWS IAM role via GitHub OIDC, fetch one-or-more SSM SecureString parameters in batched calls, mask their values, and emit them to `$GITHUB_ENV` for subsequent steps.

Designed for repos that have moved off plaintext GitHub Actions secret fanout and onto runtime SSM reads.

## Pre-reqs

- An AWS IAM role with:
  - Trust policy allowing GitHub OIDC `AssumeRoleWithWebIdentity` from the calling repo + sub claim (e.g. `repo:OWNER/REPO:environment:production`).
  - Inline policy granting `ssm:GetParameter` / `GetParameters` / `GetParametersByPath` on the specific parameter ARNs.
- The role's ARN and AWS region published to the calling repo as Actions Variables (typical names: `AWS_ROLE_ARN`, `AWS_REGION`) — or hardcoded in the workflow.
- Workflow job has `permissions: { id-token: write, contents: read }` (so the OIDC token can be issued).
- Workflow trigger context matches the role's trust-policy `sub` patterns (typically `environment: production` on the job, or a push to an allowed branch).

## Usage

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  release:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: patrick204nqh/aws-ssm-fetch-action@v1
        with:
          role-arn: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
          params: |
            /providers/messaging/slack/main/bot-token            SLACK_BOT_TOKEN
            /providers/messaging/slack/main/channel-id           SLACK_CHANNEL_ID
            /providers/package-registry/rubygems/main/api-key    RUBYGEMS_API_KEY

      # Subsequent steps see $SLACK_BOT_TOKEN, $SLACK_CHANNEL_ID, $RUBYGEMS_API_KEY in env.
```

### Inputs

| Name | Required | Description |
|---|---|---|
| `role-arn` | yes | IAM role ARN to assume. Typically `${{ vars.AWS_ROLE_ARN }}`. |
| `aws-region` | yes | AWS region containing the SSM parameters. Typically `${{ vars.AWS_REGION }}`. |
| `params` | yes | Newline-separated `<ssm-path> <whitespace> <ENV_VAR>` pairs. See example above. |

### What it does

1. Calls `aws-actions/configure-aws-credentials@v4` to assume the OIDC role.
2. Parses the `params` block into (path, env-var) tuples.
3. Calls `aws ssm get-parameters --with-decryption` in batches of 10 (SSM API limit).
4. For each returned value:
   - Calls `::add-mask::` on each line of the value, so multi-line values (PEM private keys) stay masked when echoed.
   - Appends a heredoc-formatted `VAR<<delim ... delim` block to `$GITHUB_ENV`, so subsequent steps see the value.

## Versioning

- `@v1` — moving tag, follows the latest `v1.x` release. Recommended for most consumers.
- `@v1.2.3` — immutable pin. Use when you want explicit rollbacks.
- `@<sha>` — full SHA. Maximally pinned; consider for security-sensitive workflows.

## License

MIT — see `LICENSE`.
