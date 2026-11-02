# Comment Resolver Agent

A reusable GitHub Actions workflow that resolves pull request review comments with [Gemini CLI](https://github.com/google-gemini/gemini-cli), included as a submodule for reference. Submit a review with inline comments, and the agent applies them and pushes the fix to the pull request's branch.

## Why This Workflow

The stock [run-gemini-cli](https://github.com/google-github-actions/run-gemini-cli) assistant needs an `@gemini-cli` mention and an `/approve` reply, then opens a new pull request. This workflow differs:

- **No mention needed:** submitting a review starts it. Approvals and empty reviews are ignored.
- **Same branch:** the fix is pushed to the reviewed pull request, not to a new one.
- **Checks run normally:** pushes come from your GitHub App, so your checks run on them, see [Authentication](#authentication).
- **Sandboxed model:** Gemini only reads and edits files. It has no shell, network, or token.

## How It Works

1. A review is submitted on an open pull request from the same repository, by an owner, member, or collaborator.
2. The workflow mints a short-lived GitHub App token and checks out the pull request branch without persisting the token.
3. It saves the review summary and inline comments (file, line, text, etc.) to `.resolver/review-comments.json`.
4. Gemini CLI edits the files, then writes a commit message and a summary. The action runs Gemini with `--yolo`, which approves every tool call without asking, so the workflow's `tools.core` allowlist gives it file tools only.
5. The workflow commits as `<app-slug>[bot]` and pushes. It refuses to push changes under `.github/` or to its own files.
6. It posts a summary comment on the pull request: the commit, plus one bullet per comment.

## Authentication

The "Mint app token" step authenticates as the GitHub App:

1. The workflow reads the App's private key from the `APP_PRIVATE_KEY` secret.
2. It signs a JWT with the key. The JWT lasts 10 minutes and only proves the App's identity: it can't access repositories.
3. It exchanges the JWT for an installation token. The token expires after 1 hour, works only on repositories where the App is installed, and has only the App's permissions.
4. Later steps use the token, so the push and the comment appear as `<app-slug>[bot]`.

The App's permissions cover what those steps do:

- **Contents:** pushing the fix.
- **Pull requests:** reading the review and its comments.
- **Issues:** posting the summary comment.

**Why not `GITHUB_TOKEN`:** GitHub ignores its pushes when triggering workflows, to prevent loops. Your checks never run on the fix, so a required check blocks the pull request.

**Why not a personal access token:** it triggers workflows, but:

- **Acts as you:** the bot's pushes and comments can't be told apart from yours.
- **Long-lived:** a leaked token works until someone revokes it.
- **Broad:** a classic token covers all your repositories, and a fine-grained one still needs manual renewal.
- **Tied to your account:** if your access changes, every caller breaks.

## Setup

### 1. Create a GitHub App

Go to **Settings → Developer settings → GitHub Apps → New GitHub App**:

| Setting                                | Value                                        |
| -------------------------------------- | -------------------------------------------- |
| Webhook → Active                       | Unchecked                                    |
| Where can this App be installed?       | Only on this account                         |
| Repository permissions → Contents      | Read and write                               |
| Repository permissions → Pull requests | Read and write                               |
| Repository permissions → Issues        | Read and write (to comment on pull requests) |

Leave every other permission at "No access", including Workflows, so the agent can't change workflow files. Its commits appear as `<app-name>[bot]`.

After creating it, note the **Client ID**, generate a **private key**, and use **Install App** to install it on the repositories that should use the agent.

### 2. Add Credentials to Each Repository

| Name              | Kind                | Value                                                             |
| ----------------- | ------------------- | ----------------------------------------------------------------- |
| `APP_CLIENT_ID`   | Variable            | The App's Client ID                                               |
| `APP_PRIVATE_KEY` | Secret              | Contents of the `.pem` private key                                |
| `GEMINI_API_KEY`  | Secret              | A key from [Google AI Studio](https://aistudio.google.com/apikey) |
| `GEMINI_MODEL`    | Variable (optional) | A model ID, see [Models](#models)                                 |

```bash
gh variable set APP_CLIENT_ID --body "<client-id>"
gh secret set APP_PRIVATE_KEY < path/to/private-key.pem
gh secret set GEMINI_API_KEY
```

### 3. Add the Caller Workflow

Copy [`examples/resolve-review-comments.yml`](examples/resolve-review-comments.yml) to `.github/workflows/` in the target repository. Add these entries to its `.gitignore`:

```gitignore
.gemini/
gemini-artifacts/
gha-creds-*.json
```

The workflow must exist on the default branch: GitHub only runs review-triggered workflows from there.

## Inputs

| Input                     | Default                                | Description                                              |
| ------------------------- | -------------------------------------- | -------------------------------------------------------- |
| `client-id`               |                                        | GitHub App Client ID. Required.                          |
| `pr-number`               | Triggering pull request                | Pull request number, for manual runs.                    |
| `review-id`               | Triggering review                      | Review ID, for manual runs.                              |
| `gemini-model`            | `gemini-3.8-flash`                     | Gemini model ID.                                         |
| `gemini-cli-version`      | `latest`                               | Gemini CLI version to install.                           |
| `commit-types`            | `feat fix docs refactor test ci chore` | Allowed types in `<type>: <summary>` commit messages.    |
| `fallback-commit-message` | `fix: address review comments`         | Used when Gemini's message doesn't match `commit-types`. |
| `instructions`            |                                        | Extra project instructions appended to the prompt.       |
| `max-turns`               | `40`                                   | Maximum Gemini session turns.                            |
| `timeout-minutes`         | `30`                                   | Job timeout.                                             |

Secrets: `APP_PRIVATE_KEY` and `GEMINI_API_KEY`, both required. Outputs: `pushed` (`true` or `false`) and `commit` (the pushed SHA).

## Manual Runs

GitHub re-runs a workflow with the workflow file of the original event, so re-running an old run never picks up later fixes. To apply the current workflow to an earlier review, use the caller's manual trigger:

```bash
gh workflow run resolve-review-comments.yml -f pr-number=<number> -f review-id=<id>
```

Find review IDs with `gh api repos/<owner>/<repo>/pulls/<number>/reviews --jq '.[].id'`.

## Models

The Gemini API bills separately from Google AI subscriptions: a Google AI Pro or Ultra plan doesn't cover API keys. On the free tier, Pro models have a quota of 0, so they need billing enabled on the key's Google Cloud project. Check the current [pricing](https://aistudio.google.com/docs/pricing) before choosing:

| Model                    | Free tier | Notes                                                  |
| ------------------------ | --------- | ------------------------------------------------------ |
| `gemini-3.8-flash`       | Yes       | Default. Strongest free model, but busy at peak times. |
| `gemini-2.5-flash`       | Yes       | Weaker, usually available.                             |
| `gemini-3.1-pro-preview` | No        | Stronger. Needs billing.                               |

## Security

- **Trusted reviewers only:** reviews by people without write access, by bots, and on fork pull requests are skipped.
- **Least privilege:** the workflow's `GITHUB_TOKEN` is read-only, and the App token never touches disk while Gemini runs.
- **Restricted tools:** Gemini gets file tools only. It can't run commands, fetch URLs, or push.
- **Protected paths:** the push step fails if a change touches `.github/` or the workflow's own folders.
- **Pinned actions:** third-party actions are pinned to commit SHAs and updated by Dependabot.

## Troubleshooting

| Symptom                                                | Cause                                                                 | Fix                                                  |
| ------------------------------------------------------ | --------------------------------------------------------------------- | ---------------------------------------------------- |
| `503 This model is currently experiencing high demand` | Free-tier traffic is served last at peak times.                       | Re-run later, pick another model, or enable billing. |
| `429 Quota exceeded … limit: 0`                        | The model isn't on the key's free tier.                               | Enable billing, or pick a free model.                |
| `Required status check … is expected` after a fix      | The push used `GITHUB_TOKEN`.                                         | Use a GitHub App token, as this workflow does.       |
| The push is rejected                                   | Branch protection blocks the App, or the branch moved during the run. | Allow the App to push, or re-run.                    |
| A re-run repeats an old failure                        | Re-runs reuse the original workflow file.                             | Start a [manual run](#manual-runs).                  |

## Limitations

- **One commit per review:** all comments of a review are fixed in a single commit.
- **No workflow edits:** comments asking to change files under `.github/` are left for a human.
- **Same-repository pull requests only:** fork pull requests get no secrets, so they're skipped.

## Versioning

Pin a major tag (`@v1`) to receive fixes, or a full commit SHA for strict reproducibility. Releases are listed in [CHANGELOG.md](CHANGELOG.md).
