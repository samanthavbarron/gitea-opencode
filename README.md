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
Unfortunately forgejo does not provide a token to actions capable of creating a new PR, only capable of actions like responding to an issue with a new comment. This means in order to seamlessly creeate a PR for a changes made in a new branch, this action must be provided with a Personal Access Token (PAT) capable of opening a new PR within the repostitory (requires SOLELY the `repository read+write` permission on the repoisitory in question). This is optional, but highly recommended. The ephemeral token provided by forgejo automatically is used for ALL other actions, whether or not this `PR_CREATION_PAT` is provided.

Although Forgejo does not yet fully support "bot" accounts (service accounts), it's recommended to create a new user (e.g. "opencode-bot") and produce the `PR_CREATION_PAT` under this user, giving it the `repository read+write` permission to all repositories that it's been added as a collaborator to. This separate user can then be added as a collaborator to any projects that use this action, allowing them to create the PR as required.

## Protections against random code changes
This prepares an environment for the agent by pre-checking out the relevant codebase, checking out the appropriate branch if it's a PR related event, or creating a new local branch for commits made by opencode if the event is yet unrelated to a specific PR. While the opencode runner is given effectively free reign by design while executing, it is NOT provided access keys or an authenticated git environment to allow it to push back to other branches (or to `origin/main`). All work unrelated to an existing PR is done on a new branch created for this purpose, and work related to an existing PR is done on the branch associated with the PR in question. This is the only branch pushed back to the repository in question.

## Attribution of Code Changes
The Author of any commits made is set to whichever user triggered the event in the first place, such as the person who made a comment on the issue which triggered PR creation to occur. All commits performed are attributed with the git trailer `Co-authored-by: opencode <opencode@noreply.localhost>`, and set with the committer name as `opencode` and committer email as `opencode@noreply.localhost`.

# Usage
Usage should look something like the below, just create it as a file in `.forgejo/workflows/opencode.yaml` within a repository. The `PR_CREATION_PAT` secret is optional, but highly recommended. Without it, the action only creates branches vs PRs on any changes.

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
      (
        forgejo.event_name == 'issues' && (
          contains(forgejo.event.issue.body, ' /oc') ||
          startsWith(forgejo.event.issue.body, '/oc') ||
          contains(forgejo.event.issue.body, ' /opencode') ||
          startsWith(forgejo.event.issue.body, '/opencode')
        )
      ) || (
        contains(forgejo.event.comment.body, ' /oc') ||
        startsWith(forgejo.event.comment.body, '/oc') ||
        contains(forgejo.event.comment.body, ' /opencode') ||
        startsWith(forgejo.event.comment.body, '/opencode') ||
        contains(forgejo.event.review.content, ' /oc') ||
        startsWith(forgejo.event.review.content, '/oc') ||
        contains(forgejo.event.review.content, ' /opencode') ||
        startsWith(forgejo.event.review.content, '/opencode')
      )

    # The actions/checkout@v4 action may have requirements, but this
    # does not require node or other languages be present. It only
    # requires a semi-recent git, curl, bash and ability to run the
    # latest opencode and jq versions (both of which it retrieves)
    runs-on: ubuntu-latest

    steps:
      - name: Run opencode
        uses: https://codeberg.org/dragonfyre13/forgejo-opencode@latest
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

Another example, if you wanted to simply have opencode respond to any/all issues opened in a given repository, without any kind of trigger word(s):

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
        uses: https://codeberg.org/dragonfyre13/forgejo-opencode@latest
        with:
          model: opencode-go/kimi-k2.6
          pr-creation-pat: ${{ secrets.PR_CREATION_PAT }}
          agent: build
          variant: ""
          mentions: ""
        env:
          OPENCODE_API_KEY: ${{ secrets.OPENCODE_API_KEY }}
```
