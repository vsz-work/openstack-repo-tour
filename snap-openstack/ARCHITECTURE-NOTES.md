# snap-openstack: maintainer orientation

A guide to the `canonical/snap-openstack` repo (Sunbeam / Canonical OpenStack), written from `main` at `b659a406`.

**Path convention:** Python paths are relative to `sunbeam-python/sunbeam/`. Paths that start with a top-level folder (`snap/`, `cloud/`, `sunbeam-microcluster/`, `snap-wrappers/`, `manifests/`) are relative to the repo root. Line numbers apply to that commit and will drift.

---

## 1. Repo map

### Top-level layout

| Path | What it is |
|---|---|
| `snap/snapcraft.yaml` | Builds the `openstack` snap (core24, strict). It contains `sunbeamd` (Go), terraform v1.15.8 and the Terraform plans: the local ones from `cloud/` plus `canonical/sunbeam-terraform`, which becomes `etc/deploy-openstack`. It also bundles clusterctl, the manifests, and the Python CLI with the OpenStack clients. |
| `sunbeam-python/` | The `sunbeam` CLI, which does all the orchestration. Packaging is in `pyproject.toml`, tests are under `tests/unit` and `tests/functional`. |
| `sunbeam-microcluster/` | `sunbeamd`, the clusterd daemon. It's written in Go on canonical/microcluster v2.2.0 with a dqlite database. |
| `cloud/etc/deploy-*` | Terraform plans that ship in the snap: sunbeam-machine, k8s, microceph, microovn, role-distributor, cinder-volume, openstack-hypervisor, storage and demo-setup. |
| `sunbeam-python/sunbeam/features/*/etc/` | Terraform plans that belong to individual features (loadbalancer CNI, COS, pro, manila-data, consul…). |
| `manifests/<release>/<risk>.yml` | Default manifests embedded in the snap, one per risk level. The ones for 2026.1 are symlinked into `etc/manifests/` (the `manifests` part of `snap/snapcraft.yaml`). |
| `snap-wrappers/` | `commands/clusterd.start` launches `sunbeamd --state-dir $SNAP_COMMON/state`. Also holds the shell completions. |
| `tools/` | Developer scripts (`add_charm_revisions.py`, `fast8.sh`). |
| `docs/` | Documentation. |

### Python package layout (`sunbeam-python/sunbeam/`)

| Path | Contents |
|---|---|
| `main.py` | The click root and command registration |
| `commands/` | Top-level commands: configure, launch, openrc, manifest, proxy, refresh, resize, backup/restore, sso, … |
| `core/` | Framework: steps, terraform, juju, manifest, deployment, checks, k8s, questions |
| `steps/` | Reusable step classes, one module per component |
| `provider/local`, `provider/maas` | Deployment providers and their `cluster …` commands |
| `clusterd/` | REST client for `sunbeamd` |
| `features/` | Pluggable features (dns, loadbalancer, vault, tls, observability, …) |
| `storage/` | Plugin system for storage backends |
| `hooks.py` | Snap hooks |
| `feature_gates.py` | Feature gating |
| `versions.py` | Charm channels and tfvar maps |

### Entry points

- **CLI:** `sunbeam = sunbeam.main:main` (`sunbeam-python/pyproject.toml`). `main.py:106` builds the click tree. It registers the provider commands (`main.py:143-147`), storage backends (`:176`) and features (`:179`), then calls `cli(obj=deployment)`.
- **Snap hooks:** `install`, `configure` and `post-refresh` map to `hooks.py:213`, `:242` and `:230` through the snaphelpers entry points declared in `pyproject.toml`.
- **Daemon:** `sunbeam-microcluster/cmd/sunbeamd/main.go:63` (`Run`). Its hooks start at `:71`; `OnStart` starts the feature-gate sync at `:91`.
- **Snap apps** (`snap/snapcraft.yaml`): `openstack` (the OpenStack client), `clusterd` (daemon), `sunbeam`, `terraform`.

### Build and test tooling

- Python uses `uv` and `tox` (`sunbeam-python/tox.ini`):
  - `unit` runs pytest on `tests/unit`
  - `pep8` and `fmt` run ruff
  - `mypy` type-checks
  - `functional` needs real hardware
  - `cover` measures coverage
- Go has its own `sunbeam-microcluster/Makefile`.
- The snap is built with `snapcraft`. The Python part generates `requirements.txt` with `uv export` and writes the snap hooks with `snap-helpers write-hooks`.
- CI lives in `.github/workflows/` (`pr-python.yml`, `pr-golang.yml`, `build-snap.yml`, `test-snap.yml`, `test-python.yml`, `test-golang.yml`, TiCS).

---

## 2. Core responsibilities and abstractions

Sunbeam is a client-side orchestrator. Each CLI command builds a list of **Steps**, runs them, and drives Juju through Terraform (terraform-provider-juju). State that has to be shared between nodes lives in **clusterd**.

| Abstraction | Where | Role |
|---|---|---|
| `BaseStep` | `core/common.py:251` | Methods `prompt` / `is_skip` / `run`, each returning a `Result`. |
| `run_plan` | `core/common.py:317` | Runs steps in order. It prompts first, records a skip and moves on, and raises a `ClickException` on FAILED. |
| `DeployMachineApplicationStep` | `core/steps.py:59` | Base class for "apply one Terraform plan, then wait for the Juju app". It sets `machine_ids` from the clusterd node roles. |
| `Deployment` | `core/deployment.py:100` | Model of one deployment (pydantic). Supplies the client, the manifest (`:335`), tfhelpers (`:399`, `:439`) and the feature manager. |
| `LocalDeployment`, `MaasDeployment` | `provider/local/deployment.py`, `provider/maas/deployment.py` | Provider subclasses. `ProviderBase` is in `provider/base.py`, and `LocalProvider` registers `cluster bootstrap/add/join/...` at `provider/local/commands.py:224-251`. |
| `DeploymentsConfig` | `core/deployments.py` | The local `~/.local/share/openstack/deployments.yaml` (`:19`). |
| `TerraformHelper` | `core/terraform.py:88` | Wraps one plan directory. It writes `backend.tf` (an HTTP backend pointing at clusterd), merges tfvars, and runs init/apply/output. |
| `TerraformInitStep` | `core/terraform.py:975` | Runs `terraform init` for a plan. |
| `Manifest` | `core/manifest.py:413` | Pydantic model with `core`, `features` and `storage`, plus merge and validate. |
| `Client` / `ClusterService` | `clusterd/client.py:58`, `clusterd/cluster.py:372` | REST client for clusterd, over the unix socket or HTTPS with mTLS. |
| `JujuHelper` | `core/juju.py:252` | Juju access through `jubilant` and the `juju` CLI (`_juju_cmd`, `core/juju.py:1980`). |
| `BaseFeature`, `EnableDisableFeature`, `OpenStackControlPlaneFeature` | `features/interface/v1/base.py:306` and `:606`, `features/interface/v1/openstack.py:86` | Plugin framework for `sunbeam enable/disable X`. |
| `FeatureManager` | `feature_manager.py:234` | Imports every `features/*/feature.py` (`:251`) and registers their commands (`:371`). |
| `StorageBackendManager` | `storage/manager.py` | Plugin system for storage backends (cinder backends). |
| Version and tfvar maps | `versions.py` | Charm channels, `TERRAFORM_DIR_NAMES` (`:99`) and `MANIFEST_ATTRIBUTES_TFVAR_MAP` (`:312`). |

