# Scope: generate code, not an end-to-end tech solution

Status: positioning principle, captured from a brainstorm on
2026-06-13. Likely belongs in the README and/or the site
eventually, not only in future-work.

---

## The boundary

Code from Spec answers a precise question: **how to use AI to
turn knowledge into verified code** — not "how to generate a
complete end-to-end tech solution." The architecture of the
*running solution* (SQL database, Redis cache, AWS EKS, etc.)
is out of scope.

The dividing line is not "technology in or out." It is
**decision/provisioning vs. generated software**:

**Out of scope — the operational solution architecture:**
- where it runs (EKS, EC2, Lambda);
- provisioned resources (the Redis cluster, the RDS instance,
  the VPC);
- infrastructure-as-code (Terraform, Helm);
- deployment and operations (scaling, monitoring,
  platform-level secrets).

**In scope — the software and what is needed to generate it:**
- the components and their interfaces — including the component
  that talks to the database (it exists, has an interface, its
  code is generated);
- the technical constraints that shape the generated code.

The subtlety that prevents confusion with the earlier
"decide the solution shape" step: the choice "we use Postgres,
serializable isolation, Redis for cache" appears in the tree
**only as a constraint** — a guard node, an input to
generation. The framework-mcp guard node does exactly this
("serializable default, direct SQL, no ORM"). The tree
**records** the decision so the code respects it; it does not
**make** the decision or **provision** anything. CFS consumes
"we use Postgres" the way it consumes a domain rule: as input.

## The one-liner

**Code from Spec specifies the software that uses the database;
it does not choose, provision, or deploy the database.** It sits
*downstream* of the architecture decision (which enters as a
constraint) and *upstream* of the running system (another
tool's job). It owns neither end — it owns the software in the
middle.

## The gray zone: database schema

The schema splits. The schema *as a contract the code consumes*
can live in the tree (via `EXTERNAL/` or extraction). The
schema *as a provisioned object* (migrations applied to a
running database) is operational, out of scope. The spec/
contract side is in; the provisioned side is out. Same logic as
everything else: the logical plane is CFS's, the physical plane
is not.

## Why this is a virtue, not a limitation

- It keeps CFS focused on its hard, under-solved problem
  (knowledge → verified software) instead of becoming a
  do-everything tool that does each thing poorly.
- The other ends already have excellent owners (Terraform/
  Pulumi for provisioning, Helm/Argo for deploy, Datadog for
  ops). CFS fits *between* them, it does not compete.
- It separates two planes cleanly: CFS lives in the logical
  plane (what the software *is and does*); infrastructure lives
  in the physical/operational plane (where and how it *runs*).

Consistent with the established "deploy is not our scope"
boundary and with the site's framing that code generation is
the easy part — with the missing complement: even the hard part,
CFS owns only the stretch from knowledge to code.
