# Multi-Environment Deployment Pipeline (GitHub Actions + OIDC)

A CI/CD pipeline that deploys the same CloudFormation template to separate dev and prod environments on AWS, with no stored AWS keys, environment-scoped IAM roles, and a manual approval gate before production.

This is Project 3 in a series. See also [accessibility-scanner](https://github.com/Canzo32/accessibility-scanner) and the scalable scanning pipeline.

## Problem

Long-lived AWS access keys stored in CI are a common breach path, and a single deploy role that can touch every environment means a mistake in dev can hit prod. I wanted credentials that expire, roles that can only touch their own environment, and a human gate before production.

## Architecture

```
push to main
   |
   v
GitHub Actions (OIDC token, no stored keys)
   |
   +--> deploy-dev  --> role GitHubDeployDev  --> CloudFormation stack scanner-dev
   |
   +--> approval gate (required reviewer)
   |
   +--> deploy-prod --> role GitHubDeployProd --> CloudFormation stack scanner-prod
```

## Design Decisions

| Choice | Reason |
|---|---|
| OIDC instead of access keys | Short-lived credentials, nothing to leak or rotate |
| One IAM role per environment | A dev job cannot assume the prod role |
| Trust policy scoped to `environment:dev` / `environment:prod` | Only the matching GitHub environment can assume each role |
| Inline policies limited to `scanner-dev-*` / `scanner-prod-*` | Least privilege per environment |
| Required reviewer on `prod` | Human approval before production changes |
| Separate VPCs, no NAT gateways | Network isolation at zero cost |
| CloudTrail trail (management events) | Audit trail of who created what |

## Lessons Learned

- **"Not authorized to perform sts:AssumeRoleWithWebIdentity" gives no detail.** The role, provider, audience and workflow were all correct. CloudTrail showed the real `sub` claim: it contained numeric owner and repository IDs (`repo:Canzo32@<id>/multi-env-pipeline@<id>:environment:dev`), so my trust policy pattern never matched. I fixed it with wildcards around the ID segments.
- **A re-run replays the old commit.** Edits to the workflow need a new push to take effect.
- **A wrong role name and a rejecting role return the same error**, so verify names before debugging conditions.
- **`aws cloudformation deploy` uses change sets**, so CloudTrail shows `CreateChangeSet` and `ExecuteChangeSet` rather than `CreateStack`.

## Known Limitations and What I'd Improve

- Use separate AWS accounts for dev and prod, not just separate roles.
- Pin the trust policy to exact repository IDs with `StringEquals`.
- Add a CloudFormation change-set review step before prod.
- Add automated tests and linting (cfn-lint) before deploy.

## Cost

Free Tier only. A zero-spend budget alert was active, there are no NAT gateways or always-on compute, and both stacks were deleted after testing.

## Repository Structure

```
.github/workflows/deploy.yml   pipeline definition
template.yaml                  CloudFormation template (Lambda function)
screenshots/                   4 proof screenshots
```
