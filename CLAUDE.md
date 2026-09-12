# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Authorship

Never add AI attribution anywhere. No `Co-Authored-By: Claude`, no "Generated
with Claude Code", no "written by Claude", in commit messages, PR titles or
bodies, file headers, or code comments. Commits and pull requests are authored by
the repository owner alone.

## Comments

The default is no comment. Add one only when a reader who knows CloudFormation
would still ask "why".

- **Explain why, never what.** If a comment restates the line below it, delete it.
- **Never interrupt a property block.** No comments between YAML keys, inside a
  `Properties:` map, or between the entries of a list. If a choice needs
  explaining, put one short comment above the whole resource.
- **One comment per resource, at most, and two lines at most.** Anything longer
  belongs in the README or in `docs/`, not in the template.
- **No section-divider banners.** No `# ----- Network -----` blocks.
- **No comments on parameter declarations.** Parameters already have a
  `Description` field; use it and leave the declaration clean.
- **No comments restating a policy.** An IAM statement with a `Sid` is already
  labelled.

Bad:

```yaml
  Service:
    Type: AWS::ECS::Service
    Properties:
      DeploymentController:
        # CodeDeploy owns rollout from here on, so stack updates do not fight it
        Type: CODE_DEPLOY
      NetworkConfiguration:
        AwsvpcConfiguration:
          # no public IP - everything the task needs is behind a VPC endpoint
          AssignPublicIp: DISABLED
```

Good:

```yaml
  # CodeDeploy owns task definition rollout, so pipeline deploys and stack
  # updates cannot fight over this service.
  Service:
    Type: AWS::ECS::Service
    Properties:
      DeploymentController:
        Type: CODE_DEPLOY
      NetworkConfiguration:
        AwsvpcConfiguration:
          AssignPublicIp: DISABLED
```

### The two exceptions

These are graded, so they stay — one line each, above the resource or on the line
itself:

- Every `Resource: "*"` in an IAM policy states why the API cannot be scoped.
- Every `DependsOn` states why implicit ordering is not enough.

## YAML style

Block style everywhere, including intrinsics. No inline `{}` or `[]` flow style.
Embedded JSON uses a `|` block literal. Quote anything YAML would coerce:
`"256"`, `"false"`, `"2010-09-09"`.

## Scope

This repository holds one stack and no application or nested-stack templates.
Resources that are recreated freely belong in `todo-app-infra`; only things that
must pre-exist it or survive its teardown belong here.

Changing an `Outputs` export name breaks `todo-app-infra`, which imports it.
Check `aws cloudformation list-imports --export-name <name>` first.

## Before proposing a change as done

`cfn-lint 00-bootstrap.yaml` must pass with no errors and no warnings.