---

## 3. Integration points

### clusterd REST API (Python ↔ Go `sunbeamd`)

- **Transport:** `Client.from_socket()` uses `$SNAP_COMMON/state/control.socket` (`clusterd/client.py:109`). Remote access uses HTTPS on `CLUSTERD_PORT` with mTLS (`clusterd/client.py:20-104`, `provider/local/deployment.py:155`).
- **Microcluster core endpoints:** `/core/control` (bootstrap/join), `/core/1.0/cluster` and `/core/control/tokens` (`clusterd/cluster.py:20-93`).
- **Sunbeam's own `/1.0/*` endpoints** are registered in `sunbeam-microcluster/api/servers.go:11-45`: nodes, config, jujuusers, manifests, terraformstate, terraformlock, terraformunlock, status, storage-backend, feature-gates. There's also `/local/certpair/server`.

### Terraform

- **State backend:** every plan uses `backend "http"` pointing at clusterd's `/1.0/terraformstate/{plan}`, with lock/unlock endpoints (`core/terraform.py:40-52`, `:111-124`). The client cert reaches Terraform via `TF_HTTP_CLIENT_CERTIFICATE_PEM` (`core/deployment.py:384-389`).
- **Juju provider credentials** go in as the env vars `JUJU_USERNAME`, `JUJU_PASSWORD`, `JUJU_CONTROLLER_ADDRESSES` and `JUJU_CA_CERT` (`core/deployment.py:372-383`).
- **Providers** are installed from the snap's filesystem mirror `usr/share/terraform-providers` (`core/terraform.py:54-61`).
- **Plan copies:** each plan is copied from the snap into `$SNAP_USER_COMMON/etc/<deployment>/<plan-dir>` (`core/deployment.py:417-430`).
- **Tfvars** come from the manifest-to-tfvar maps (`versions.py:312`) plus overrides computed in each step's `extra_tfvars()`:
  - `steps/hypervisor.py:110` reads the openstack-plan outputs `rabbitmq-offer-url`, `keystone-offer-url`, `cert-distributor-offer-url`, `ca-offer-url`, `nova-offer-url`, … and passes them in as variables.
  - Similar `extra_tfvars()` hooks exist in `steps/microceph.py:129`, `steps/cinder_volume.py:169`, `steps/k8s.py:290`, `steps/sunbeam_machine.py:63` and `steps/microovn.py:152`.

### Juju relations created by the bundled Terraform

The control-plane relations (keystone, nova, neutron, …) live in the external `sunbeam-terraform` repo and aren't listed here.

| Location | This side | Other side |
|---|---|---|
| `cloud/etc/deploy-openstack-hypervisor/main.tf:42-157` | openstack-hypervisor `amqp`, `identity-credentials`, `receive-ca-cert`, `certificates`, `ceilometer-service`, `nova-service`, `masakari-service`, `barbican-service` | Cross-model offers from the openstack model: rabbitmq, keystone, cert-distributor, CA, ceilometer, nova, masakari, barbican |
| same file, `:115` | `ceph-access` | cinder-volume-ceph |
| `cloud/etc/deploy-cinder-volume/main.tf:45-134` | cinder-volume `identity-credentials`, `amqp`, `database`, `receive-ca-cert` | keystone, rabbitmq, mysql and cert-distributor offers |
| same file | `cinder-volume-ceph` | cinder-volume (`cinder-volume`) and microceph (`ceph`) |
| `cloud/etc/deploy-microceph/main.tf:63-92` | microceph `identity-service`, `traefik-route-rgw`, `receive-ca-cert` | keystone-endpoints, traefik-rgw and cert-distributor offers |
| `cloud/etc/deploy-microovn/main.tf:145-282` | microovn `cluster` | microcluster-token-distributor |
| | `certificates` | CA offer |
| | `juju-info` | openstack-network-agents |
| | `role-assignment` | role-distributor |
| | `ovsdb` | sunbeam-ovn-proxy (amd64 and arm64 variants) |
| `cloud/etc/deploy-sunbeam-machine/main.tf:52` | epa-orchestrator | sunbeam-machine |
| `cloud/etc/deploy-storage/modules/*` | storage backend and cinder-volume | identity, amqp, database, ca and cinder-volume |
| `features/loadbalancer/etc/deploy-cni/main.tf:58,72` | openstack-port-cni | keystone (`identity-credentials`, `send-ca-cert`) |
| `features/observability/etc/deploy-grafana-agent/main.tf:42-99` | grafana-agent | principals (`cos-agent`, `juju-info`) and the COS offers (remote-write, logging, grafana-dashboard) |
| `features/observability/etc/deploy-cos/main.tf:140-429` | — | Relations inside the COS stack (traefik, prometheus, loki, grafana, alertmanager, catalogue) |
| `features/observability/etc/deploy-hardware-observer/main.tf:35,49` | hardware-observer | observability agent and principals |
| `features/pro/etc/deploy-pro/main.tf:35` | ubuntu-pro | sunbeam-machine (`juju-info`) |
| `features/instance_recovery/etc/deploy-consul-client/modules/consul-client/main.tf:38-67` | consul-client | principal (`juju-info`, `consul-notify`) and the consul-server offer |
| `features/shared_filesystem/etc/deploy-manila-data/main.tf:39-68` | manila-data | keystone, amqp and database offers |

### Snap config (`snapctl` / snaphelpers)

- **Defaults** are `daemon.group`, `daemon.debug`, `k8s.provider`, `deployment.risk` and `deployment.version` (`hooks.py:18-24`).
  - `daemon.*` is read by `snap-wrappers/commands/clusterd.start`.
  - `deployment.*` is read by `core/common.py:660,679`, and `k8s.provider` by `core/k8s.py:76`.
  - See §5.2 for the details.
