---
title: "github actions with oidc to aws"
date: 2026-01-27T10:00:00-08:00
draft: false
summary: "replacing aws access keys with oidc tokens for automated deployments."
---

i automated the deployment of this site using github actions. the interesting part was setting up oidc authentication instead of using long-lived aws credentials.

## the problem with access keys

the straightforward approach is to create aws access keys and store them as github secrets. this works but has downsides.

access keys are long-lived credentials. they sit in your secrets until you rotate them. if they leak somehow, they're valid until you notice and delete them. could be days, could be months.

you also need to manage rotation. aws recommends rotating credentials regularly. more work.

## how oidc works

oidc lets github actions assume an aws iam role without storing any credentials.

when a workflow runs, github generates a jwt token. this token includes claims about the workflow. which repo it's from, what branch triggered it, the specific workflow run id.

the workflow exchanges this token with aws sts. aws validates the token by checking it was signed by github's private key. aws fetches github's public keys from `https://token.actions.githubusercontent.com/.well-known/jwks` to verify the signature.

if the token is valid and the claims match your trust policy, aws issues temporary credentials. these last only for the duration of the job. usually around 15 minutes.

## the trust chain

you configure aws to trust `token.actions.githubusercontent.com` as an oidc provider. when setting this up, you specify github's endpoint url and a thumbprint of their certificate.

in the iam role's trust policy, you define conditions. things like which repository can assume the role, which branches, or even specific workflows.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:yourusername/yourrepo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

this trust policy says only workflows from your specific repo on the main branch can assume the role.

## setting it up

first, create the oidc provider in aws. you need github's thumbprint, which you can get by inspecting their certificate.

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 7560D6F40FA55195F740EE2B1B7C0B4836CBE103
```

then create an iam role with the trust policy above. attach permissions for what the workflow needs to do. for this site, it's s3 operations on the bucket.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::judelanning.com",
        "arn:aws:s3:::judelanning.com/*"
      ]
    }
  ]
}
```

## the workflow

the github actions workflow needs two changes from using access keys.

add permissions to request oidc tokens.

```yaml
permissions:
  id-token: write
  contents: read
```

use `role-to-assume` instead of access key secrets.

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/YourRoleName
    aws-region: us-east-1
```

the action handles the token exchange. it requests a jwt from github, sends it to aws sts, and receives temporary credentials. these get set as environment variables for the rest of the job.

## why this is better

no long-lived credentials stored anywhere. the jwt expires in about 5 minutes. the temporary aws credentials expire when the job finishes.

the action masks credentials in logs automatically. even someone watching workflow runs in real-time wouldn't see usable tokens. they never appear in readable form.

the trust policy limits which repos and branches can assume the role. you can get very granular. only production deployments from main, or only specific workflow files.

no rotation needed. each job gets fresh credentials.

## what gets logged

the `aws-actions/configure-aws-credentials` action masks the credentials in logs automatically. you'll see this in the workflow output.

```
env:
  AWS_ACCESS_KEY_ID: ***
  AWS_SECRET_ACCESS_KEY: ***
  AWS_SESSION_TOKEN: ***
```

all three values are redacted. even in public repos, these tokens remain hidden.

## the result

now when i push to main, the workflow builds the hugo site and deploys to s3. no manual steps. no stored credentials. the whole flow takes about 12 seconds.

the oidc setup took maybe 10 minutes. worth it for the security improvement and not having to think about key rotation.

## references

- [github docs on oidc](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [aws docs on oidc](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
