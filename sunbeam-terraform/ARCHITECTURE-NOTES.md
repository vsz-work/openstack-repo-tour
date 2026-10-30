# sunbeam-terraform — architecture notes

Onboarding notes from reading `sunbeam-terraform` (`main` @ `2a2a076`) against
`snap-openstack` (`main` @ `b659a406`). Written 2026-10-29.

**Path convention**

- Bare paths (`main.tf:181`, `modules/…`) are in this repo (`~/dev/sunbeam-terraform`).
- `sunbeam/…` is `~/dev/snap-openstack/sunbeam-python/sunbeam/…`.
- `snap/…` and `cloud/…` are `~/dev/snap-openstack/…`.
- "module `:N`" means `modules/openstack-api/main.tf:N`.

## Contents

1. [Repo map](#1-repo-map)
2. [Core responsibilities and abstractions](#2-core-responsibilities-and-abstractions)
   - [Juju provider resources](#juju-provider-resources)
   - [The openstack-api module](#the-openstack-api-module)
   - [Variables](#variables)
   - [Database topology](#database-topology)
   - [Ingress](#ingress)
3. [Integration points](#3-integration-points)
4. [Q&A](#4-qa)
   - [A. Integration graph: keystone, nova, neutron, glance](#a-integration-graph-keystone-nova-neutron-glance)
   - [B. Variables snap-openstack sets](#b-variables-snap-openstack-sets)
   - [C. Conditional resources](#c-conditional-resources)
   - [D. Source of truth and the 2024.1 branch](#d-source-of-truth-and-the-20241-branch)
5. [Issues worth raising](#5-issues-worth-raising)

---

## 1. Repo map

| Path | What it is |
|---|---|
| `main.tf` (2858 lines) | The root plan: provider, locals, the Juju model, every app, relation and offer |
| `variables.tf` (1254) | About 190 inputs |
| `outputs.tf` (96) | Offer URLs only — the contract the machine-side plans consume |
| `modules/openstack-api/` | Reusable "one OpenStack API service" module |
| `modules/mysql/` | `mysql-k8s` plus its observability relations |
| `modules/rabbitmq/` | `rabbitmq-k8s` plus the `amqp` offer |
| `modules/consul/` | `consul-k8s` plus the `consul-cluster` offer |
| `channels/edge.tfvars` | Channel overrides used by CI |
| `Makefile:18-27` | `init`, `check` (`terraform fmt -check` + `validate`), `clean` |
| `.github/workflows/` | `pr.yaml` and `main.yml` both call `check.yml` and `deploy.yml` |

**Entry point.** A single root module. snap-openstack ships it as `etc/deploy-openstack`
(`snap/snapcraft.yaml:161-186`) and runs it through `TerraformHelper` (`sunbeam/core/terraform.py`).

**Provider.** `juju/juju = 2.3.1`, pinned in the root (`main.tf:18-25`) and in every module.

**Tests.** There are no unit tests. Testing is `terraform validate` plus a live deploy in
`deploy.yml`:

| Step | Lines |
|---|---|
| concierge sets up Juju 3.6 + LXD + Canonical K8s | `deploy.yml:31-63` |
| Machine model with MicroOVN; offer `sunbeam-ovn-proxy:ovsdb-cms` | `deploy.yml:68-79` |
| `terraform apply` with most features on and `channels/edge.tfvars` | `deploy.yml:82-85` |
| Relate `microovn:certificates` to `terraform output ca-offer-url` | `deploy.yml:86` |
| `juju-wait` on both models | `deploy.yml:87-88` |

> The README still says MicroK8s; CI and the snap use Canonical K8s.

---

## 2. Core responsibilities and abstractions

The plan:

- creates the k8s-side model (`juju_model.sunbeam`, `main.tf:90-100`);
- deploys the OpenStack control plane, its infra charms (MySQL, RabbitMQ, three Traefiks,
  CA, Vault, OTel collectors) and every relation between them;
- publishes offers that the machine-side plans consume.

### Juju provider resources

| Resource | Use |
|---|---|
| `juju_model` | The one k8s model (`main.tf:90`) |
| `juju_application` | Charm block sets `name`/`channel`/`revision`/`base`; resource sets `config`, `storage_directives`, `units`, `trust` |
| `juju_integration` | Two `application { name, endpoint }` blocks, or one with `offer_url` for a cross-model relation |
| `juju_offer` | Offers out of the k8s model |
| `data "juju_offer"` | Only for MicroCeph (`main.tf:75-88`) |

### The openstack-api module

`modules/openstack-api/main.tf` bundles a service with its database router and optional relations.

**Always created**

- The charm (`:27-44`).
- Its own `<name>-mysql-router` (`:46-59`).
- router → mysql (`:61-73`) and service → router (`:75-87`).

**Optional — created only when the input is non-empty**

| Input | Relation | Line |
|---|---|---|
| `rabbitmq` | `amqp` | `:90` |
| `keystone` | `identity-service` | `:106` |
| `keystone-credentials` | `identity-credentials` | `:136` |
| `keystone-ops` | `identity-ops` | `:167` |
| `keystone-endpoints` | `identity-endpoints` | `:197` |
| `keystone-cacerts` | `send-ca-cert` → `receive-ca-cert` | `:212` |
| `ingress-public` / `ingress-internal` | `traefik:ingress` → `svc:ingress-*` | `:244-273` |
| `logging-app`, `mysql-router-*-app` | logs, metrics, dashboards | `:368-516` |

Each identity relation has an `external-*-offer-url` twin for multi-region
(`:121`, `:152`, `:182`, `:227`).

**Service-specific code inside the generic module**

- `name == "nova"` adds `nova-api-mysql-router` and `nova-cell-mysql-router` (`:276-366`)
  and the `nova-service` offer (`:554`).
- `name == "keystone"` creates four offers (`:522-552`), split per endpoint as a workaround
  for provider issue #787.

`ingress-internal` and `ingress-public` have no default (`modules/openstack-api/variables.tf:120-128`),
so callers pass `""` to turn them off.

**Locals that drive topology** (`main.tf:29-73`)

| Local | Effect |
|---|---|
| `mysql-services`, `mysql` | Which MySQL each service uses |
| `keystone-service-name` | `""` on a secondary region |
| `standard-*-traefik-name` | `""` on a region controller |
| `controller-*-traefik-name` | `""` on a secondary region |

**Infra apps declared directly in `main.tf`**

| App | Line |
|---|---|
| `traefik`, `traefik-public`, `traefik-rgw` | `:374`, `:436`, `:501` |
| `certificate-authority` | `:574` |
| `vault`, `bind` | `:1426`, `:1332` |
| `ceilometer`, `openstack-exporter` | `:967`, `:1096` |
| `opentelemetry-collector`, `-infra` | `:2324`, `:2389` |
| `tempest`, `images-sync` | `:2217`, `:2452` |
| `keystone-ldap-*` | `:2095` |
| SSO IdPs (openid, saml2) | `:2800`, `:2830` |
| `manual-tls-certificates` | `:2141` |

### Variables

| Kind | Pattern | Default |
|---|---|---|
| Feature flags | 25 × `enable-*`, plus `many-mysql`, `is-region-controller`, `is-secondary-region` | `false` |
| Release channel | `openstack-channel` | `"2026.1/stable"` (`variables.tf:16`) |
| Per-service channel/revision | `<svc>-channel`, `<svc>-revision` | `null`; a null channel falls back to `openstack-channel` (e.g. `main.tf:154`) |
| Per-service config/storage | `<svc>-config`, `<svc>-storage` | `{}`; merged with computed keys like `region`, `enable-telemetry-notifications` (e.g. `main.tf:173-177`) |
| Infra channels | mysql `8.0/stable`, mysql-router `8.0/stable`, traefik `latest/stable`, rabbitmq `3.12/stable`, vault `1.18/stable`, bind `9/stable`, otel `2/stable`, CA `1/stable` | — |
| Bases | `base` `ubuntu@24.04`; mysql + mysql-router `ubuntu@22.04`; traefik `ubuntu@26.04` | — |
| Scale | `ha-scale`, `os-api-scale`, `ingress-scale` | `1` |
| Offer URLs in | `ceph-*-offer-url`, `cinder-volume-offer-urls`, COS URLs, `external-*` | MicroCeph `admin/controller.*`; `external-ovsdb-cms-offer-url` is the only required input (`variables.tf:1201`) |

> The Terraform defaults are not what the snap deploys. The snap overrides them, e.g.
> RabbitMQ `4.0/stable` and Vault `2.0/stable` (`sunbeam/versions.py:26`, `:35`).

### Database topology

**Single vs many**

| Mode | Resource | What you get |
|---|---|---|
| `many-mysql = false` (default) | `module.single-mysql` (`main.tf:102-116`) | One `mysql` |
| `many-mysql = true` | `module.many-mysql`, `for_each` over non-null `local.mysql-services` (`main.tf:118-132`) | One `<svc>-mysql` per service |

- In many-mysql mode, feature services get a DB only when their flag is on (`main.tf:38-49`).
- Per-service config and storage come from `mysql-config-map` / `mysql-storage-map` (`main.tf:126-127`).

**Routers**

- Always one router per app, created inside the module and pointed at `local.mysql[svc]`.
- Shared databases:
  - nova, nova-ironic and nova-ironic shards → nova's DB (`main.tf:1584`, `:1729`)
  - ironic-conductor → ironic's DB (`main.tf:1612`)
  - manila-cephfs → manila's DB (`main.tf:1961`)
- Two standalone routers, exposed with `expose-external=loadbalancer` and offered to machine charms:
  - `cinder-volume-mysql-router` (`main.tf:715-755`)
  - `manila-data-mysql-router` (`main.tf:2008-2048`)

**Snap side**

- Topology defaults to `"single"` (`sunbeam/steps/openstack.py:78`).
- Chosen by a prompt that says it "cannot be changed later" (`sunbeam/steps/openstack.py:1551-1560`).
- `--database` is deprecated and ignored (`sunbeam/provider/local/commands.py:654-660`).
- Memory and connection sizing is computed per service (`sunbeam/steps/openstack.py:513-554`).

### Ingress

**Three Traefik apps**

| App | Purpose | Line |
|---|---|---|
| `traefik` | Internal endpoints | `main.tf:374` |
| `traefik-public` | Public endpoints | `main.tf:436` |
| `traefik-rgw` | Ceph RGW; only with `enable-ceph`; offered as `traefik-route` to MicroCeph | `main.tf:501-527` |

**Two relation styles**

- **`ingress`** — most services, via the module: `traefik:ingress` ↔ `svc:ingress-internal` / `ingress-public`.
- **`traefik-route`** — the service owns its routing:
  - nova (`main.tf:240-268`), plus `ingress-metadata` on internal Traefik (`main.tf:270`)
  - heat (`main.tf:862-890`)
  - ironic, nova-ironic, nova-ironic shards (`main.tf:1629`, `:1644`, `:1761`)

**TLS.** Controlled by `traefik-to-tls-provider` plus `enable-tls-for-{public,internal,rgw}-endpoint`
(`main.tf:2172-2215`).

---

## 3. Integration points

### Inputs

| Other side | How | Where |
|---|---|---|
| Juju controller | `provider "juju" {}`, ambient client credentials | `main.tf:27` |
| K8s cloud + credential | `cloud`, `credential` vars | `main.tf:93-98`; set by `sunbeam/steps/openstack.py:938-941` |
| MicroCeph `ceph` | data offer | `main.tf:654`, `:955` |
| MicroCeph `ceph-nfs` | data offer | `main.tf:1993` |
| MicroCeph `ceph-rgw` | data offer | `main.tf:1690`, `:1705`, `:1804` |
| cinder-volume `storage-backend` | `offer_url` | `main.tf:808-821` |
| sunbeam-ovn-proxy `ovsdb-cms` | `offer_url` | `main.tf:606`, `:1252` |
| COS (remote-write, Loki, Grafana) | `offer_url` | `main.tf:2345-2450` |
| External keystone (multi-region) | `offer_url` | module `:121-240`; `main.tf:1013`, `:1127`, `:2248`, `:2483`, … |
| Charmhub | channels and revisions | every `juju_application` |

### Offers produced

| Output (`outputs.tf`) | Endpoint | Consumer in snap-openstack |
|---|---|---|
| `ca-offer-url` (`main.tf:823`) | `certificates` | microovn (`sunbeam/steps/microovn.py:157`), hypervisor (`sunbeam/steps/hypervisor.py:126`) |
| `keystone-offer-url` | `identity-credentials` | hypervisor (`sunbeam/steps/hypervisor.py:124`), cinder-volume (`sunbeam/steps/cinder_volume.py:90`), manila-data (`sunbeam/features/shared_filesystem/manila_data.py:37`) |
| `keystone-endpoints-offer-url` | `identity-service` | microceph RGW (`sunbeam/steps/microceph.py:204`) |
| `keystone-ops-offer-url` | `identity-ops` | **none** |
| `cert-distributor-offer-url` | `send-ca-cert` | hypervisor, cinder-volume (`sunbeam/steps/cinder_volume.py:106`), microceph (`sunbeam/steps/microceph.py:207`) |
| `rabbitmq-offer-url` | `amqp` | hypervisor, cinder-volume (`sunbeam/steps/cinder_volume.py:94`), manila-data (`manila_data.py:39`) |
| `nova-offer-url` | `nova-service` | hypervisor (`sunbeam/steps/hypervisor.py:127`) |
| `ceilometer-offer-url` | `ceilometer-service` | hypervisor via `sunbeam/features/telemetry/feature.py:130` |
| `masakari-offer-url` | `masakari-service` | hypervisor via `sunbeam/features/instance_recovery/feature.py:186` |
| `barbican-offer-url` | `barbican-service` | hypervisor via `sunbeam/features/secrets/feature.py:142` |
| `cinder-volume-database-offer-url` | `database` | `cloud/etc/deploy-storage/modules/cinder-volume/main.tf:74` |
| `manila-data-database-offer-url` | `database` | `sunbeam/features/shared_filesystem/manila_data.py:38` |
| `ingress-rgw-offer-url` | `traefik-route` | `sunbeam/steps/microceph.py:209` |
| `consul-*-cluster-offer-url` | `consul-cluster` | `sunbeam/features/instance_recovery/etc/deploy-consul-client/main.tf:32`, `:47`, `:62` |

- consul-client is the only consumer that reads via `terraform_remote_state`.
- Every other plan receives offer URLs as tfvars, so that passing `null` can remove a
  cross-model relation (LP#2085310, explained at `sunbeam/steps/hypervisor.py:115-121`).

### Other

- snap-openstack writes tfvars and reads outputs through `TerraformHelper`.
- The plan itself calls no CLI or API. CI uses the `juju` CLI and `terraform output`.

---

## 4. Q&A

### A. Integration graph: keystone, nova, neutron, glance

Assumes single region and single MySQL.

```
                       traefik (internal) ── traefik-public ── certificate-authority
                          │ ingress / traefik-route   │                 │
   ┌──────────────────────┼───────────────────────────┤        certificates (neutron)
   │                      │                           │                 │
 keystone ◄─identity-service / send-ca-cert─► glance, nova, neutron(, placement, cinder…)
   │ database                │ database             │ database           │ database
 keystone-mysql-router   glance-mysql-router   nova-, nova-api-,      neutron-mysql-router
   └───────── backend-database ──────────► mysql ◄── nova-cell-mysql-router ┘
                         rabbitmq ◄── amqp ── keystone, glance, nova, neutron
 cross-model: glance→microceph:ceph   neutron→sunbeam-ovn-proxy:ovsdb-cms   nova offers nova-service
```

With `enable-observability`, every service also gets `logging` → `opentelemetry-collector`,
and each router gets logs, metrics and dashboards → `opentelemetry-collector-infra`.

#### keystone (`main.tf:181-208`)

| This side | Other side | Condition | Where |
|---|---|---|---|
| `keystone:database` → router; router `backend-database` → `mysql:database` | MySQL | always | module `:61-87` |
| `keystone:amqp` | `rabbitmq:amqp` | always | module `:90` |
| `keystone:ingress-internal` / `ingress-public` | `traefik:ingress` / `traefik-public:ingress` | not secondary region | module `:244-273`; `main.tf:191-192`, `:71-72` |
| `keystone:receive-ca-cert` | `certificate-authority:send-ca-cert` | `traefik-to-tls-provider == "certificate-authority"` | `main.tf:591` |
| `keystone:trusted-dashboard` | `horizon` | not secondary region | `main.tf:2785` |
| `keystone:domain-config` | `keystone-ldap-*` | per `ldap-apps` | `main.tf:2126` |
| `keystone:external-idp`, `keystone-saml` | SSO IdP apps | per `sso-providers` | `main.tf:2815`, `:2845` |
| provides `identity-service`, `send-ca-cert` | glance, nova, neutron, … | — | — |
| offers `keystone-credentials`, `keystone-endpoints`, `keystone-ops`, `cert-distributor` | machine plans | always | module `:522-552` |

Units: `is-secondary-region ? 0 : os-api-scale` (`main.tf:193`).

#### glance (`main.tf:147-179`)

| This side | Other side | Condition | Where |
|---|---|---|---|
| `database` via `glance-mysql-router` | MySQL | always | module `:61-87` |
| `amqp` | rabbitmq | always | module `:90` |
| `identity-service`, `receive-ca-cert` | keystone | not secondary region | module `:106`, `:212` |
| `ingress-internal` / `ingress-public` | `traefik` / `traefik-public` | not region controller | module `:244-273` |
| `glance:ceph` | MicroCeph offer | `enable-ceph` | `main.tf:654` |
| `glance:ceph-rgw-ready` | MicroCeph RGW | `enable-ironic && enable-ceph` | `main.tf:1705` |
| `glance:cors-origin` | `horizon:cors-origin` | always | `main.tf:669` |

Units: 0 on a region controller; otherwise `os-api-scale` with Ceph, 1 without (`main.tf:164`).

#### nova (`main.tf:210-283`)

| This side | Other side | Condition | Where |
|---|---|---|---|
| `database` → `nova-mysql-router` | MySQL | always | module `:46-87` |
| `api-database` → `nova-api-mysql-router` | same MySQL | always | module `:276-320` |
| `cell-database` → `nova-cell-mysql-router` | same MySQL | always | module `:322-366` |
| `amqp`, `identity-service`, `receive-ca-cert` | rabbitmq, keystone | as for glance | module |
| `ingress-internal` / `ingress-public` | `traefik` / `traefik-public` | not region controller | module `:244-273` |
| `traefik-route-public` | `traefik-public:traefik-route` | not region controller | `main.tf:240` |
| `traefik-route-internal` | `traefik:traefik-route` | not region controller | `main.tf:255` |
| `ingress-metadata` | `traefik:ingress` | not region controller | `main.tf:270` |
| `nova:placement` | `placement:placement` | not region controller | `main.tf:638` |
| offers `nova-service` | openstack-hypervisor | always | module `:554` |

Nova uses both ingress styles at once.

#### neutron (`main.tf:315-343`)

| This side | Other side | Condition | Where |
|---|---|---|---|
| `database` via router; `amqp`; `identity-service`; `receive-ca-cert`; `ingress-*` | as for glance | as for glance | module |
| `neutron:ovsdb-cms` | external `sunbeam-ovn-proxy` offer | not region controller | `main.tf:606-619`; URL set by `sunbeam/core/ovn.py:126` |
| `neutron:certificates` | `certificate-authority` | not region controller | `main.tf:622` |
| `neutron:external-dns` | `designate:dnsaas` | `enable-designate` | `main.tf:1411` |
| `neutron:ironic-api` | `ironic:ironic-api` | `enable-ironic` | `main.tf:1674` |
| `baremetal-switch-config`, `generic-switch-config` | switch-config charms | `enable-ironic` + secrets non-empty | `main.tf:1856`, `:1871` |

### B. Variables snap-openstack sets

#### Computed in `DeployControlPlaneStep.run`

`sunbeam/steps/openstack.py:924-987`. The reapply step repeats the storage block at `:1091-1123`.

| Variables | Produced at |
|---|---|
| `enable-ceph`, `ceph-offer-url`, `ceph-nfs-offer-url`, `ceph-rgw-offer-url`, `ceph-osd-replication-count`, `enable-cinder-volume`, `cinder-volume-offer-urls` | `sunbeam/steps/openstack.py:742-767` |
| `external-ovsdb-cms-offer-url` | `sunbeam/core/ovn.py:126` |
| `region` | `sunbeam/steps/openstack.py:810` |
| `many-mysql`, `mysql-config` or `mysql-config-map` | `sunbeam/steps/openstack.py:532-548` |
| `mysql-storage` or `mysql-storage-map` | `sunbeam/steps/openstack.py:793-800` |
| `traefik-config`, `traefik-public-config`, `traefik-rgw-config` (hostname, LB IP) | `sunbeam/steps/openstack.py:812-842` |
| `rabbitmq-storage` | `sunbeam/steps/openstack.py:685` |
| `model`, `cloud`, `credential`, `config`, `ha-scale`, `os-api-scale`, `ingress-scale`, `is-region-controller` | `sunbeam/steps/openstack.py:937-950` |
| `is-secondary-region` | `sunbeam/steps/openstack.py:972` |
| `nova-config` (`pci-aliases`) | `sunbeam/steps/openstack.py:1131` (reapply only) |

#### From the manifest, via `DEPLOY_OPENSTACK_TFVAR_MAP`

`sunbeam/versions.py:171-217`.

- For each charm in `K8S_CHARMS` (`sunbeam/versions.py:51-91`) it generates
  `<charm minus -k8s>-channel`, `-revision`, `-config`, `-resources`. That covers cinder, glance,
  horizon, keystone, keystone-saml, neutron, nova, placement, mysql, mysql-router, rabbitmq, traefik.
- Explicit extras:
  - `certificate-authority-*` (`sunbeam/versions.py:214-218`)
  - `kratos-idp-channel`, `kratos-idp-revision` (`sunbeam/versions.py:196-199`)
  - `mysql-config-map`, `mysql-storage-map`, `mysql-storage`, `traefik-config-map`,
    `rabbitmq-storage`, `glance-storage` (`sunbeam/versions.py:201-212`)
- A variable is only emitted when the manifest attribute is truthy (`sunbeam/core/terraform.py:826-830`).

#### Feature flags and feature charm vars

Set by each feature's `set_tfvars_on_enable` / `_disable` plus its manifest map.

| Feature | Flag set at | Charm vars at |
|---|---|---|
| heat | `sunbeam/features/orchestration/feature.py:60` | `:39-41` |
| telemetry | `sunbeam/features/telemetry/feature.py:389` | aodh, gnocchi, ceilometer, openstack-exporter `:66-83` |
| masakari | `sunbeam/features/instance_recovery/feature.py:274` | `:72-74` |
| consul (management/tenant/storage) | `sunbeam/features/instance_recovery/feature.py:281-283`; `consul.py:113-199` | — |
| octavia | `sunbeam/features/loadbalancer/feature.py:1636` | `octavia-to-tls-provider` `:1645` |
| designate | `sunbeam/features/dns/feature.py:182` | `nameservers` `:183`; bind/designate `:107-114` |
| vault | `sunbeam/features/vault/feature.py:645` | — |
| barbican | `sunbeam/features/secrets/feature.py:75` | — |
| ironic | `sunbeam/features/baremetal/feature.py:219` | `:78-98`; shards, groups, switch secrets `constants.py:6-10` |
| magnum | `sunbeam/features/caas/feature.py:812` | — |
| manila, manila-cephfs, ceph-nfs | `sunbeam/features/shared_filesystem/feature.py:218-229` | — |
| TLS (CA, self-signed, vault) | `sunbeam/features/tls/ca.py:151-173`, `self_signed.py:89-107`, `vault.py:163-260` | `manual-tls-certificates-*` `sunbeam/features/tls/common.py:124-126` |
| validation | `sunbeam/features/validation/feature.py:287` | — |
| observability | `sunbeam/features/observability/feature.py:1197` | COS URLs `:1208-1210`; otel `:1138-1169` |
| images-sync | `sunbeam/features/images_sync/feature.py:60` | — |
| watcher | `sunbeam/features/optimization/feature.py:69` | — |
| ldap | `sunbeam/features/ldap/feature.py:100-295` | — |
| sso | `sunbeam/steps/sso.py:208-219` | `saml-x509-keypair` `:1490` |
| horizon plugins | `sunbeam/features/interface/v1/openstack.py:455-480` | — |

#### Mismatches between the two repos

1. **`rabbitmq-revision` is silently ignored.** The snap emits it and `main.tf:139` passes it,
   but `modules/rabbitmq/main.tf:33-37` has no `revision =`. Pinning a RabbitMQ revision in the
   manifest does nothing.
2. **`manila-cephfs-config` is dropped.** The snap emits it
   (`sunbeam/features/shared_filesystem/feature.py:90`), but `module "manila-cephfs"`
   (`main.tf:1951-1975`) never passes `resource-configs`.
3. **`switch-conf-secrets` is unused.** The snap sets it (`sunbeam/features/baremetal/constants.py:10`);
   nothing in `main.tf` reads it. `metrics-storage-offer-url` is unused on both sides.
4. **Generated names with no variable.** The map can produce `*-resources`, `mysql-router-config`
   and `keystone-saml-config`, none of which exist in `variables.tf`. They only appear if a manifest
   sets that attribute; Terraform then warns about an undeclared variable.
5. **External keystone offer URLs are never set.** The snap has them commented out
   (`sunbeam/steps/openstack.py:973-985`, pending provider issue #805) but still sets
   `is-secondary-region = true`. The module guards its own relations with `can(coalesce())`;
   the direct relations in `main.tf` do not (`:1013`, `:1043`, `:1127`, `:1157`, `:2248`, `:2279`,
   `:2483`, `:2543`). On a secondary region with telemetry, validation or images-sync enabled they
   would be created with `offer_url = null`. *Not confirmed with a plan run.*
6. **cloudkitty is never enabled by the snap.** Nothing references `enable-cloudkitty` or
   `cloudkitty-*`; only CI exercises it.
7. **Left at Terraform defaults:** `openstack-channel`, `mysql-base`, `mysql-router-base`,
   `traefik-base`, `kratos-idp-base`, `cloud-region`, `keystone-storage`, `octavia-storage`,
   `vault-storage`, `traefik-storage`. `cinder-volume-offer-url` is deprecated (`main.tf:802-806`).

### C. Conditional resources

| # | Pattern | Examples |
|---|---|---|
| 1 | `count = var.enable-X ? 1 : 0`, then `[count.index]` / `[0]` | heat `main.tf:831`; aodh, gnocchi `:894`, `:925`; vault `:1427`; otel `:2325`; traefik-rgw `:502` |
| 2 | Compound: `(var.enable-a && var.enable-b) ? 1 : 0` | octavia→barbican `:1286`; barbican→vault `:1523`; watcher/cloudkitty→gnocchi `:2606`, `:2771`; masakari→consul `:2688-2718`; every `*-to-observability` relation |
| 3 | Chained: `count = length(<other conditional>)` | glance→ceph `:655`; router relations `:735`, `:2028`; nova routers module `:293` |
| 4 | Conditional `for_each`: `var.enable-ironic ? var.map : {}` | shards and conductor groups `:1720`, `:1747`, `:1777`, `:1805` |
| 5 | Null-filtered `for_each` for many-mysql; `count = many-mysql ? 0 : 1` for single | `:31-54`, `:119`; `:103` |
| 6 | Data-driven `for_each`, no flag | `ldap-apps` `:2096`; `sso-providers` `:2801`, `:2831` |
| 7 | Inside the module: `for_each = var.x == "" ? {} : { target = var.x }` | module `:91-245` |
| 8 | Inside the module: `!can(coalesce(var.url))` for null-or-empty | module `:122` |
| 9 | Inside the module: `!= null` for observability | module `:369` |
| 10 | Inside the module: branch on service name | `var.name == "nova"` `:277`; `"keystone"` `:523` |
| 11 | Topology by name-blanking: locals set an app name to `""`, which removes the relation | `main.tf:60-72` |
| 12 | Topology by `scale = 0`: app kept, zero units | `main.tf:164`, `:193`, `:226`, `:301`, `:331`, `:360` |
| 13 | String-valued switches | `traefik-to-tls-provider` / `octavia-to-tls-provider` `:592`, `:1302`, `:2142`, `:2158`, `:2173-2203`; `netconf-conf-secrets != ""` `:1857`, `:1872` |

**Rules you'll likely trip over**

- Some feature modules hard-code `juju_application.traefik.name` instead of the region-aware local:
  aodh (`main.tf:907`), octavia (`:1235`), designate (`:1379`), magnum, manila, watcher, masakari,
  cloudkitty.
- `ironic-conductor-groups-to-ceph-rgw` (`main.tf:1804-1816`) indexes `microceph-ceph-rgw[0]` with
  no `enable-ceph` guard. It also uses endpoint `ceph-rgw`, while its siblings use `ceph-rgw-ready`
  (`main.tf:1696`).
- `var.traefik-public-config != {}` (`main.tf:449`, also `:515`) compares a `map(string)` to an
  `{}` literal. Terraform's type-strict equality may make this always true — check in
  `terraform console`.

### D. Source of truth and the 2024.1 branch

**This repo is the source of truth.** snap-openstack does not vendor `deploy-openstack`.

- `cloud/etc/` holds only machine-side plans: `deploy-k8s`, `deploy-microceph`, `deploy-microovn`,
  `deploy-openstack-hypervisor`, `deploy-cinder-volume`, `deploy-storage`, `deploy-role-distributor`,
  `deploy-sunbeam-machine`, `demo-setup`.
- Some features ship their own plans under `sunbeam/features/*/etc/`: consul-client, cni,
  loadbalancer-setup, cos, grafana-agent, hardware-observer, pro, manila-data.
- Every OpenStack control-plane feature uses `TerraformPlanLocation.SUNBEAM_TERRAFORM_REPO`, which
  resolves to the same `etc/deploy-openstack`
  (`sunbeam/features/interface/v1/openstack.py:75-131`, `:156`).

**How the snap gets the plan**

- `snap/snapcraft.yaml:161-186` fetches `canonical/sunbeam-terraform` at build time with
  `source-depth: 1` and **no `source-branch`**.
- So snap-openstack `main` (version `"2026.1"`, `snap/snapcraft.yaml:7`) gets whatever
  sunbeam-terraform `main` is on build day. It is not pinned.

**Branch history**

| Commit | Date | Change |
|---|---|---|
| `77a8bc4f` | 2026-06-12 | Added `source-branch: stable/2024.1` to cut the 2024.1 branch |
| `4dbc038c` | 2026-06-12 | Bumped version to 2026.1 and removed `source-branch` |

So the 2024.1 snap most likely builds from snap-openstack `stable/2024.1` with sunbeam-terraform
`stable/2024.1` pinned. *Not confirmed:* the local `origin` is the fork `vsz-work/snap-openstack`,
which only has `main`. To check:

```sh
git fetch https://github.com/canonical/snap-openstack stable/2024.1
```

**Mapping for 2024.1**

| | sunbeam-terraform `stable/2024.1` | sunbeam-terraform `main` |
|---|---|---|
| Head | `a0265ec` (2026-10-08) | `2a2a076` |
| `openstack-channel` default | `"2024.1/stable"` | `"2026.1/stable"` |
| Provider | `juju/juju 2.3.1` | `juju/juju 2.3.1` |
| OVN | `ovn-central-*`, `ovn-relay-*` vars (OVN in k8s; matches `ovn-central-k8s` in `manifests/2024.1/stable.yml`) | MicroOVN on machines via `external-ovsdb-cms-offer-url` |
| `mysql-router-revision` | absent | present |

- In the snap, `OPENSTACK_CHANNEL = f"{determine_version()}/stable"` (`sunbeam/versions.py:25`), so
  the snap version selects the charm track.
- The branches have diverged: each has commits the other lacks (`main` has 25 not on the branch).

> **Gotcha:** `stable/2024.1:channels/edge.tfvars` sets `openstack-channel = "2025.1/edge"`, so CI
> on the 2024.1 branch tests 2025.1 charms.

---

## 5. Issues worth raising

| Issue | Where | See |
|---|---|---|
| RabbitMQ revision pin ignored | `modules/rabbitmq/main.tf:33-37` | B1 |
| `manila-cephfs-config` never applied | `main.tf:1951-1975` | B2 |
| Unguarded external keystone relations on secondary regions | `main.tf:1013`, … | B5 |
| `[0]` index without `enable-ceph` guard; endpoint name differs | `main.tf:1804-1816` | C |
| `!= {}` map comparison may always be true | `main.tf:449`, `:515` | C |
| 2024.1 edge CI tests 2025.1 charms | `stable/2024.1:channels/edge.tfvars` | D |
