
## Developer desktop flow

The developer writes Terraform on their workstation and runs terraform plan through the Guardian CLI. Plan is the only Terraform command the CLI allows on the desktop. The CLI uses a read-only role, so the plan can read current state but nothing can change infrastructure. The plan output goes to the central Checkov engine in the Compliance service account. Checkov checks it against the enterprise policy repo (CIS benchmarks and data perimeter rules) and sends the pass/fail result back to the CLI.
This scan is feedback only. It lets the developer find and fix violations before pushing, but it doesn't block or approve anything. Nothing is deployed, and nothing is written to Redshift.


## GitLab pipeline flow

The developer pushes code and opens a merge request in their customer's GitLab subgroup. The pipeline runs on that subgroup's GitLab runner, an EC2-based container in the customer's Services account. The pipeline calls the Guardian CLI, which runs terraform init and terraform plan and converts the plan to tfplan.json. The CLI submits that file to the Guardian API Gateway endpoint in the Compliance service account. The API passes it to the Guardian Lambda orchestrator, and the Lambda has the central Checkov engine scan it against the same policy repo the desktop uses.

The Checkov result decides what happens next, and it's posted back to the merge request as a comment either way:

Checkov fails (exit 1): the merge request is blocked with the violation details, and no infrastructure is touched.
Checkov passes (exit 0): the Lambda assumes the deployment role in the target Dev, Staging or Prod account and starts CodeBuild there to run terraform apply.

Apply, destroy and every other Terraform command beyond plan are only available through this pipeline. CodeBuild's CloudWatch logs go back to the Compliance account through Kinesis into Redshift. Those build logs are the only thing stored in Redshift; plan and scan results are not.

<img width="672" height="694" alt="image" src="https://github.com/user-attachments/assets/821acd43-fdd1-4a40-9508-b9d533866b2a" />

Another view

<img width="1402" height="868" alt="image" src="https://github.com/user-attachments/assets/bbec439b-3be6-4bd1-8631-4f780225a680" />
