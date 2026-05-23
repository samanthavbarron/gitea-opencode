Forked from https://codeberg.org/dragonfyre13/forgejo-opencode

# Overview

This action allows for integration with opencode, similar to [github](https://opencode.ai/docs/github/) actions integration, but without the benefit of custom handling built into opencode itself. With respect to how it functions underneath, it's actually a bit closer to the current [opencode gitlab](https://opencode.ai/docs/gitlab/) integration with respect to using a somewhat raw `opencode run` command wrapped entirely in scripting for environment preparation and teardown.

The action is intended to be triggered based on events like:
* New issues being opened
* Comments on existing issues or PRs
* Responding to reviews submitted on PRs (including support for processing review comments made to individual lines of a PR).

The type of responses supported include one or more of the following:
* Creating a new PR (in response to issues or issue comments, if a PR_CREATION_PAT is available)
* Creating a new branch while adding a comment to quickly create an associated PR (in response to issues or issue comments, if no PR_CREATION_PAT is available)
* Making additional changes to an existing branch (associated with events on a PR with that branch)
* Adding a comment response to the issue or PR in question

## Requiring the `PR_CREATION_PAT` secret
Unfortunately Gitea does not provide a token to actions capable of creating a new PR, only capable of actions like responding to an issue with a new comment. This means in order to seamlessly creeate a PR for a changes made in a new branch, this action must be provided with a Personal Access Token (PAT) capable of opening a new PR within the repostitory (requires SOLELY the `repository read+write` permission on the repoisitory in question). This is optional, but highly recommended. The ephemeral token provided by Gitea automatically is used for ALL other actions, whether or not this `PR_CREATION_PAT` is provided.

Although Gitea does not yet fully support "bot" accounts (service accounts), it's recommended to create a new user (e.g. "opencode-bot") and produce the `PR_CREATION_PAT` under this user, giving it the `repository read+write` permission to all repositories that it's been added as a collaborator to. This separate user can then be added as a collaborator to any projects that use this action, allowing them to create the PR as required.

## Protections against random code changes
This prepares an environment for the agent by pre-checking out the relevant codebase, checking out the appropriate branch if it's a PR related event, or creating a new local branch for commits made by opencode if the event is yet unrelated to a specific PR. While the opencode runner is given effectively free reign by design while executing, it is NOT provided access keys or an authenticated git environment to allow it to push back to other branches (or to `origin/main`). All work unrelated to an existing PR is done on a new branch created for this purpose, and work related to an existing PR is done on the branch associated with the PR in question. This is the only branch pushed back to the repository in question.

## Attribution of Code Changes
The Author of any commits made is set to whichever user triggered the event in the first place, such as the person who made a comment on the issue which triggered PR creation to occur. All commits performed are attributed with the git trailer `Co-authored-by: opencode <opencode@noreply.localhost>`, and set with the committer name as `opencode` and committer email as `opencode@noreply.localhost`.

# Usage
Usage should look something like the below, just create it as a file in `.gitea/workflows/opencode.yaml` within a repository. The `PR_CREATION_PAT` secret is optional, but highly recommended. Without it, the action only creates branches vs PRs on any changes.

```yaml
name: opencode

on:
  issues:
    types: [opened]
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  pull_request_review:
    types: [submitted]

jobs:
  opencode:
    # This looks complicated, but basically is filtering events down to
    # only those that contain either "/oc" or "/opencode" in the body.
    # You can write filters however you want to, without a restriction
    # on specific wording being included. It is however recommended to
    # keep the "mentions" input value up to date with any references to
    # the agent, but this only affects the prompt details sent as input
    # to the opencode process directly.
    if: |
      (github.event_name == 'issues' && (
        contains(github.event.issue.body, ' /oc') ||
        startsWith(github.event.issue.body, '/oc') ||
        contains(github.event.issue.body, ' /opencode') ||
        startsWith(github.event.issue.body, '/opencode')
      )) ||
      contains(github.event.comment.body, ' /oc') ||
      startsWith(github.event.comment.body, '/oc') ||
      contains(github.event.comment.body, ' /opencode') ||
      startsWith(github.event.comment.body, '/opencode') ||
      contains(github.event.review.content, ' /oc') ||
      startsWith(github.event.review.content, '/oc') ||
      contains(github.event.review.content, ' /opencode') ||
      startsWith(github.event.review.content, '/opencode')

    # This action runner only requires bash 4.4+, git, curl, bash and the
    # ability to run latest opencode and jq (both of which it retrieves)
    runs-on: ubuntu-latest

    steps:
      - name: Run opencode
        uses: samanthavbarron/gitea-opencode@main
        with:
          # This is passed as --model to opencode when called
          model: opencode-go/kimi-k2.6

          # Used solely for creating PRs, all other operations use the
          # provided emphemeral action token
          pr-creation-pat: ${{ secrets.PR_CREATION_PAT }}

          # This is passed as --agent to opencode when called
          agent: build

          # This is passed as --variant to opencode when called
          variant: ""

          # This is used to inform opencode of how it may have been
          # mentioned within the requesting event. Setting to blank
          # removes any prompt references to it.
          mentions: "/oc,/opencode"
        env:
          # Env vars trickle down to the opencode process, in this
          # case authenticating the opencode-go provider to allow
          # targeting usage of the chosen model above.
          OPENCODE_API_KEY: ${{ secrets.OPENCODE_API_KEY }}
```

Another example, if you wanted to simply have opencode respond to any/all issues opened in a given repository, without any kind of trigger word(s). This one uses a custom prompt as well.

```yaml
name: opencode

on:
  issues:
    types: [opened]

jobs:
  opencode:
    runs-on: ubuntu-latest
    steps:
      - name: Run opencode
        uses: samanthavbarron/gitea-opencode@main
        with:
          model: opencode-go/kimi-k2.6
          pr-creation-pat: ${{ secrets.PR_CREATION_PAT }}
          agent: build
          prompt: |
            You are an AI assistant that helps develop and improve code repositories. The git repository in question is in the current directory.

            ## Task
            Evaluate and either resolve or respond with a comment to the issue described below, using the available tools.
            Evaluate the Issue and any attached information to fully understand the task.
            Be thorough in your analysis and provide clear explanations.

            ## Context
            Issue Title:
            $ISSUE_TITLE

            Issue Body:
            $TRIGGERING_REQUEST_BODY

            ## Guidelines
            - Explore the codebase before making changes
            - Follow existing code/style conventions
            - To reply with a comment: output text response (markdown format is allowed)
            - To make code changes: commit with clear messages
            - To create a new Pull Request, commit changes locally. DO NOT PUSH TO THE REMOTE REPO. Follow Pull Request Creation Guidelines.
            - If no response comment is neccesary to complete the task, simply commit any required changes and respond with the exact text \`Changes complete\`.
            - **ALWAYS COMMIT OR RESET ANY CHANGES**: Do not leave the repository in a "dirty" state (can check with \`git status --porcelain\`).
            - **First Commit Creates the PR**: The first commit summary (first line) and body text (subsequent lines) made will be used to define the title and description of the new pull request, respectively.
            - **Description is Markdown**: The Pull Request description (subsequent lines in the commit) is interpreted in markdown format.
            - **DO NOT follow commit message standards**: Since the first commit message is used to create the Pull Request itself, you ABSOLUTELY SHOULD NOT follow existing or standard commit message guidelines. The PR commit message will be squashed when merged by the user in any case.
            - **Detailed Pull Request Descriptions**: Details regarding justification, investigation, changes, decisions made, and other information about changes should be included as part of the Pull Request description, rather than on a seperate comment.
            - **Auto-Closing Original Issues**: Auto-closing the original issue on merge of a PR can be triggerd by including \`fixes #123\` or \`closes #345\` somewhere within the PR description.
            - **An Issue Link is Already Created**: A separate comment will already be added to the issue, notifying users of the PR creation and linking to it directly. When a Pull Request is created, only respond with a comment if the response is unrelated to the changes made.
            - **DO NOT switch branches**: The current branch has already been created to hold your changes in case a PR is needed.
            - **DO NOT Submit For A PR**: A Pull Request will automatically be created after you have finished if there are any local commits, do not attempt to create one yourself.
        env:
          OPENCODE_API_KEY: ${{ secrets.OPENCODE_API_KEY }}
```

A final simple example specifying a custom opencode configuration file, allowing the user to use a self hosted model via llama.cpp (llama-server).

```yaml
name: opencode

on:
  issues:
    types: [opened]
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  pull_request_review:
    types: [submitted]

jobs:
  opencode:
    if: |
      (github.event_name == 'issues' && contains(github.event.issue.body, '/opencode')) ||
      contains(github.event.comment.body, '/opencode') ||
      contains(github.event.review.content, '/opencode')
    runs-on: ubuntu-latest
    steps:
      - name: Run opencode
        uses: samanthavbarron/gitea-opencode@main
        with:
          pr-creation-pat: ${{ secrets.PR_CREATION_PAT }}
          config: |
            {
              "$schema": "https://opencode.ai/config.json",
              "provider": {
                "local-llm": {
                  "npm": "@ai-sdk/openai-compatible",
                  "name": "llama-server (local)",
                  "options": {"baseURL": "http://llm:8080/v1"},
                  "models": {
                    "qwen3.6-27B": {
                      "name": "Qwen3.6: 27B (local)",
                      "limit": {"context": 262144, "output": 65536}
                    }
                  }
                }
              }
            }
          model: local-llm/qwen3.6-27B
          agent: build
          mentions: /opencode
```
