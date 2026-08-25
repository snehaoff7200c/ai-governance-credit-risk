# Access Control — Credit Risk Governance Project

## Intent
To apply least-privilege access on the `credit_risk` BigQuery dataset, restricting read/write access to only the roles that genuinely need it — the same principle applied to production data platforms (e.g., IAM least-privilege role design at EY, restricting encrypted PII access to only authorized principals).

## What was configured
A `BigQuery Data Viewer` role was granted to the owner's account on the `credit_risk` dataset, intended to demonstrate read-only access provisioning for a hypothetical downstream consumer (e.g., a risk analyst who should be able to review governed data without modifying it).

## Finding: additive IAM makes this ineffective as configured
On reviewing the resulting permissions panel, the same account already held `BigQuery Data Owner` access, inherited automatically from the Owner role at the project level (as the project creator). Google Cloud IAM permissions are **additive, not restrictive** — granting a lower-privilege role on top of an existing higher-privilege one does not downgrade or restrict access. The account's effective permissions remained full Owner-level access despite the added Viewer role.

**This is a real and common governance failure mode**, not specific to this project: access reviews that check "was a restrictive role added?" without checking "does the principal already hold broader access some other way?" can create a false sense of least-privilege compliance. The correct review question is always about a principal's *effective* combined permissions, not any single role binding in isolation.

## What a correct implementation would require
A genuine least-privilege demonstration needs a **second principal with no pre-existing project-level role** — e.g., a separate Google account or service account granted only `BigQuery Data Viewer` at the dataset level, with no Owner/Editor binding at the project level to inherit from. This was not implemented in this pass, and is noted here as a scoped-out next step rather than silently omitted.

## Takeaway for review
Effective access = union of all role bindings a principal holds, across every level (project, dataset, table). Least-privilege reviews must check the *effective* permission set per principal, not just whether a restrictive role was technically added.
