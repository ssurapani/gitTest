
## Developer desktop flow

The developer writes Terraform on their workstation and runs terraform plan through the Guardian CLI. Plan is the only Terraform command the CLI allows on the desktop. The CLI uses a read-only role, so the plan can read current state but nothing can change infrastructure. The plan output goes to the central Checkov engine in the Compliance service account. Checkov checks it against the enterprise policy repo (CIS benchmarks and data perimeter rules) and sends the pass/fail result back to the CLI.
This scan is feedback only. It lets the developer find and fix violations before pushing, but it doesn't block or approve anything. Nothing is deployed, and nothing is written to Redshift.

