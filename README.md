# todo-app-bootstrap

Foundation stack for the To-Do lab. One CloudFormation template holding the
resources that must exist **before** the application infrastructure and must
**survive** its teardown.

Deployed once. Changed rarely. Deliberately separate from
[`todo-app-infra`](https://github.com/1MuhireDavid/todo-app-infra), which is
destroyed and rebuilt freely, and from
[`todo-app`](https://github.com/1MuhireDavid/todo-app), the application itself.

- **Region:** us-east-1
- **Account:** 047719661196
- **Stack name:** `todo-app-bootstrap`

## What it owns

| Resource | Deletion policy | Why it lives here |
|---|---|---|
| `todo-app-cfn-templates-047719661196-us-east-1` | **Retain** | holds the packaged nested templates; deleting it would destroy what the application stack needs to redeploy |
| `todo-app` (ECR) | **Retain** | the tasks run with no internet route, so losing the image means the application stack can never start again |
| `todo-app-pipeline-artifacts-047719661196-us-east-1` | Delete | pipeline artifacts and ALB access logs, all regenerable |
| `todo-app-gha-infra-packaging` | Delete | OIDC role for the infra repo's packaging workflow |
| `todo-app-gha-app-build` | Delete | OIDC role for the app repo's build workflow |

The two `Retain` resources are the whole reason this repository exists. Deleting
the application stack must never destroy the artifacts needed to recreate it, and
a repository boundary makes that harder to do by accident than a directory
boundary does.

### Why ECR is here and not in the app repo

It holds application artifacts, so there is a reasonable argument for the app
repository. Three properties win the other way: it must exist before the
application stack is created, it must survive that stack's teardown, and it is
shared — the app repo pushes to it, the infra repo's pipeline reads from it.
Same argument as the templates bucket.

### Why the artifact bucket is here

A compromise, and worth knowing. The ALB in `06-alb-ecs.yaml` writes access logs
to it, but the CI/CD stack that would naturally own it has to be created *after*
the ALB, because CodeDeploy needs the service, target groups and listeners.
Owning it here breaks that cycle, and it also lets the app build role — defined
here — be scoped to a real bucket ARN instead of one guessed with `Fn::Sub`.

The cost is that a near-permanent foundation stack owns a bucket the pipeline
writes to on every build. Clean lines lost to a dependency cycle.

## Deploy

Create the CloudFormation execution role first — see [`iam/`](iam/) — then pass
it with `--role-arn`, so the stack is bounded by its own policy rather than by
whatever your console user happens to be allowed to do.

```bash
aws cloudformation create-stack \
  --stack-name todo-app-bootstrap \
  --template-body file://00-bootstrap.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::047719661196:role/todo-app-cfn-exec-bootstrap \
  --region us-east-1 \
  --parameters \
      ParameterKey=ProjectName,ParameterValue=todo-app \
      ParameterKey=GitHubOwner,ParameterValue=1MuhireDavid \
      ParameterKey=InfraRepoName,ParameterValue=todo-app-infra \
      ParameterKey=AppRepoName,ParameterValue=todo-app \
      ParameterKey=CreateOIDCProvider,ParameterValue=false

aws cloudformation wait stack-create-complete \
  --stack-name todo-app-bootstrap --region us-east-1

aws cloudformation describe-stacks \
  --stack-name todo-app-bootstrap --region us-east-1 \
  --query "Stacks[0].Outputs" --output table
```

`CreateOIDCProvider=false` because account 047719661196 already federates GitHub
Actions. An account holds exactly one provider per URL; a second fails with
`EntityAlreadyExists`. Pass `true` in a fresh account.

Optionally manage later changes through CloudFormation Git sync instead, pointed
at `deployment-file.yaml` with stack name `todo-app-bootstrap`. Keep it a
separate sync configuration from the application stack.

**This stack must exist before anything else.** Neither other repository can
create it: neither OIDC role holds a CloudFormation write permission, by design.

## The contract with the other repos

This stack publishes `Export`s; `todo-app-infra`'s root stack consumes them with
`Fn::ImportValue`. Nothing is copy-pasted between repositories.

| Export | Consumed by |
|---|---|
| `todo-app-bootstrap-TemplatesBucketName` | root stack, to build every nested `TemplateURL` |
| `todo-app-bootstrap-ArtifactBucketName` / `-ArtifactBucketArn` | ALB access logs, CodePipeline artifact store |
| `todo-app-bootstrap-EcrRepositoryUri` / `-Name` / `-Arn` | task definition image, pipeline ECR source, execution role |
| `todo-app-bootstrap-InfraPackagingRoleArn` | referenced in `package-templates.yml` |
| `todo-app-bootstrap-AppBuildRoleArn` | referenced in `build-and-push.yml` |

Exports are scoped to an account and region, not to a repository — CloudFormation
has no idea which repo a template came from. The split changes who reviews what,
not how the stacks wire together.

**Two consequences of being an exporter.** While the application stack imports
these values you cannot change an exported value, and you cannot delete this
stack at all. Both failures surface here while the cause lives in the other
repository, so check what imports an export before changing it:

```bash
aws cloudformation list-imports \
  --export-name todo-app-bootstrap-EcrRepositoryUri --region us-east-1
```

## Renaming things

`ProjectName` and the two repo-name parameters are independent, and only one of
them is about GitHub.

- `ProjectName` names the ECR repository, both buckets and both roles. Changing
  it replaces resources and cascades into every stack.
- `InfraRepoName` / `AppRepoName` appear only in the OIDC trust policies. If you
  rename a repository on GitHub, the `repository` and `job_workflow_ref` claims
  change immediately — GitHub's redirect covers git remotes, not OIDC — so update
  the matching parameter here and redeploy, or that repo's workflow starts
  failing at `sts:AssumeRoleWithWebIdentity`.

## Teardown

Only after the application stack is fully deleted, and only if you are finished
with the project. The two `Retain` resources survive the stack delete and have to
be removed by hand:

```bash
aws s3 rm s3://todo-app-cfn-templates-047719661196-us-east-1 --recursive
aws s3 rb s3://todo-app-cfn-templates-047719661196-us-east-1 --force
aws ecr delete-repository --repository-name todo-app --force --region us-east-1

aws cloudformation delete-stack --stack-name todo-app-bootstrap --region us-east-1
```

Empty the artifact bucket first if the application stack's teardown did not —
a versioned bucket with objects in it hangs the delete. The full ordered
procedure is in the `todo-app-infra` README.
