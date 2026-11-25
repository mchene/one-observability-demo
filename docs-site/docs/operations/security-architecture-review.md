# Security Architecture Review Procedure

| | |
|---|---|
| **Document ID** | SEC-ARCH-REVIEW-001 |
| **Environment** | PetAdoptions (One Observability Demo) |
| **Control mapping** | SSAE-18 SOC 2 CC6.1, requirement R2 (security review of new system architectures before go-live) |
| **Owner** | Security Lead (assign name in the Revision History table) |
| **Version** | 1.0 |
| **Review cycle** | At least every 12 months, and after any major change to the pipeline or architecture |

## 1. Purpose

This procedure defines how the security of a new or changed PetAdoptions architecture is reviewed and approved **before** it is deployed to a live environment. It produces a retained record for each review so that the control can be evidenced.

## 2. Scope

The procedure applies to everything deployed by the CDK pipeline (`src/cdk/`) and the application code under `src/applications/`, in every AWS account that hosts PetAdoptions (workshop, staging, production-like).

Components in scope include the VPC and network layout, ECS and EKS workloads, Bedrock AgentCore agents, Lambda functions, canaries, Aurora PostgreSQL, DynamoDB, S3, SQS, EventBridge, OpenSearch, CloudFront and WAF, and IAM roles and policies.

## 3. Roles and responsibilities

| Role | Responsibility |
|---|---|
| **Change Author** | Proposes the change, completes Section 6 (Design Review Request), and fixes findings. |
| **Reviewer (Architect / Security Engineer)** | Performs the review in Section 5, records findings, and recommends approve, approve with conditions, or reject. The Reviewer must not be the Change Author. |
| **Approver (Security Lead / CISO delegate)** | Grants or withholds final approval and signs the record. Accepts any residual risk in writing. |
| **Change Advisory Board (CAB) / Maintainers** | Confirms the change is scheduled and that an approved review exists before merge to `main`. For a small team this can be the repository maintainers. |

One person may hold the Reviewer and Approver roles only if no second qualified person is available. The record must then state this and the Change Author may not hold either role.

## 4. Review triggers

A review is **required** before go-live when a change does any of the following:

1. Introduces a new service, microservice, Lambda, agent runtime, or CDK stack/stage.
2. Adds or changes a network path: new subnet, security group or NACL rule, VPC endpoint, load balancer, CloudFront distribution, public endpoint, or cross-account/peering link.
3. Adds or changes IAM roles, trust policies, resource policies, or permission boundaries (including widening existing permissions).
4. Introduces a new data store, a new data flow, or changes the data classification of existing data.
5. Changes encryption (KMS keys, TLS settings, at-rest settings) or secrets handling.
6. Adds a new third-party dependency, container base image, external integration, or AI model/tool access.
7. Changes logging, monitoring, or audit configuration (CloudTrail, CloudWatch, Config).
8. Changes the deployment pipeline itself (`src/cdk/lib/pipeline.ts`, CodeBuild/CodePipeline roles).

A review is **not required** for changes that only alter application business logic without touching items 1 to 8, documentation, or dependency patch updates that pass automated scanning. The pull request must state "No architecture review required" with the reason.

Emergency changes may be deployed first only with Approver verbal approval. A full review must be completed and recorded within 5 business days after deployment.

## 5. What is reviewed

The Reviewer completes the checklist below for every in-scope change. Mark each item Pass, Fail, or N/A with a short note.

### 5.1 Automated checks (evidence attached to the pull request)

| Check | Source |
|---|---|
| CDK synth passes with cdk-nag `AwsSolutionsChecks` and the custom `WorkshopNagPack`; every suppression in `NagSuppressions` has a written justification | `src/cdk/bin/workshop.ts`, `src/cdk/bin/local.ts`, `tests.yml` workflow |
| Security scan (ASH) has no unresolved high or critical findings | `security-scan.yaml` workflow, `.ash/` |
| Secret and credential detection passes (detect-secrets, detect-aws-credentials, detect-private-key) | `pre-commit.yml` workflow, `.secrets.baseline` |
| Dependency vulnerability scanning (Snyk, python-safety) has no unresolved high or critical findings | CI results |
| Build and unit tests pass for every changed component | `build-test.yml`, `tests.yml` |

### 5.2 Manual design review