- **Feature gates** use `feature.*` snap keys:
  - Snap → cluster DB: `hooks.py:92`, and the `SyncFeatureGatesToCluster` step (`steps/sync_feature_gates.py:12`).
  - Cluster DB → snap config: the Go watcher (`sunbeam-microcluster/sunbeam/feature_gate_sync.go`), started from `cmd/sunbeamd/main.go:91`.
- **Manifest selection:** `deployment.risk` and `deployment.version` choose which embedded manifest is used (`core/manifest.py:39`).

### CLIs and APIs called directly

- **Juju:** the CLI via `_juju_cmd` (add-user, grant, list-users, show-user, switch, models, …) and the `jubilant` library (`core/juju.py`, `steps/juju.py`).
- **Kubernetes:** `lightkube`, using the kubeconfig stored in clusterd as `K8SKubeConfig` (`core/k8s.py:38`, `steps/k8s.py:784`).
- **OpenStack API:** `openstack.connect` (`core/openstack_api.py`).
- **MAAS:** `python-libmaas` (`provider/maas/client.py`).
- **snapd REST socket:** `requests_unixsocket` (`hooks.py`, `commands/prepare_node.py`, `steps/upgrades/intra_channel.py`).
- **clusterctl:** `init` and `upgrade`, for the CAAS feature.
- **Local files:** `~/.local/share/openstack/deployments.yaml` (`core/deployments.py:19`) and the Juju data in `~/.local/share/juju`.

---

## 4A. `sunbeam cluster bootstrap`, step by step

Entry point: `bootstrap()` at `provider/local/commands.py:632`.

**Before the plans run:**
- Preflight checks (`:691-711`): `SystemRequirementsCheck`, `JujuSnapCheck`, `SshKeysConnectedCheck`, `DaemonGroupCheck` and `LocalShareCheck`, plus checks for the hypervisor hostname and the Juju/LXD controller.
- The deployment is added to `deployments.yaml` (`:714`).

The plans below run in order. "DB" means clusterd.

### cidr_plan (`:719`)

1. `AskManagementCidrStep` (`steps/clusterd.py:121`): prompt, or read the CIDR from the manifest. Changes nothing yet.

### plan (`:729-762`)

2. `SaveControllerStep` (`steps/juju.py:1904`): writes the controller and a Juju account (random admin password) into `deployments.yaml`.
3. `SwitchToController` (`steps/juju.py:1876`): `juju switch` to the bootstrap controller (default `localhost-localhost`).
4. `JujuLoginStep` (`steps/juju.py:1406`): logs the Juju client in.
5. `ClusterInitStep` (`steps/clusterd.py:62`): POSTs `/core/control` with `bootstrap: true`, which creates the dqlite cluster. It also adds the `nodes` row with roles and machine 0 (`clusterd/cluster.py:383`).
6. `SyncFeatureGatesToCluster` (`steps/sync_feature_gates.py:12`): copies `feature.*` snap config into the DB's `feature_gates` table.
7. `SaveManagementCidrStep` (`steps/clusterd.py:191`): stores `BootstrapAnswers.bootstrap.management_cidr` in the DB.
8. `AddManifestStep` (`core/manifest.py:589`): adds a `manifest` row with the user's manifest, or the embedded one if there's nothing in the DB yet.
9. `PromptForProxyStep` (`commands/proxy.py:255`): writes `ProxySettings` to the DB (`:332`).
10. `PromptDatabaseTopologyStep` (`steps/openstack.py:1565`): writes `Topology` (`:1611`).
11. `PromptRegionStep` (`steps/openstack.py:1281`): writes `Region` (`:1326`).
12. `ValidateIdentityManifest` (`steps/sso.py:998`): validation only.

**Between plans:**
- If `--region-controller-token` was given, `connect_to_region_controller` runs (`provider/common/multiregion.py`).
- The `deployments` and `deployment.type` keys are written to the DB (`:774-776`).

### Default path, no `--controller`: `deploy_and_migrate_juju_controller` (`:522`)

**plan1** (`get_juju_controller_plans`, `:340-376`), skipped if the controller was already migrated:
- `AddCloudJujuStep` (`steps/juju.py:66`) and `AddCredentialsJujuStep` (`steps/juju.py:151`): add the manual cloud and empty credentials to the LXD controller.
- `BackupBootstrapUserStep` (`steps/juju.py:1126`): backs up the local Juju account.
- `ClusterUpdateJujuControllerStep` (`steps/clusterd.py:604`): writes `JujuController` to the DB (`core/juju.py:249`).
- `SaveJujuAdminUserLocallyStep` (`steps/juju.py:1236`): writes the admin account locally.

**plan2** (`get_juju_model_machine_plans`, `:379`):
- `AddJujuModelStep` (`steps/juju.py:1491`): creates the `openstack-machines` model.
- `AddJujuMachineStep` (`steps/juju.py:938`): adds this host as a manual machine.

**plan3** (spaces, sunbeam-machine and k8s plans):
- `AddJujuSpaceStep` (`steps/juju.py:1650`) and `UpdateJujuModelConfigStep` (`steps/juju.py:1571`): create the management space and make it the default space.
- `TerraformInitStep` then `DeploySunbeamMachineApplicationStep` (`steps/sunbeam_machine.py:31`): applies `sunbeam-machine-plan`; tfvars stored as `TerraformVarsSunbeamMachine`.
- `TerraformInitStep` then `DeployK8SApplicationStep` (`steps/k8s.py:164`): applies `k8s-plan`; tfvars stored as `TerraformVarsK8S` and `TerraformVarsK8SAddons`.
- `StoreK8SKubeConfigStep` (`steps/k8s.py:728`): writes `K8SKubeConfig`.
- `EnsureK8SUnitsTaggedStep` (`steps/k8s.py:325`), `EnsureCiliumDeviceByHostStep` (`:1412`), `EnsureL2AdvertisementByHostStep` (`:1840`) and `EnsureDefaultL2AdvertisementMutedStep` (`:2075`): create k8s node labels and Cilium/L2 CRs.

**plan4** (`get_juju_bootstrap_plans`, `:480`):
- `AddK8SCloudInClientStep` (`steps/k8s.py:609`): adds the k8s cloud to the local Juju client.
- `BootstrapJujuStep` (`steps/juju.py:209`): bootstraps `sunbeam-controller` on the new k8s cluster, with `controller-service-type=loadbalancer` and `secret-backend=internal`.
- `PatchServiceExternalTrafficStep` (`steps/k8s.py:2172`): workaround for LP#2111922.

