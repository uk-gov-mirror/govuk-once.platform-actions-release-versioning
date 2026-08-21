# Reusable GitHub Actions workflows

This repository contains centrally maintained GitHub Actions workflows for GOV.UK App services.

It initially provides two capabilities:

- creating semantic versions, Git tags and GitHub releases; and
- notifying a caller-selected Slack channel when a major or minor release has successfully reached production.

Application repositories remain responsible for their own quality checks, builds, tests and deployments. These workflows provide a consistent release contract without centralising service-specific deployment logic.

## Available workflows

| Workflow | Purpose |
| --- | --- |
| `.github/workflows/semantic-release.yml` | Determines whether a release is required, creates the Git tag and GitHub release, and returns explicit release outputs. |
| `.github/workflows/release-notification.yml` | Publishes a major or minor production release notification to an SNS topic supplied by the caller. |

## Intended release flow

The calling repository should use the workflows in this order:

1. Complete its quality checks.
2. Call `semantic-release.yml`.
3. Deploy through its service-owned environments.
4. Call `release-notification.yml` after the production deployment succeeds.

The notification workflow only publishes for `major` and `minor` releases. It does not publish notifications for:

- patch releases;
- changes that do not create a semantic release;
- failed releases;
- failed deployments; or
- successful development or staging deployments.

Deployment and release failures are not reported to Slack by these workflows.

## Semantic release

### Usage

```yaml
jobs:
  release:
    name: Create Release
    needs: qualityChecks
    uses: govuk-once/github-actions-workflows/.github/workflows/semantic-release.yml@v1
    permissions:
      contents: read
    with:
      node-version: "24"
      github-app-id: ${{ vars.GH_TAG_RELEASE_APP_ID }}
    secrets:
      GITHUB_APP_PRIVATE_KEY: ${{ secrets.GH_TAG_RELEASE_PRIVATE_KEY }}
```

### Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `node-version` | No | `24` | Node.js version used to install dependencies and run semantic-release. |
| `github-app-id` | Yes | | ID of the GitHub App used to create tags and releases. |

### Secrets

| Secret | Required | Description |
| --- | --- | --- |
| `GITHUB_APP_PRIVATE_KEY` | Yes | Private key belonging to the GitHub App. |

### Outputs

| Output | Description |
| --- | --- |
| `released` | `true` when a release was created; otherwise `false`. |
| `version` | New semantic version without the `v` prefix. Empty when no release was created. |
| `type` | `major`, `minor` or `patch`. Empty when no release was created. |

Callers access the outputs through the release job:

```yaml
${{ needs.release.outputs.released }}
${{ needs.release.outputs.version }}
${{ needs.release.outputs.type }}
```

### Calling-repository requirements

The workflow checks out and runs against the calling repository. Each caller must contain:

- a `pnpm-lock.yaml` file;
- `semantic-release` and its required plugins in `devDependencies`; and
- a semantic-release configuration such as `.releaserc.json`.

For example:

```json
{
  "branches": ["main"],
  "tagFormat": "v${version}",
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/github"
  ]
}
```

The GitHub App must be installed on the calling repository and granted the minimum permissions required to create Git tags and GitHub releases. If the caller's semantic-release configuration comments on issues or pull requests, the App will also need the corresponding permissions.

The workflow uses a GitHub App token instead of the repository's standard `GITHUB_TOKEN`. This allows release-created events to support future event-driven automation and avoids relying on the restricted event behaviour of `GITHUB_TOKEN`.

## Production release notification

### Usage

The notification job should depend on both the semantic-release job and the service's production deployment pipeline:

```yaml
jobs:
  notifyRelease:
    name: Notify Production Release
    needs: [release, deployment]
    uses: govuk-once/github-actions-workflows/.github/workflows/release-notification.yml@v1
    permissions:
      id-token: write
      contents: read
    with:
      application: "FLEX"
      version: ${{ needs.release.outputs.version }}
      type: ${{ needs.release.outputs.type }}
      aws-region: "eu-west-2"
      notification-topic-arn: ${{ vars.RELEASE_NOTIFICATIONS_TOPIC_ARN }}
    secrets:
      deployment-role: ${{ secrets.DEV_DEPLOYMENT_ROLE }}
```

No condition is required in the caller:

- GitHub skips `notifyRelease` if `release` or `deployment` fails.
- The reusable notification workflow skips releases which are not `major` or `minor`.

This keeps the release-notification rule in one centrally maintained place.

### Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `application` | Yes | — | Application name displayed in the notification, for example `FLEX` or `UDP`. |
| `version` | Yes | — | Semantic version without the `v` prefix. |
| `type` | Yes | — | Semantic release type. Only `major` and `minor` are published. |
| `notification-topic-arn` | Yes | — | SNS topic ARN connected to the required notification destination. |
| `aws-region` | No | `eu-west-2` | AWS region containing the SNS topic. |

