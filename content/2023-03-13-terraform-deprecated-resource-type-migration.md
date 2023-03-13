+++

title = "Terraform: Migrating deprecated resource type name"
date = "2023-03-13T21:43:00.00Z"
+++

This is a short post noting down how to migrate a deprecated resource type name to its new name.
I needed to perform this migration when updating the cloudflare provider. 
<!-- more -->
Somewhere between versions `3.11.0` and `4.1.0`, the Cloudflare Terraform provider deprecated `cloudflare_argo_tunnel` resource type and renamed it to `cloudflare_tunnel`.

With Terraform, you can't simply rename the resource types. We need to remove the old resource from Terraform state and then import the new resource type. For this resource, the import uses `<account_id>/<tunnel_id>` notation, so let's `terraform show` and note down the account ID and tunnel ID of the `cloudflare_argo_tunnel` resource.

Then, let's bump up the provider version and run `terraform init -upgrade`.

Then, remove the deprecated state:
```bash
terraform state rm module.cloudflare.cloudflare_argo_tunnel.homelab
```
And import the new resource type:
```bash
terraform import module.cloudflare.cloudflare_tunnel.homelab <account_id>/<tunnel_id>
```

And we have migrated!