**plan5** (`get_juju_migrate_plans`, `:428`):
- `SwitchToController`, `AddCloudJujuStep`, `AddCredentialsJujuStep`.
- `MigrateModelStep` (`steps/juju.py:2111`): moves the machines model from the LXD controller to the k8s controller.
- `ClusterUpdateJujuControllerStep` and `SaveJujuAdminUserLocallyStep`.
- Then `set_juju_controller_migrated()` writes `juju_controller_migrated_to_k8s` to the DB.

**plan6 and plan7** (`get_juju_user_plans`, `:459`):
- `CreateJujuUserStep` (`steps/juju.py:424`): `juju add-user <fqdn>`, which yields a registration token.
- `ClusterAddJujuUserStep` (`steps/clusterd.py:509`): adds a `jujuuser` row.
- `JujuGrantModelAccessStep` (`steps/juju.py:542`), `BackupBootstrapUserStep`, `SaveJujuUserLocallyStep` (`:1171`) and `RegisterJujuUserStep` (`:643`): grant model access and switch the local client to the per-node user.

**After that:** `AddK8SCredentialStep` (`steps/k8s.py:681`) adds the k8s credential to the controller.

### With `--controller` (an external controller, `:778-801`)

This path is shorter: `get_juju_controller_plans(external_controller=True)`, then model and machine, then spaces plus sunbeam-machine and k8s, then `AddK8SCloudStep` (`steps/k8s.py:571`).

### plan1 (`:826-978`)

- If MicroOVN is needed (`core/ovn.py:42`): `DeployRoleDistributorApplicationStep` (`steps/role_distributor.py:99`) and `DeployMicroOVNApplicationStep` (`steps/microovn.py:116`). Their tfvars go to `TerraformVarsRoleDistributorPlan` and `TerraformVarsMicroovnPlan`.
- `DeployMicrocephApplicationStep` (`steps/microceph.py:97`) and `DeployCinderVolumeApplicationStep` (`steps/cinder_volume.py:113`). These are always deployed, regardless of role.
- `TerraformInitStep` for `openstack-plan`.
- **Storage role only:** `ConfigureMicrocephOSDStep` (`steps/microceph.py:445`) adds OSD disks and writes `TerraformVarsMicroceph`.
- **Control role (or region controller) only:**
  - `LocalEndpointsConfigurationStep` (`provider/local/steps.py:790`, based on `steps/openstack.py:1345`): writes `Endpoints`.
  - `DeployControlPlaneStep` (`steps/openstack.py:701`, `run` at `:896`): writes `Topology` and `OpenStackModelConfig`, then applies `openstack-plan` (tfvars in `TerraformVarsOpenstack`) and waits for the openstack model.
  - `AttachHorizonThemeStep` (`steps/horizon.py:42`): attaches the Horizon theme.
  - `SetKeystoneSAMLCertAndKeyStep` (`steps/sso.py:1343`) and `DeployIdentityProvidersStep` (`steps/sso.py:1205`): set up SSO and identity providers.
  - Re-runs of `DeployMicrocephApplicationStep` and `DeployCinderVolumeApplicationStep`, now that the openstack offer URLs (keystone, rgw, amqp, mysql) exist.
  - `ReapplyMicroOVNOptionalIntegrationsStep` (`steps/microovn.py:263`).

### plan2 (`:981-1005`)

- `OpenStackPatchLoadBalancerServicesIPStep` (`steps/openstack.py:1030`): pins LoadBalancer IPs.
- `DeployHypervisorApplicationStep` (`steps/hypervisor.py:78`): applies `hypervisor-plan` with the offer URLs; tfvars in `TerraformVarsHypervisor`. Skipped for region controllers.
- `SetBootstrapped` (`steps/bootstrap_state.py:12`): sets `sunbeam_bootstrapped = "True"` in the DB. This is the last step.

---

## 4B. Manifest: parsing, merging and tfvars

### Building the defaults

`Deployment.get_manifest` (`core/deployment.py:335`) works in stages:

- **Base:** `Manifest(features=feature_manager.get_all_feature_manifests())` (`feature_manager.py:314`). Each feature contributes `default_software_overrides()`. The core software comes from `_default_software_config()` (`core/manifest.py:170`), which reads charm channels from `MANIFEST_CHARM_VERSIONS` (`versions.py:93`) and plan sources from `TERRAFORM_DIR_NAMES` (`versions.py:99`).
- **Override,** the first one found among:
  1. the `--manifest` file
  2. the latest manifest in clusterd (`client.cluster.get_latest_manifest()`)
  3. the embedded `etc/manifests/<ver>/<risk>.yml`, used only when risk ≠ stable (`core/deployment.py:364-372`)

### Parsing

`Deployment.parse_manifest` (`core/deployment.py:320`) validates the override with `Manifest.model_validate`. Features get typed config through `parse_feature_manifest` (`:251`), and storage through `parse_storage_manifest` (`:306`). `Manifest.from_file` is at `core/manifest.py:458`.

### Merging

- `validate_against_default` (`core/manifest.py:484`) rejects unknown charms and plans (`SoftwareConfig.validate_against_default`, `:122`).
- `Manifest.merge` (`:463`) calls `CoreManifest.merge` (`:328`), then `SoftwareConfig.merge` (`:129`), which uses `utils.merge_dict` (`utils.py:294`). Feature merges go through `_AddonManifest.merge` (`:347`) and `FeatureGroupManifest.merge` (`:393`).
- Storage is replaced wholesale, not merged (`:466`).
- The result is cached in `Deployment._manifest`.
- Persisting to the DB is a separate job, done by `AddManifestStep` (`core/manifest.py:589`).

### Turning it into tfvars

- **The map:** `MANIFEST_ATTRIBUTES_TFVAR_MAP` (`versions.py:312`; the format is documented at `versions.py:112-160`), with each feature's `manifest_attributes_tfvar_map()` merged in at `core/deployment.py:399-403`. Each `TerraformHelper` gets its own plan's slice (`:426`).
- **The merge:** `TerraformHelper.update_tfvars` (`core/terraform.py:522`) builds the final values in four stages:
  1. DB values from `_load_and_filter_db_tfvars` (`:663`), keeping `_computed_keys` and anything the manifest can't derive.
  2. Manifest values from `_get_tfvars` (`:798`), which maps charm `channel`/`revision`/`config` to tfvar names.
  3. Step overrides from `extra_tfvars()` or a feature's `set_tfvars_on_enable()`. For charm-config dicts the manifest still wins (`_merge_manifest_into_override`, `:725`); for everything else the override wins (`_apply_tfvars`, `:699`).
  4. Save to the DB with `update_config(tfvar_config, …)` and write the tfvars file (`:570-574`).