### Secrets

| Secret | Required | Description |
| --- | --- | --- |
| `deployment-role` | Yes | AWS role assumed through GitHub OIDC. It must be permitted to publish to the supplied SNS topic. |

### Notification content

The notification contains:

- application name;
- release type;
- semantic version;
- confirmation that the release reached production;
- generated GitHub release notes;
- a link to the GitHub release; and
- a link to the successful workflow run.

If AWS authentication or SNS publication fails, the reusable workflow writes a warning to the GitHub Actions run. It does not send a failure message to Slack and does not change the result of an otherwise successful production deployment.

## Selecting the Slack channel

The reusable workflow does not hard-code a Slack channel or an SNS topic. The caller supplies `notification-topic-arn`, allowing different repositories to select different destinations.

The SNS topic must already be associated with the intended Slack channel through the relevant Amazon Q Developer in chat applications or AWS Chatbot configuration.

A service that needs a different destination can set the same variable to an SNS topic connected to its required channel. The AWS role passed as `deployment-role` must have `sns:Publish` permission for that topic.

SNS topic ARNs are treated as configuration rather than credentials and should normally be stored as GitHub Actions variables. If organisational policy classifies the ARN as sensitive, the reusable-workflow interface should be changed to accept it as a named secret instead.

## Complete caller example

```yaml
name: Continuous Deployment

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  qualityChecks:
    name: Quality Checks
    uses: ./.github/workflows/_quality-checks.yml
    # Service-specific inputs, permissions and secrets omitted.

  release:
    name: Create Release
    needs: qualityChecks
    uses: govuk-once/github-actions-workflows/.github/workflows/semantic-release.yml@v1
    permissions:
      contents: read
    with:
      node-version: "24"
    secrets:
      GITHUB_APP_ID: ${{ secrets.GH_TAG_RELEASE_APP_ID }}
      GITHUB_APP_PRIVATE_KEY: ${{ secrets.GH_TAG_RELEASE_PRIVATE_KEY }}

  deployment:
    name: Deploy
    needs: release
    uses: ./.github/workflows/_deployment-pipeline.yml
    permissions:
      id-token: write
      contents: read
    # Service-specific inputs and deployment-role secrets omitted.

  notifyRelease:
    name: Notify Production Release
    needs: [release, deployment]
    uses: govuk-once/github-actions-workflows/.github/workflows/release-notification.yml@v1
    permissions:
      id-token: write
      contents: read
    with:
      application: "SERVICE_NAME"
      version: ${{ needs.release.outputs.version }}
      type: ${{ needs.release.outputs.type }}
      aws-region: "eu-west-2"
      notification-topic-arn: ${{ vars.RELEASE_NOTIFICATIONS_TOPIC_ARN }}
    secrets:
      deployment-role: ${{ secrets.DEV_DEPLOYMENT_ROLE }}
```

The local `_deployment-pipeline.yml` remains owned by the calling service. It should only complete successfully after the production deployment has succeeded.

## Repository access

For private repositories, configure this repository under **Settings → Actions → General → Access** so approved organisation repositories can call its workflows.

Calling repositories must also permit actions and reusable workflows from this repository under their GitHub Actions policy.

## AWS OIDC requirements

The role used by the notification workflow must:

- trust GitHub's OIDC provider;
- permit the intended calling repositories and reusable workflow;
- allow `sns:Publish` only to approved notification topics; and
- avoid broader AWS permissions that are not required for notification delivery.

Where appropriate, use the `job_workflow_ref` OIDC claim to restrict role assumption to the notification workflow in this repository.

## Versioning and compatibility

Consumers should reference an approved major version:

```yaml
uses: govuk-once/github-actions-workflows/.github/workflows/semantic-release.yml@v1
```

Workflow releases should use immutable semantic-version tags such as `v1.0.0`. A maintained `v1` tag may point to the latest backwards-compatible `v1` release.

The following changes are considered breaking and require a new major version:

- removing or renaming an input, output or secret;
- changing the meaning or format of an output;
- requiring additional caller permissions;
- changing notification eligibility; or
- changing behaviour in a way that requires caller modifications.

Callers should not reference `@main` in production workflows.

## Maintenance

- Third-party actions must be pinned to full commit SHAs.
- Dependabot should be enabled for GitHub Actions dependencies.
- Changes require review from the repository CODEOWNERS.
- Changes should be tested for major, minor, patch and no-release scenarios.
- Notification tests must verify routing without posting unintentionally to an unrelated production Slack channel.
