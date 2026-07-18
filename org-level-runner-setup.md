# Self-Hosted Runner Setup for `example-github-org`

This guide covers how to:

1. Add a self-hosted runner to the `example-github-org` organization
2. Control access to the runner using a runner group called `example-runner-group`
3. Automatically assign repositories to the runner group by repo name, so no admin action is needed when new repos are created

Based on the official GitHub documentation:

- [Adding self-hosted runners](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/add-runners)
- [Managing access to self-hosted runners using groups](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/manage-access)
- [REST API endpoints for self-hosted runner groups](https://docs.github.com/en/rest/actions/self-hosted-runner-groups)

---

## Prerequisites

- You must be an **organization owner** of `example-github-org`.
- You must have access to the machine that will act as the self-hosted runner.
- Additional organization-level runner groups require the **GitHub Team plan** (or an enterprise-owned organization). Every org has a single Default group regardless of plan.

> **Security warning:** GitHub recommends only using self-hosted runners with **private repositories**. Forks of a public repository can potentially run dangerous code on your runner machine by opening a pull request that executes code in a workflow.

---

## Part 1: Create the Runner Group `example-runner-group`

Create the group first so the runner can be registered directly into it (see Part 2).

1. Navigate to the main page of the `example-github-org` organization.
2. Click **Settings**.
3. In the left sidebar, click **Actions**, then click **Runner groups**.
4. In the "Runner groups" section, click **New runner group**.
5. Enter the name: `example-runner-group`.
6. Assign a **policy for repository access**:
   - **All repositories**, or
   - **Selected repositories** (manual checkbox list)
   - Note: by default, only private repositories can access runners in a runner group. This can be overridden in the group settings.
7. Click **Create group**.

---

## Part 2: Add the Self-Hosted Runner to the Organization

1. Navigate to the main page of the `example-github-org` organization.
2. Click **Settings**.
3. In the left sidebar, click **Actions**, then click **Runners**.
4. Click **New runner**, then click **New self-hosted runner**.
5. Select the operating system image and architecture of your runner machine.
6. Follow the on-screen instructions on your runner machine. They walk you through:
   - Downloading and extracting the runner application
   - Running the `config` script to register the runner (uses the destination URL and an auto-generated token that expires after **one hour**)
   - Running the runner application to connect the machine to GitHub Actions

### Register the runner directly into the group (recommended)

Instead of registering into the Default group and moving it later, pass the `--runnergroup` parameter to the config script:

```shell
./config.sh --url https://github.com/example-github-org --token <TOKEN> --runnergroup example-runner-group
```

The command fails if the group does not exist yet:

```text
Could not find any self-hosted runner group named "example-runner-group".
```

### Verify the runner was added

The runner and its status appear under **Settings > Actions > Runners**. When the runner application is connected and ready, the machine's terminal shows:

```shell
√ Connected to GitHub

2019-10-24 05:45:56Z: Listening for Jobs
```

### If the runner is already in the Default group, move it

1. Go to **Settings > Actions > Runners**.
2. Click the runner you want to configure.
3. Select the **Runner group** drop-down.
4. Under "Move runner to group", choose `example-runner-group`.

---

## Part 3: Assign Repositories to the Runner Group by Repo Name

**Important:** the GitHub UI has no "match repos by name pattern" option. The access policy is only **All repositories** or **Selected repositories** with manual checkboxes. There are two ways to avoid contacting the admin for every new repo:

### Option A (simplest): Set the policy to "All repositories"

1. Go to **Settings > Actions > Runner groups**.
2. Click `example-runner-group`.
3. Under "Repository access", select **All repositories**.
4. Click **Save group**.

Every new repository in the organization automatically has access the moment it is created. Zero maintenance. Use this unless certain repos must be fenced off from this runner.

### Option B (name-based): Automated sync via the REST API

A scheduled workflow finds repos matching a naming convention and adds them to the group using the REST API.

Save this as `.github/workflows/sync-runner-group.yml` in an admin/ops repository in the org:

```yaml
name: Sync runner group repos
on:
  schedule:
    - cron: '0 * * * *'   # hourly
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Add matching repos to example-runner-group
        env:
          GH_TOKEN: ${{ secrets.ORG_ADMIN_TOKEN }}
        run: |
          ORG="example-github-org"
          PREFIX="app-"   # your repo naming convention

          # Find the runner group ID by name
          GROUP_ID=$(gh api "/orgs/$ORG/actions/runner-groups" \
            --jq '.runner_groups[] | select(.name=="example-runner-group") | .id')

          # Add every repo whose name matches the prefix
          gh api --paginate "/orgs/$ORG/repos" --jq '.[] | [.id, .name] | @tsv' |
          while IFS=$'\t' read -r REPO_ID REPO_NAME; do
            if [[ "$REPO_NAME" == "$PREFIX"* ]]; then
              gh api -X PUT "/orgs/$ORG/actions/runner-groups/$GROUP_ID/repositories/$REPO_ID"
              echo "Added $REPO_NAME"
            fi
          done
```

#### Notes on Option B

- The `PUT /orgs/{org}/actions/runner-groups/{group_id}/repositories/{repo_id}` endpoint is additive and idempotent. Re-running on repos already in the group is harmless.
- The default `GITHUB_TOKEN` **cannot** be used. The token needs org-admin level permission:
  - a classic PAT with the `admin:org` scope, or
  - a fine-grained PAT / GitHub App with write access to **organization self-hosted runners**.
- The admin sets up the token **once** and stores it as an org or repo secret (`ORG_ADMIN_TOKEN`). After that, no human intervention is needed when new repos appear.
- For instant (event-driven) assignment instead of hourly polling, use an organization webhook on the `repository` created event pointed at a small handler or GitHub App that makes the same API call. The cron approach is usually good enough.

---

## Using the Runner in a Workflow

Once configured, target the runner in workflow files with the `runs-on` key using its labels, for example:

```yaml
jobs:
  build:
    runs-on: self-hosted
```

See [Using self-hosted runners in a workflow](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/use-in-a-workflow) for label details.