- **Callers:** `update_tfvars_and_apply_tf` (`:642`) is used by `DeployMachineApplicationStep.run` (`core/steps.py:109`). A partial, per-charm variant is `update_partial_tfvars_and_apply_tf` (`:382`).
- **Docstring caveat:** the docstring at `:533` says "Manifest > Override > DB", but that's only accurate for charm-config keys. For plain keys, the override wins.

---

## 4C. What clusterd stores, and which calls touch it

Schema: `sunbeam-microcluster/database/schema.go`. Python API: `clusterd/cluster.py`.

| Table | Contents | Written by | Read by |
|---|---|---|---|
| `nodes` (`:28`): name, role, machine_id, … | Cluster members | `add_node_info` (`clusterd/cluster.py:99`) via `bootstrap` / `join_node` (`:383`, `:394`); `update_node_info` (`:135`) via `ClusterUpdateNodeStep`; `remove_node_info` (`:131`) | `list_nodes`, `get_node_info`, `list_nodes_by_role` (`:122`, `:127`, `:201`), used by `DeployMachineApplicationStep` (`core/steps.py:125`) |
| `config` (`:47`), a key/value table | JSON blobs. See the next table for the keys. | `update_config` (`clusterd/cluster.py:193`), wrapped by `update_config` (`core/common.py:477`) and `questions.write_answers` | `get_config` (`:189`), wrapped by `read_config` (`core/common.py:481`) and `questions.load_answers` |
| `jujuuser` (`:63`): username, token | Juju registration tokens | `add_juju_user`, `update_juju_user` (`clusterd/cluster.py:155`, `:160`) | `get_juju_user` / `list_juju_users` (`:179`, `:170`), used by `RegisterJujuUserStep` |
| `manifest` (`:80`): manifest_id, applied_date, data | Manifest history | `add_manifest` (`:244`) | `get_latest_manifest` (`:261`) and `list_manifests` / `get_manifest`, used by `Deployment.get_manifest` |
| `storage_backends` (`:138`): name, type, principal, model_uuid, config | Cinder backends | `add_storage_backend` / `update_storage_backend` / `delete_storage_backend` (`:299-340`) | `get_storage_backends` / `get_storage_backend` (`:289`, `:294`) |
| `feature_gates` (`:156`): gate_key, enabled | Gate on/off flags | `add_feature_gate` / `update_feature_gate` (`:351`, `:363`) | `get_feature_gates` / `get_feature_gate` (`:341`, `:346`); also the Go sync to snap config |

Main keys in the `config` table:

| Key | Holds | Defined at |
|---|---|---|
| `tfstate-<plan>`, `tflock-<plan>` | Terraform state and locks, written by Terraform itself over HTTP | `sunbeam-microcluster/sunbeam/terraform.go:15-16` |
| `TerraformVarsOpenstack` | openstack-plan tfvars | `steps/openstack.py:73` |
| `TerraformVarsK8S`, `TerraformVarsK8SAddons` | k8s-plan tfvars | `steps/k8s.py:101-102` |
| `TerraformVarsMicrocephPlan`, `TerraformVarsMicroceph` | microceph tfvars and disks | `steps/microceph.py:43-44` |
| `TerraformVarsMicroovnPlan`, `TerraformVarsMicroovn` | microovn tfvars | `steps/microovn.py:38-39` |
| `TerraformVarsHypervisor` | hypervisor-plan tfvars | `steps/hypervisor.py:65` |
| `TerraformVarsCinderVolumePlan` | cinder-volume-plan tfvars | `steps/cinder_volume.py:50` |
| `TerraformVarsSunbeamMachine` | sunbeam-machine-plan tfvars | `steps/sunbeam_machine.py:21` |
| `TerraformVarsRoleDistributorPlan` | role-distributor tfvars | `steps/role_distributor.py:33` |
| `BootstrapAnswers` | Bootstrap answers, including the management CIDR | `steps/juju.py:58` |
| `ProxySettings` | Proxy configuration | `core/deployment.py:56` |
| `deployment.type` | Deployment type | `core/deployment.py:57` |
| `deployments` | Deployment metadata | `provider/local/commands.py:209` |
| `Region`, `Endpoints` | Region name and API endpoints | `core/openstack.py:11,13` |
| `Topology`, `OpenStackModelConfig`, `DatabaseMemory`, `DatabaseStorage`, `RabbitmqStorage` | Control-plane sizing and topology | `steps/openstack.py:74-77,358` |
| `JujuController` | Juju controller details | `core/juju.py:41` |
| `K8SKubeConfig` | Kubeconfig for the k8s cluster | `core/k8s.py:38` |
| `Feature-<name>` | Feature state `{enabled, version}` | `features/interface/v1/base.py:351` |
| `sunbeam_bootstrapped`, `juju_controller_migrated_to_k8s` | Flags | `clusterd/cluster.py:377,381` |
| `StorageBackendsEnabled` | Enabled storage backends | `storage/base.py:107` |
| SSO keys (`SSOFeatureConfigKey`, …) and feature-specific keys | e.g. loadbalancer, observability, vault, tls | `steps/sso.py:48-49`, `features/loadbalancer/feature.py:102-129` |

---

## 4D. Enabling one feature end to end: `sunbeam enable dns ns1.example.com.`

1. **Discovery.** `FeatureManager._load_features` imports `features/dns/feature.py` (`feature_manager.py:251-268`). `DnsFeature` registers itself in `_FEATURES` through `__init_subclass__` (`features/interface/v1/base.py:331`).
2. **CLI registration.** `FeatureManager.register` (`feature_manager.py:371`) skips any feature that's gated, checks `is_enabled`, then calls `BaseRegisterable.register` (`features/interface/v1/base.py:205`). That attaches `enable_cmd` under `enable` and `disable_cmd` under `disable` (`:906-925`). If DNS is already enabled, it also adds `sunbeam dns address` (`features/dns/feature.py:287`).
3. **Command.** `enable_cmd` (`features/dns/feature.py:206`) builds `DnsFeatureConfig`, whose validator requires each nameserver to end in a dot (`:48-60`). It then calls `enable_feature` (`features/interface/v1/base.py:859`), which picks up `sunbeam enable -m <manifest>` from the parent click context if given.
4. **Pre-enable.**
   - `OpenStackControlPlaneFeature.pre_enable` (`features/interface/v1/openstack.py:167`) runs `VerifyBootstrappedCheck`.
   - `EnableDisableFeature.pre_enable` (`features/interface/v1/base.py:835`) runs `JujuLoginCheck`, then `check_enablement_requirements` (`:726`) and `enable_requirements` (`:767`), which auto-enables dependencies where possible.