| Area | Questions |
|---|---|
| **Network exposure** | Which endpoints are public? Is access limited by WAF, security groups, and NACLs? Are VPC endpoints used instead of internet egress where possible? Are subnets placed correctly (public, private, isolated)? |
| **Identity and access** | Do roles follow least privilege? Are wildcard actions or resources justified? Are trust policies scoped with `aws:SourceAccount` / `aws:SourceArn`? Are there long-lived credentials? |
| **Encryption** | Is data encrypted at rest (KMS or service default) and in transit (TLS 1.2 or higher)? Who manages the keys? |
| **Data classification** | Is the data classified, and does the resource carry the `data-classification` tag with the correct value (Internal, Confidential)? Is sensitive data kept out of logs and traces? |
| **Secrets** | Are secrets in Secrets Manager or SSM Parameter Store and never in code, environment files, or logs? |
| **Logging and monitoring** | Are CloudTrail, CloudWatch Logs, Application Signals, and X-Ray enabled for the new component? Are alarms defined for security-relevant events? |
| **Resilience and recovery** | Are backups, deletion protection, and retention set as needed? |
| **Supply chain** | Are base images and packages from trusted sources and pinned? Is the image scanned in ECR? |
| **AI/agent components** | Are Bedrock guardrails configured? Are agent tool permissions limited to what is needed? Is prompt and response data handled according to its classification? |
| **Tagging and ownership** | Do resources carry the required tags, including `data-classification`, `environment`, and `application`? |

## 6. Review workflow and approval

1. **Request.** The Change Author opens a pull request and fills in the Design Review Request (Section 9.1) in the PR description or in a linked issue. Label it `security-review`.
2. **Triage.** The Reviewer confirms whether a review is required (Section 4) within 2 business days.
3. **Review.** The Reviewer completes the checklist (Section 5) and records findings in the Review Record (Section 9.2). Each finding gets a severity: Critical, High, Medium, Low.
4. **Remediation.** Critical and High findings must be fixed before go-live. Medium and Low findings may be accepted by the Approver with a written due date or risk acceptance.
5. **Decision.** The Approver records one decision: **Approved**, **Approved with conditions**, or **Rejected**, and signs by one of: an approving GitHub pull request review that references the Review Record, a closed ticket in the tracking system, or a signed record file.
6. **Gate.** The change merges to `main` (and therefore deploys through the pipeline) only after the decision is recorded. Branch protection should require the Approver's review and passing CI checks.
7. **Retention.** The Review Record is stored in `docs-site/docs/operations/security-reviews/` (or the ticket system) and kept for at least 3 years, or for the audit period the organization requires if that is longer.

## 7. Procedure maintenance

- The Owner reviews this procedure at least **every 12 months** and after any major architecture, pipeline, or compliance-framework change.
- Each review is recorded in the Revision History table, even if no change is made.
- Findings that recur in reviews are fed back into the checklist in Section 5.

## 8. Evidence for auditors

For each audit period, provide:

1. This procedure (current version and revision history).
2. The Review Register (Section 10) listing every review in the period.
3. The completed Review Records for the most recent 1 to 3 PetAdoptions changes, including approver sign-off (PR approval, ticket closure, or signed record).
4. CI output (cdk-nag, ASH, secret scan) linked from each record.

## 9. Templates

### 9.1 Design Review Request

```markdown
**Review ID:** SR-YYYY-NNN
**Change title:**
**Pull request / ticket:**
**Change Author:**
**Target environment / account / region:**
**Planned go-live date:**

**Trigger(s) from Section 4:** (list numbers)
**Summary of change:**
**Architecture diagram or link:**
**New or changed data flows and data classification:**
**New or changed IAM roles / network paths / public endpoints:**
**Known risks or exceptions (including cdk-nag suppressions):**
```

### 9.2 Review Record

```markdown
**Review ID:** SR-YYYY-NNN
**Reviewer:**                      **Date reviewed:**
**Approver:**                      **Date decided:**

**Automated checks** (link to CI run)
- cdk-nag / synth:        Pass | Fail
- ASH security scan:      Pass | Fail
- Secret detection:       Pass | Fail
- Dependency scan:        Pass | Fail

**Manual checklist (Section 5.2)**
| Area | Result (Pass/Fail/N/A) | Notes |
|---|---|---|
| Network exposure | | |
| Identity and access | | |
| Encryption | | |
| Data classification | | |
| Secrets | | |
| Logging and monitoring | | |
| Resilience and recovery | | |
| Supply chain | | |
| AI/agent components | | |
| Tagging and ownership | | |

**Findings**
| # | Severity | Description | Owner | Due | Status |
|---|---|---|---|---|---|

**Decision:** Approved | Approved with conditions | Rejected
**Conditions / accepted risks:**
**Approver sign-off (name, date, link to PR approval or ticket):**
```

## 10. Review Register

Add one row for every completed review. Link each row to its Review Record.

| Review ID | Date | Change / PR | Reviewer | Approver | Decision | Record link |
|---|---|---|---|---|---|---|
| | | | | | | |

## 11. Revision history

| Version | Date | Author | Change | Reviewed by |
|---|---|---|---|---|
| 1.0 | | | Initial procedure | |
