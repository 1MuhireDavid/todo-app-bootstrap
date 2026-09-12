# CloudFormation execution role for the bootstrap stack

Create this **before** the stack. Without it, CloudFormation falls back to your
own console credentials, which works but means the stack's blast radius is
whatever your user can do — and you cannot answer "what can this stack change?"
without auditing your own permissions.

Role name used throughout: **`todo-app-cfn-exec-bootstrap`**

## Console

1. IAM → Roles → **Create role** → **Custom trust policy**
2. Paste [`trust-policy.json`](trust-policy.json)
3. Skip the permissions page (**Next** without selecting anything)
4. Name it `todo-app-cfn-exec-bootstrap`, create
5. Open the role → Permissions → **Add permissions** → **Create inline policy**
   → JSON → paste [`permissions-policy.json`](permissions-policy.json) → name it
   `todo-app-cfn-exec-bootstrap-policy`

Then pass it when you create the stack — the **IAM role** field under Permissions
on the stack options page, or `--role-arn` on the CLI:

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
```

## CLI, if you would rather not click

```bash
aws iam create-role \
  --role-name todo-app-cfn-exec-bootstrap \
  --assume-role-policy-document file://iam/trust-policy.json \
  --max-session-duration 3600 \
  --tags Key=Project,Value=todo-app

aws iam put-role-policy \
  --role-name todo-app-cfn-exec-bootstrap \
  --policy-name todo-app-cfn-exec-bootstrap-policy \
  --policy-document file://iam/permissions-policy.json
```

## What the trust policy says

Only CloudFormation can assume it, and only on behalf of a stack named
`todo-app-bootstrap` in this account. The `aws:SourceAccount` and
`aws:SourceArn` conditions are the confused-deputy guard: without them, any
CloudFormation stack in any account that somehow referenced this role ARN could
use these permissions.

## What the permissions policy allows

| Statement | Scope |
|---|---|
| `TemplatesAndArtifactBuckets` | the two bucket ARNs by name, nothing else |
| `EcrRepository` | `repository/todo-app` only |
| `GitHubOidcRoles` | `role/todo-app-gha-*` — the two OIDC roles, and no other role in the account |
| `OidcProviderOnlyWhenCreateOIDCProviderIsTrue` | the single GitHub provider ARN |

Everything is resource-scoped; there is no `Resource: "*"` in this policy at all.
The stack creates five resources and this role can touch exactly those five.

The last statement is unused while `CreateOIDCProvider` is `"false"`, which it is
in this account — the provider already exists. Delete the statement if you prefer
a policy with no dead weight; leave it if you might deploy into a fresh account.

## When it fails

An `AccessDenied` during stack creation names the action and the resource. Add
that one action to the matching statement rather than widening the resource — a
denial here is the policy doing its job, and the fix is almost always a sibling
action of one already listed (`GetBucketTagging` next to `PutBucketTagging`, that
kind of thing).
