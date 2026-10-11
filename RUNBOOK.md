# TEAM runbook (jalcalaroot backup)

Personal backup of [`aws-samples/iam-identity-center-team`](https://github.com/aws-samples/iam-identity-center-team) (TEAM: Temporary Elevated Access Management for AWS IAM Identity Center), pinned at **v1.5.2**. This file is the short version of what is needed to deploy it later. The authoritative docs are in [`docs/docs/`](docs/docs) and online at https://aws-samples.github.io/iam-identity-center-team/.

> Status: **not deployed**. Nothing here has been run in the `jalcalaroot` account yet.

## What it is

A serverless web app (Amplify, AppSync, Lambda, DynamoDB, Step Functions, Cognito) that gives **time-bound, approved, audited access** to AWS accounts. A user requests a permission set on an account for a duration, an approver accepts, TEAM attaches the permission set, and removes it when the time is up. Sessions are logged in CloudTrail Lake. It is the AWS counterpart of Microsoft Entra PIM, but self-hosted and unsupported (AWS Samples).

## Prerequisites

| Requirement | Detail |
|---|---|
| AWS Organizations | A real organization. TEAM cannot work with standalone accounts. |
| IAM Identity Center | **Organization instance** (enabled from the management account), in the same region as TEAM. An account instance is not enough. |
| Dedicated TEAM account | A member account, nominated as delegated admin for IAM Identity Center, CloudTrail and Account management. Do not run other workloads in it. |
| CloudTrail Lake | An **organization event datastore** in the TEAM account. This is the main cost driver. |
| Permission sets | The permission sets users will be able to request (predefined or custom, least privilege). |
| Groups | Two Identity Center groups: TEAM admins and TEAM auditors. They can be synced from Entra ID. |
| Source repository | A secret in Secrets Manager with the repo URL and a **classic** GitHub personal access token (fine-grained tokens do not work). Point it to this fork. Without it, `deploy.sh` creates a CodeCommit repository instead. |
| Workstation | AWS CLI, `jq`, `git-remote-codecommit`, and two named CLI profiles: one for the management account and one for the TEAM account. AWS CloudShell works too. |

Hard limits (by design): TEAM cannot grant temporary access to the **management account** nor manage permission sets provisioned in it.

## Deploy

```bash
git clone https://github.com/jalcalaroot/aws-team.git && cd aws-team/deployment
cp -n parameters-template.sh parameters.sh      # then edit it
./init.sh      # makes TEAM_ACCOUNT delegated admin (run with the management profile)
./deploy.sh    # creates the Amplify app and pipeline in TEAM_ACCOUNT, ~20 min
```

Required values in `parameters.sh`: `IDC_LOGIN_URL`, `REGION` (same as Identity Center), `TEAM_ACCOUNT`, `ORG_MASTER_PROFILE`, `TEAM_ACCOUNT_PROFILE`, `TEAM_ADMIN_GROUP`, `TEAM_AUDITOR_GROUP`, `CLOUDTRAIL_AUDIT_LOGS` (ARN of the event datastore), `SECRET_NAME`. Optional: `TAGS`, `UI_DOMAIN`, `CACHE_TTL`. `parameters.sh` holds account IDs; it is already in `.gitignore`, never force-add it.

Then: verify the app in Amplify (**TEAM-IDC-APP**), add TEAM as a SAML application in Identity Center, and configure eligibility and approval policies from the TEAM admin page (see `docs/docs/deployment/configuration/`). Deploying into the management account is possible (`parameters-mgmt-template.sh`, skip `init.sh`) but discouraged.

## Connecting Microsoft Entra ID

Entra acts as the external identity provider of Identity Center (SAML for sign-in, SCIM for users and groups, if the tenant licence allows provisioning). Entra is only the identity source: the temporary elevation lives in TEAM, so Entra PIM (P2 licence) is not needed. Tutorial: *Configure SAML and SCIM with Microsoft Entra ID and IAM Identity Center* (AWS docs).

## Cost

Identity Center and Organizations are free. TEAM is usage-based (Amplify, AppSync, DynamoDB, Lambda, Step Functions, Cognito, Secrets Manager). **CloudTrail Lake** is billed for ingestion and storage by volume of management events. Estimate with the AWS Pricing Calculator before deploying.

## Update and uninstall

```bash
git fetch upstream --tags && git merge upstream/main     # refresh this backup from AWS
cd deployment && ./update.sh                              # redeploys a running TEAM (it pulls from aws-samples, see docs/docs/deployment/update.md)
./destroy.sh                                              # (from deployment/) removes the Amplify stacks
```

`destroy.sh` does not remove the delegated-admin configuration, permission set templates, the TEAM SAML application in Identity Center, nor the Amplify deployment bucket (`amplify-teamidcapp-main-*-deployment`, versioned: empty it from the console). Remove those by hand.

## Security notes

TEAM controls access to your accounts: treat its account as highly privileged. Set up break-glass access (the regional dependency on Cognito and Identity Center can make TEAM unavailable). Consider WAF on the AppSync API and S3 access logging on the Amplify bucket. See `docs/docs/overview/security.md`.

## Remotes

- `origin`: `jalcalaroot/aws-team` (this backup)
- `upstream`: `aws-samples/iam-identity-center-team` (source of updates)