5. **`run_enable_plans`** (`features/dns/feature.py:120`):
   - `TerraformInitStep` and `ValidateInfraAppsStep` (`steps/openstack.py:185`) refuse to continue if reapplying the openstack plan would change mysql, vault or traefik.
   - `AddManifestStep`, only if the user supplied a manifest.
   - `EnableOpenStackApplicationStep` (`features/interface/v1/openstack.py:607`, `run` at `:669`):
     - Overrides are `set_tfvars_on_enable`, which returns `{"enable-designate": True, "nameservers": …}` (`features/dns/feature.py:177`), plus `get_database_tfvars`.
     - It first does a dry `update_tfvars(persist=False)` for a drift check (`_infra_app_drift_result`), then the real `update_tfvars` into `TerraformVarsOpenstack`. DNS sets `tf_plan_location = SUNBEAM_TERRAFORM_REPO`, so it reuses `openstack-plan` (`features/interface/v1/openstack.py:122-127`, `:272`).
     - The charm channel and revision map for `designate-k8s` and `designate-bind-k8s` comes from `manifest_attributes_tfvar_map` (`features/dns/feature.py:101`), with default channels from `default_software_overrides` (`:92`).
     - It runs `tfhelper.apply()`, then waits for `bind`, `designate` and `designate-mysql-router` to go active. `designate-mysql` is added when the topology is multi (`:167`).
   - On MAAS only: `PatchBindLoadBalancerIPPoolStep`. Then `PatchBindLoadBalancerIPStep` (`:63-80`, `:155-162`) pins the bind LoadBalancer IP.
6. **Record the state.** `update_feature_info(..., {"enabled": "true"})` writes `Feature-dns = {enabled, version}` to clusterd (`features/interface/v1/base.py:879`, `:430`).

Disabling runs the same path in reverse: `disable_cmd` calls `disable_feature` (`base.py:898`), then `DisableOpenStackApplicationStep` (`openstack.py:725`) with `{"enable-designate": False}`.

---

## 4E. `cluster add` vs `cluster join`, and the join token

### `sunbeam cluster add <fqdn>`

Runs on an existing member (`provider/local/commands.py:1222`). It only issues credentials and deploys nothing.

1. Preflight: `DaemonGroupCheck` and `VerifyFQDNCheck`.
2. `JujuLoginStep`.
3. `ClusterAddNodeStep` (`steps/clusterd.py:231`) calls `add_node`, then `generate_token`, which POSTs `/core/control/tokens` (`clusterd/cluster.py:390`, `:71`).
   - The token is generated by the microcluster library, not by code in this repo. It's a base64-encoded JSON object that includes `name` and `join_addresses`.
   - If an unused token already exists for that name, the step reuses it (`steps/clusterd.py:261-266`).
4. `CreateJujuUserStep` runs `juju add-user <fqdn>` and pulls the registration string out of `juju register …` (`steps/juju.py:424`).
5. `JujuGrantModelAccessStep` grants access to every model (`provider/local/commands.py:1250-1256`).
6. The Juju token is stored in clusterd with `ClusterAddJujuUserStep`. If the node is being re-added, `ResetJujuUserStep` and `ClusterUpdateJujuUserStep` run instead (`:1265-1285`).
7. The clusterd token is printed or written to a file (`:1287-1300`, `_print_output` / `_write_to_file` at `:1179`, `:1189`).

### `sunbeam cluster join <token>`

Runs on the new node (`provider/local/commands.py:1405`). `-` reads the token from stdin. This command does the actual deployment.

1. `TokenCheck` (`core/checks.py:532`) checks that the token is base64 JSON, that `name` matches the hostname, and that `join_addresses` isn't empty.
2. `get_local_cidr_matching_token` (`utils.py:307`) decodes the token and picks the local IP on the same network as `join_addresses`.
3. `ClusterJoinNodeStep` (`steps/clusterd.py:280`) calls `join_node`. That POSTs `/core/control` with `join_token`, and microcluster validates the token and joins dqlite. It then calls `add_node_info` (`clusterd/cluster.py:394`).
4. Now the node can read shared state: it reads `deployments` from the DB to learn the deployment name (`:1477`).
5. `CheckJujuReachableStep`, `JujuLoginStep`, `SaveJujuUserLocallyStep`, then `RegisterJujuUserStep` (`steps/juju.py:643`), which fetches the Juju token from the `jujuuser` table and runs `juju register`.
6. Optionally, `connect_to_region_controller` for multi-region.
7. `AddJujuMachineStep` adds the machine, then the INFRA_APPS gate runs (`ValidateInfraAppsStep`).
8. plan4 (`:1549-1827`):
   - `ClusterUpdateNodeStep` stores the machine id.
   - The relevant plans are re-applied for the node's roles: sunbeam-machine (always), k8s plus labels/L2 (control), role-distributor and MicroOVN (network), MicroCeph plus OSDs, cinder-volume and the control plane on the first storage node (storage), and hypervisor plus SR-IOV/DPDK (compute).
   - Each `DeployMachineApplicationStep` picks the new node up through `list_nodes_by_role`.

### Two tokens

| Token | Produced by | Carried by | Consumed by |
|---|---|---|---|
| Microcluster join token | `ClusterAddNodeStep`, which calls `/core/control/tokens` | The user, copied by hand | `ClusterJoinNodeStep`, which calls `/core/control` (validated by dqlite/microcluster) |
| Juju registration token | `CreateJujuUserStep` (`juju add-user`) | The clusterd `jujuuser` table | `RegisterJujuUserStep` (`juju register`) |

Both are tied to the node's FQDN.

---

## 5. Background concepts and follow-up notes

### 5.1 Snap base and confinement (`snap/snapcraft.yaml:2`, `:10`)

**`base: core24`**
- The binaries inside the snap run on an Ubuntu 24.04 (Noble) runtime filesystem, which is shipped as a separate snap. They don't use the host's libraries.
- So `sunbeam`, `sunbeamd` and `terraform` link against Noble's libc on any host, and the bundled Python is 3.12.
- The base also sets which snapcraft syntax and rules apply (for example `layout:`).

