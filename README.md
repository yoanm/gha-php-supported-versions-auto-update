# gha-php-supported-versions-auto-update

![GitHub Tag](https://img.shields.io/github/v/tag/yoanm/gha-php-supported-versions-auto-update?sort=semver&logo=githubactions&logoColor=white&logoSize=auto)

Lightweight composite github action updating PHP `max` and `next` supported versions.

New versions come from [yoanm/php-versions](https://github.com/yoanm/php-versions)

---

Action does the following:
 1. Checkout `yoanm/php-versions`
 2. Retrieve latest and nightly versions
 3. Fetch `max` and `next` PHP version for the given supported versions file
 4. Update currently configured `max` version in case there is a difference with PHP latest version
 5. Update currently configured `next` version in case there is a difference with PHP nightly version

> [!NOTE]
>
> `min` version is always left as is !
>
> Reason behind this is the fact that upgrading minimal PHP version would create a breaking change and therefore should rather be managed by the maintainer
>

## Usage

```yaml
  - name: Checkout current repository
    uses: actions/checkout@v5

  - name: Update PHP versions
    id: update-versions
    uses: yoanm/gha-php-supported-versions-auto-update@v1
    with:
      # Replace the following with the right path for you !
      path: .supported-versions.json
```

## Inputs
- `path`: **Required** Path to the supported versions file (local path)

## Outputs
- `updated`: Whether one of the versions have been updated or not (`0` / `1`)
- `max-updated`: Whether `max` version has been updated or not (`0` / `1`)
- `max-updated`: Whether `max` version has been updated or not (`0` / `1`)

## PHP auto-update workflow example
The following workflow will create a dedicated branch with the updated versions as well as an issue with a link to create the PR

> [!NOTE]
>
> Automatically creating a PR would be nicer BUT it requires the repository to authorize a github workflow to create a PR.
>
> In case this option is enabled on your repository, feel free to directly create the PR instead (likely through `gh pr create`) !
>

```yaml
name: 'Supported versions auto-update'

on:
  workflow_dispatch: # Allows to run the workflow manually from the Actions tab
  schedule:
    - cron: '0 0 1 * *' # Every month

permissions:
  contents: write # Required in order to commit updated files
  issues: write # Required in order to create the related GH issue

concurrency:
  group: "${{ github.workflow }}-${{ github.head_ref || github.ref }}"
  cancel-in-progress: true

env:
  AUTHOR_NAME: AUTHOR_NAME
  AUTHOR_EMAIL: AUTHOR_EMAIL
  V_FILE: .supported-versions.json
  UPDATE_BRANCH: auto/feature/increase-supported-version
  ISSUE_TITLE: "[AUTO] update supported versions"
  PR_TITLE: "Update supported versions"
  ASSIGNEE_HANDLES: REPLACE_BY_YOUR_GITHUB_HANDLE

  SERVER_URL: ${{ github.server_url }}
  REPO_NAME: ${{ github.repository }}
  RUN_ID: ${{ github.run_id }}

jobs:
  php:
    name: PHP
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
    steps:

      - name: Get current date/time
        id: datetime
        run: echo "value=$(date +'%Y-%m-%d')" >> $GITHUB_OUTPUT

      - name: "Checkout current repository"
        uses: actions/checkout@v5
        with:
          ref: ${{ github.event.repository.default_branch }}
          fetch-depth: 0

      - name: Create/Switch to update branch
        env:
          BRANCH: ${{ env.UPDATE_BRANCH }}
          DEFAULT_BRANCH: ${{ github.event.repository.default_branch }}
        run: |
          git checkout "${BRANCH}" || git checkout --track origin/${DEFAULT_BRANCH} -b "${BRANCH}"

      - name: Update versions
        id: update-versions
        uses: yoanm/gha-php-supported-versions-auto-update@v1
        with:
          path: ${{ env.V_FILE }}

      - name: "Add / commit / push"
        id: commit-and-push
        if: ${{ steps.update-versions.outputs.updated == '1' }}
        env:
          DATE: ${{ steps.datetime.outputs.value }}
        run: |
          git config user.name "${AUTHOR_NAME}"
          git config user.email "${AUTHOR_EMAIL}"
          git add $V_FILE
          if [ $(git diff --cached --name-only | wc -l) -gt 0 ]; then
            echo "Pushing updated files:"
            git diff --cached --color $V_FILE
            git commit -m "Update PHP supported versions (${DATE})" -m "See ${SERVER_URL}/${REPO_NAME}/actions/runs/${RUN_ID}";
            git push --set-upstream origin "${UPDATE_BRANCH}";
            ISSUE_NEEDED=1
          else
            echo "Nothing to commit !";
            ISSUE_NEEDED=0
          fi;
          echo "issue-needed=${ISSUE_NEEDED}" >> $GITHUB_OUTPUT

      - name: "Create issue"
        if: ${{ steps.commit-and-push.outputs.issue-needed == 1 }}
        env:
          BODY_FILE: ${{ runner.temp }}/body-file.txt
          GH_TOKEN: ${{ github.token }}
        run: |
          ISSUE_BODY_HEADER="Supported versions file needs to be updated !"
          ISSUE_URL=$(gh issue list --search "state:open author:@me \"${ISSUE_TITLE}\"" --json url --jq '.[0].url')
          if [[ -z "${ISSUE_URL}" ]]; then
            echo "Creating new issue"
            gh issue create --title "${ISSUE_TITLE}" --body "${ISSUE_BODY_HEADER}" --assignee "${ASSIGNEE_HANDLES}" >> issue_url.txt
            ISSUE_URL=$(cat issue_url.txt)
            if [[ -z "${ISSUE_URL}" ]]; then
              echo "::error::Unable to retrieve the issue URL !"
              exit 1
            fi
            ISSUE_ID=$(gh issue view "${ISSUE_URL}" --json number --jq '.number')
            if [[ -z "${ISSUE_ID}" ]]; then
              echo "::error::Unable to retrieve the issue ID !"
              exit 1
            fi

            echo "Update issue with proper link to create the PR"
            PR_ENCODED_TITLE=$(jq -rn --arg x "${PR_TITLE}" '$x|@uri')
            PR_BODY_ENCODED=$(jq -rn --arg x "Fix #${ISSUE_ID}" '$x|@uri')
            echo "${ISSUE_BODY_HEADER}" > ${BODY_FILE}
            echo "" >> ${BODY_FILE}
            echo "Create PR [there](https://github.com/${REPO_NAME}/compare/${UPDATE_BRANCH}?quick_pull=1&assignees=${ASSIGNEE_HANDLES}&title=${PR_ENCODED_TITLE}&body=${PR_BODY_ENCODED})" >> ${BODY_FILE}
            gh issue edit "${ISSUE_URL}" --body-file "${BODY_FILE}"
          else
            echo "Issue already exists: ${ISSUE_URL}"
          fi
```
