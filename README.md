# terraform-cloud-run

## Overview

A Github action to create runs on Terraform Cloud/Enterprise workspaces and fetch workspace outputs to use in Github workflows.

**Note**: This Github action currently only supports a limited set of options for creating a run on a given workspace.

### When are workspace outputs fetched?

Outputs are fetched from a workspace and made available as outputs for your Github action if:
- The run created is **not** a destroy plan **and** the runner is configured to wait for the run to complete.
- If the run is skipped entirely using `skip-run: true`

### Inputs

- `tfe_token` (**Required**): The token of the TFC/E instance which holds the workspace that manages your tflocal instance.
- `organization` (**Required**): The TFC/E organization that manages the specified workspace.
- `workspace` (**Required**): The name of the TFC/E workspace that manages the tflocal configuration.
- `tfe_hostname` (**Optional**): The hostname of the TFC/E instance which holds the workspace that manages your tflocal instance. Defaults to `app.terraform.io`.
- `skip-run` (**Optional**): If set to `true`, will omit creating a run on the workspace and simply fetch the workspace outputs. Defaults to `false`.
- `wait-for-run` (**Optional**): If set to `true`, runs executed by this action will poll and await completion. Defaults to `false`.

#### Run Creation Options
- `is-destroy` (**Optional**): If set to `true`, will trigger a destroy run on the TFC workspace. Defaults to `false`.
- `auto-apply` (**Optional**): Whether to automatically apply changes when a Terraform plan is successful. Defaults to `true`.
- `message` (**Optional**): The message to associate with the run. Defaults to `Run queued by terraform-cloud-run Github action`.
- `replace-addrs` (**Optional**): A multi-line list of resource addresses to be replaced.
- `target-addrs` (**Optional**): A multi-line list of resources addresses that Terraform should focus its planning efforts on.

[Read more about the Runs API](https://developer.hashicorp.com/terraform/cloud-docs/api-docs/run#create-a-run)

### Outputs

- `run-id`: The ID of the run that was created.
- `workspace-outputs`: A JSON-stringified object containing the outputs fetched from the specified Terraform Cloud workspace. Output names will match those found in your workspace. Sensitive output values will be redacted from runner logs.

You will need to use `fromJSON()` to parse `workspace-outputs` in your workflow. [Read more about fromJSON()](https://docs.github.com/en/actions/learn-github-actions/expressions#fromjson)

**Example**

```yaml
env:
  AWS_ACCESS_KEY_ID: ${{ fromJSON(steps.terraform-cloud-run.outputs.workspace-outputs).aws_access_key_id }}
  AWS_SECRET_ACCESS_KEY: ${{ fromJSON(steps.terraform-cloud-run.outputs.workspace-outputs).aws_secret_access_key }}
```

Workspace outputs that are marked as sensitive will be set as secret and redacted in the workflow.

## Examples

### Nightly instance rebuild

```yaml
name: Nightly Rebuild
on:
  workflow_dispatch:
  schedule:
    - cron: 0 0 * * *

jobs:
  instance:
    runs-on: ubuntu-latest
    steps:
     - name: Checkout code
       uses: actions/checkout@v3

     - name: Create Apply Run
       id: apply
       uses: hashicorp-forge/terraform-cloud-run
       with:
        tfe_hostname: "app.terraform.io"
        tfe_token: ${{ secrets.TFE_TOKEN }}
        organization: "hashicorp"
        workspace: "my-tflocal-workspace"
        replace-addrs: |
          module.some_mod.resource_name.foo
          module.some_other_mod.resource_name.bar
```

### Ephemeral CI Instance w/ Tests
```yaml
name: Nightly Test
on:
  workflow_dispatch:
  schedule:
    - cron: 0 0 * * *

jobs:
  instance:
    runs-on: ubuntu-latest
    steps:
     - name: Checkout code
       uses: actions/checkout@v3

     - name: Create run
       id: apply
       uses: hashicorp-forge/terraform-cloud-run
       with:
        tfe_hostname: "app.terraform.io"
        tfe_token: ${{ secrets.TFE_TOKEN }}
        organization: "hashicorp"
        workspace: "my-tflocal-workspace"
        wait-for-run: true
    outputs:
      some_output_foo: ${{ fromJSON(steps.apply.outputs.workspace-outputs).some_output_foo }}
      some_output_bar: ${{ fromJSON(steps.apply.outputs.workspace-outputs).some_output_bar }}

  # Run some test job
  tests:
    runs-on: ubuntu-latest
    needs: instance
    steps:
      - name: Some tests
        env:
          SOME_VAR: ${{ needs.instance.outputs.some_output_foo }}
          SOME_OTHER_VAR: ${{ needs.instance.outputs.some_output_bar }}

  # Destroy the instance
  cleanup:
    runs-on: ubuntu-latest
    needs: [tests]
    if: "${{ always() }}"
    steps:
     - name: Destroy workspace
       id: tflocal
       uses: hashicorp-forge/terraform-cloud-run
       with:
        tfe_hostname: "app.terraform.io"
        tfe_token: ${{ secrets.TFE_TOKEN }}
        organization: "hashicorp"
        workspace: "my-tflocal-workspace"
        destroy: true
```