**`confinement: strict`**
- Every app runs in a sandbox (AppArmor, seccomp, mount namespace).
- Without extra permissions an app can only read `$SNAP` (read-only) and write to `$SNAP_DATA`, `$SNAP_COMMON` and `$SNAP_USER_*`.
- Anything else needs a **plug**:
  - Per-app plugs are at `:24-64`.
    - `clusterd` only has `network` and `network-bind`.
    - `sunbeam` has `home`, `ssh-keys` and the personal-files/system-files plugs.
  - Custom plugs are at `:276-300`:
    - `personal-files`: `~/.local/share/juju`, `~/.local/share/openstack`, `~/.config/openstack`
    - `system-files`: `/etc/openstack`, read-only
    - `juju-bin`, a **content** interface. The Juju snap mounts its binaries at `$SNAP/juju`, which is why `PATH` gets `$SNAP/juju/bin` added (`:53`, `:64`).
  - `layout:` at `:20` maps `/usr/share/terraform/plugins` to the writable `$SNAP_DATA/terraform-plugins`.

**What this means in practice**
- Sensitive plugs only auto-connect if the store grants it. Otherwise they need `snap connect`. The code checks for this:
  - `core/checks.py:259` checks that `ssh-keys` is connected.
  - `commands/prepare_node.py:106` runs the connect command.
- Local `--dangerous` installs get no auto-connections at all. `docs/CONTRIBUTING.md:64-71` and `.github/workflows/build-snap.yml:71` connect the plugs by hand.
- "Permission denied" inside the snap usually means a missing plug. Check `snap connections openstack` and look for AppArmor `DENIED` lines in `journalctl`.
- For comparison: `classic` confinement has no sandbox and needs store review; `devmode` logs violations instead of blocking them.

### 5.2 How `hooks.py` reaches the snap

`snapcraft.yaml` never names `hooks.py`. The link is made through snap-helpers when the snap is built:

1. `sunbeam-python/pyproject.toml:124-127` declares entry points in the `snaphelpers.hooks` group:
   - `configure` → `sunbeam.hooks:configure`
   - `install` → `sunbeam.hooks:install`
   - `post-refresh` → `sunbeam.hooks:upgrade`
2. `snapcraft.yaml:258` runs `snap-helpers write-hooks` in the `override-prime` of the `sunbeam-and-openstackclients` part. That writes stub scripts into `meta/hooks/` in the primed snap, and each stub calls the matching Python function with a `snaphelpers.Snap`.
3. snapd runs `meta/hooks/<name>` when the event happens. There's no `snap/hooks/` directory in the repo because the hooks are generated.

| Hook | When it runs | Function | What it does |
|---|---|---|---|
| `install` | First install | `hooks.py:213` | Sets `DEFAULT_CONFIG` |
| `configure` | After install and refresh, and on every `snap set openstack ...` | `hooks.py:242` | 1. Fills in missing defaults (`:29`).<br>2. Validates feature-gate dependencies; if this fails, snapd rolls back the `snap set`.<br>3. Syncs `feature.*` to clusterd (`:194`), or skips this if clusterd isn't up yet.<br>4. Writes `$SNAP_DATA/config.yaml` and restarts `clusterd` if `daemon.*` changed. |
| `post-refresh` | After a refresh | `hooks.py:230` | Only sets up logging (a placeholder) |

- Hooks log to `$SNAP_COMMON/hooks.log`.
- If you rename or add a hook function, update the entry points too, or no stub is generated.

### 5.3 The snapcraft lifecycle and "prime"

Each part goes through these steps: **pull → build → stage → prime → pack**.

| Step | Result |
|---|---|
| pull | Source in `parts/<part>/src` |
| build | Plugin output in `parts/<part>/install` |
| stage | All parts merged into `stage/`, so later parts can use it (`after:`) |
| prime | Only what ships, in `prime/`. This is exactly what appears at `$SNAP` after install. |
| pack | `prime/` squashed into the `.snap` file |

- `override-<step>` replaces a step's default behaviour. `craftctl default` runs the default first.
- In `override-prime` the current directory is the final snap contents. So the `sunbeam-and-openstackclients` part does three things there (`:255-268`), because each needs the finished Python environment:
  - writes the snap hooks
  - precompiles `.pyc` files, since `$SNAP` is read-only at runtime
  - generates the bash completions
- The `manifests` part's `override-prime` (`:222-230`) symlinks the 2026.1 manifests into `etc/manifests/`.
- To look at the result: run `snapcraft prime` and inspect `prime/`, or run `unsquashfs -l` on a built snap.

### 5.4 What "manifest" means here

It means the **Sunbeam deployment manifest**: YAML with `core.software.charms` (channel, revision, config), optional `core.software.terraform` plan sources, `core.config` (deployment answers), `features` and `storage`. See §4B.

**Embedded manifests**
- They're stored as `manifests/<version>/{edge,beta,candidate,stable}.yml`, for 2024.1, 2025.1 and 2026.1.
- Each one pins charm channels for that risk. For example, `2026.1/beta.yml` uses `2026.1/beta`.
- They ship at `$SNAP/etc/manifests/`.
- `embedded_manifest_path` (`core/manifest.py:39`) picks one from the `deployment.version` and `deployment.risk` snap config.

**Which manifest is used** (`core/deployment.py:346-396`)
- Precedence: `--manifest` file > latest in clusterd > embedded (only for risk ≠ stable).
- After bootstrap stores a manifest in clusterd, editing the embedded files has no effect on that cluster.

**Not to be confused with**
- snapcraft's build manifest (`snap/manifest.yaml`)
- Kubernetes manifests (e.g. the ORC CRDs in the `cluster-api-resources` part, `snapcraft.yaml:206`)

### 5.5 Daemons, and what clusterd is

**Daemons**
- A daemon is a long-running background process managed by systemd.
- `daemon: simple` on the `clusterd` app (`snapcraft.yaml:34-40`) makes snapd create `snap.openstack.clusterd.service`. It starts at install and at boot, and `restart-condition: on-failure` restarts it if it crashes.
- `sunbeam`, `openstack` and `terraform` are ordinary commands that run and exit.
- Useful commands:
  - `snap services openstack`
  - `sudo snap restart openstack.clusterd`
  - `snap logs -n 100 openstack.clusterd` (or `journalctl -u snap.openstack.clusterd`)

**clusterd** is a replicated database with a REST API. Each node runs a copy. It exists because the CLI is short-lived, and the deployment state has to persist and be shared between nodes.

