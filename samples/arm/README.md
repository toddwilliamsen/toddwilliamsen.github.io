# ARM sample templates

Educational Azure Resource Manager samples that accompany the [IaC article series](/#writing).

| Sample | Article | What it shows |
| --- | --- | --- |
| [fundamentals/storage-account.json](fundamentals/storage-account.json) | Part 1 | Parameters, variables, pinned API version, tags, explicit public access |
| [modular/network.json](modular/network.json) + [main.json](modular/main.json) | Part 2 | Module outputs + nested composition |
| [advanced/vnet-copy-and-condition.json](advanced/vnet-copy-and-condition.json) | Part 3 | `copy` for subnets + `condition` for optional public IP |
| [nuances/identity-rbac-securestring.json](nuances/identity-rbac-securestring.json) | Part 4 | Deterministic role assignment name, `dependsOn`, `secureString` |

## Deploy a sample

```bash
az group create -n rg-arm-samples -l eastus

az deployment group create \
  -g rg-arm-samples \
  -f samples/arm/fundamentals/storage-account.json \
  -p @samples/arm/fundamentals/parameters.dev.json
```

For the nuances sample, pass the secure value at deploy time (do not put it in a parameters file):

```bash
az deployment group create \
  -g rg-arm-samples \
  -f samples/arm/nuances/identity-rbac-securestring.json \
  -p bootstrapSecret=replace-me-at-deploy-time
```

These are teaching aids, not production landing-zone modules. Tighten SKUs, private endpoints, diagnostics, and RBAC before anything faces a real workload.
