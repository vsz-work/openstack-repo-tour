# Running sunbeam-terraform checks locally

All commands run from the repo root (`sunbeam-terraform/`). The only tool you
need is `terraform` (`sudo snap install --classic terraform`), which is the same
way CI installs it (`.github/workflows/check.yml:15`). The targets are defined
in `Makefile:18-27`.

## Through make (the same checks CI runs)

```sh
make check     # terraform init + fmt -check -recursive + validate (what CI runs)
make init      # terraform init only (downloads the juju provider, pinned = 2.3.1)
make clean     # remove every .terraform/ dir and .terraform.lock.hcl
```

`make check` runs on every PR and on every push to `main`
(`.github/workflows/pr.yaml`, `.github/workflows/main.yml`).

## Direct commands (faster when iterating)

### Formatting

`terraform fmt` works without `init` and covers `modules/*` when you pass
`-recursive`.

```sh
# check only (what CI does); -diff shows what would change
terraform fmt -check -recursive -diff

# fix in place
terraform fmt -recursive

# a single file
terraform fmt main.tf
```

### Validation

`validate` needs `init` first, but it never contacts a Juju controller, so it
works on a laptop with no Juju installed. Validating the root plan also
validates every module it calls (`modules/openstack-api`, `mysql`, `rabbitmq`,
`consul`).

```sh
terraform init          # once, or after changing provider/module sources
terraform validate
terraform validate -json | jq '.diagnostics'   # machine-readable errors
```

If you switch branches (for example to `stable/2024.1`) and `init` complains
about the provider or lock file, run `make clean && make init`.
`.terraform.lock.hcl` is gitignored, so this is safe.

## Full deploy (what the `deploy` CI job does)

`.github/workflows/deploy.yml` runs on a self-hosted `xlarge` runner. It does
the following:

1. Uses concierge to bootstrap Juju 3.6 on LXD and add a Canonical K8s cloud
   (`deploy.yml:31-62`).
2. Deploys MicroOVN, `sunbeam-ovn-proxy` and `microcluster-token-distributor`
   in a machine model, then offers `ovsdb-cms` (`deploy.yml:68-79`).
3. Writes a `terraform.tfvars.json` that switches on most `enable-*` flags,
   then runs `terraform apply` with `channels/edge.tfvars` (`deploy.yml:82-85`).
4. Integrates MicroOVN with the `ca-offer-url` output and waits on both models
   with `juju-wait` (`deploy.yml:86-88`).

This step needs a large machine (many GB of RAM, about an hour). To run it by
hand, copy the shell from `deploy.yml` step by step rather than scripting it.
The minimum is:

```sh
# with a Juju controller and a k8s cloud already available
juju switch <controller>:<some-model>
echo '{"cloud": "<k8s-cloud>", "credential": "<k8s-cloud>",
       "external-ovsdb-cms-offer-url": "admin/openstack-machines.sunbeam-ovn-proxy"}' \
  > terraform.tfvars.json
terraform init
terraform plan  -var-file=channels/edge.tfvars    # inspect what would be created
terraform apply -var-file=channels/edge.tfvars
```

`terraform plan` *does* connect to the controller (the juju provider reads
models and offers), so it is not a substitute for `validate` on a laptop.
`terraform.tfvars.json` isn't gitignored, so don't commit it.

## Typical pre-push routine

```sh
terraform fmt -recursive   # fix formatting first
make check                 # then run what CI runs
```