| Layer | What it does | Where |
|---|---|---|
| dqlite | SQLite with Raft replication | `dqlite` part, `snapcraft.yaml:81` |
| microcluster | Membership, join tokens, mTLS between members, schema migrations, REST server | `sunbeam-microcluster/go.mod` (v2.2.0) |
| sunbeamd | Sunbeam's tables and `/1.0/*` endpoints (§3, §4C) | `sunbeam-microcluster/` |

`sunbeamd` is the process. `clusterd` is the name of the snap app and service.

**Access**
- Locally, over the unix socket `$SNAP_COMMON/state/control.socket` (`clusterd/client.py:109-114`). The socket group is `daemon.group`, which defaults to `snap_daemon`.
- Between members, over HTTPS on port 7000 (`steps/clusterd.py:44`) with mTLS.

**Lifecycle**
- After install the daemon is running but there's no cluster yet. Python calls raise `ClusterServiceUnavailableException`.
- Bootstrap creates a one-node cluster. Join adds a member, which then receives the replicated data.
- Raft needs a majority of voters, so it takes three or more members to survive losing a node. With two nodes, losing either one makes the database unavailable.

### 5.6 Juju relations

**Terms**
- **controller**: the Juju server that manages models.
- **model**: a workspace on one substrate. Sunbeam has a machine model and a k8s model.
- **charm**: the operator package that knows how to run one piece of software.
- **application**: a deployed charm.
- **unit**: one instance of an application.

**How relations work**
- A **relation**, which newer Juju calls an *integration* (`juju integrate`), links two applications' **endpoints**.
- Each charm declares its endpoints as `provides` or `requires`, each with an **interface** name (for example `keystone` or `rabbitmq`). Only endpoints with matching interfaces can connect.
- The two charms exchange settings through **relation data bags**. Each change fires `relation-joined`, `relation-changed` or `relation-broken` hooks.
- Example: nova asks for a service user over `identity-service`, keystone writes back credentials and a URL, and nova then renders its config.
- **Cross-model relations**: one side publishes an **offer** and the other side consumes it by **offer URL**. Sunbeam uses these between the machine model and the k8s model.

**Where Sunbeam creates relations**
1. **Terraform** `juju_integration` resources (table in §3). Example: `cloud/etc/deploy-openstack-hypervisor/main.tf:42` relates `openstack-hypervisor:amqp` to `var.rabbitmq-offer-url`. Most control-plane relations are in the external `sunbeam-terraform` plan, which `snapcraft.yaml:161` copies to `etc/deploy-openstack/`.
2. **Python** connects the plans. It reads one plan's Terraform outputs (the offer URLs) and passes them as tfvars to the next plan, e.g. `steps/microceph.py:194-215` and `steps/hypervisor.py:112`.
3. **Direct calls**: `JujuHelper.integrate` (`core/juju.py:367`, `:2179`), `remove_relation` (`:2196`), `consume_offer` (`:1865`), used for example at `steps/juju.py:1763`.

**Debugging**
- `juju status --relations -m <model>`
- `juju show-unit <unit>` shows the data bags
- `juju offers`
- `juju debug-log --include <app>`

### 5.7 How one step's output reaches the next

`run_plan` (`core/common.py:317-372`) gives each step only a `StepContext` (`:243`), which carries the spinner and a reporter. Steps never see earlier steps' `Result`s during a run.
- A `Result` (`:185`) is a `result_type` plus a free-form `message` (default `""`).
- The results dict, keyed by step class name, goes back to the **command** after the whole plan finishes.

Data moves between steps in four ways:

1. **Between plans, through constructors.** The command runs a plan, reads a value with `get_step_message` or `get_step_result` (`core/common.py:375-385`), and passes it into the constructors of the next plan's steps. Constructors run when the plan list is built, so a value produced by an earlier step can only reach a later step through a new plan. That's why commands are split into plan1, plan2, and so on. Examples in `provider/local/commands.py`:
   - management CIDR, `:719-721`
   - join's `AddJujuMachineStep` machine ID → `ClusterUpdateNodeStep(machine_id=...)`, `:1515-1550`
   - add's join and Juju tokens, `:1258-1282`
   - `CreateJujuUserStep` token, `:589`, `:1356`
2. **Within a plan, through shared external state.** These reads happen in `run()` or `is_skip()`, never in `__init__`.
   - **clusterd**: in join's plan4, `ClusterUpdateNodeStep` writes the machine ID, and `DeployMachineApplicationStep.run` reads it back through `list_nodes_by_role` (`core/steps.py:119-137`). Config keys work the same way.
   - **Terraform outputs**, stored as tfstate in clusterd: the hypervisor, microceph, microovn and cinder-volume steps call `openstack_tfhelper.output()` to get offer URLs (`steps/hypervisor.py:112`, `steps/microceph.py:194`, `steps/microovn.py:155`, `steps/cinder_volume.py:87`). The comment at `steps/hypervisor.py:115-121` explains why offer URLs are passed as tfvars (LP#2085310).
   - **Juju state**: model UUIDs, unit status.
   - **Shared Python objects** passed into many steps (`deployment`, `client`, `jhelper`, `manifest`, `tfhelper`). For example, `Deployment._manifest` is a cache.
3. **Within one step.** `prompt()` and `is_skip()` store attributes that `run()` uses later.

To pass a new value: within the same plan, write it to clusterd in step A and read it in step B's `run()`. To reach a later plan, return it as `Result.message` and pass it into the next constructor.

### 5.8 Gotcha: several steps of the same class in one plan

`run_plan` keys results by `step.__class__.__name__` (`core/common.py:348` for skips, `:360` for runs), so a later instance overwrites the earlier one's entry.
- **It's common.** Join's plan4 contains `TerraformInitStep` eight times (`provider/local/commands.py:1549` onward).
- **It's harmless today**, for three reasons:
  - Failures raise `ClickException` immediately with the failing step's own message (`:357`, `:369`).
  - Nobody reads the duplicated entries. Join's plan4 result isn't even kept (`:1827`).
  - Steps whose results are read get their own small plan (`cidr_plan` at `:719`, join's plan3 at `:1515`).
- That was checked for the callers in `provider/local/commands.py`, not for the ones in `provider/maas/commands.py` or `features/maintenance/`.
- **The risk:** if you add a second instance of a step whose result is read, `get_step_message` silently returns the last instance. Keep any step you need to read the result of unique within its plan, or give it a plan of its own.
- **A possible fix:** key results by index or `step.name`, or store a list per class. This changes the API of `get_step_message` and `get_step_result` and all their callers.
