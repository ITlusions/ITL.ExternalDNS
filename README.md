# ITL.ExternalDNS

[external-dns](https://github.com/kubernetes-sigs/external-dns) for the ITL Talos cluster ([`ITlusions/ITL.K8s`](https://github.com/ITlusions/ITL.K8s)) - a thin wrapper chart around the upstream chart, same pattern as [`ITL.Cilium`](https://github.com/ITlusions/ITL.Cilium), [`ITL.Traefik`](https://github.com/ITlusions/ITL.Traefik), and [`ITL.K8s.MetricsServer`](https://github.com/ITlusions/ITL.K8s.MetricsServer). Syncs Kubernetes Service/Ingress/Gateway HTTPRoute hostnames into our self-hosted BIND server via RFC2136 dynamic DNS update - no cloud DNS provider involved.

**Status: deployed and live**, confirmed working end-to-end 2026-10-24 (`RFC2136TSIGKeyName:externaldns-int`, zone `int.itlusions.nl`, "All records are already up to date" on first sync). Scoped back down to just `k8s.int.itlusions.nl` on 2026-10-25 - see below.

## Moved here from `ITL.K8s`

This started as raw values (`manifests/external-dns/values-internal.yaml`) directly against the upstream chart in `ITL.K8s`, then got its own repo and a proper wrapper chart once it was actually working - same evolution `ITL.Cilium`/`ITL.Traefik` went through. See `ITL.K8s/docs/1-cluster/1.6-dns.md` for the DNS hierarchy design this fits into.

## Why RFC2136, not a cloud DNS provider

There's no cloud DNS zone here - `int.itlusions.nl`'s authoritative server is our own BIND instance on `pandora` (`192.168.1.241`), not Route53/CloudDNS/etc. RFC2136 is the standards-based dynamic-update protocol BIND (and effectively every self-hosted DNS server) speaks natively - see [`ITL.Bind.Config`](https://github.com/ITlusions/ITL.Bind.Config) for the server side of this (the zones, the TSIG key, the `update-policy` grants, and the CI/CD pipeline that deploys changes to it).

## The TSIG key, and why it's `externaldns-int` not `externaldns`

Two separate RFC2136 keys exist on `pandora`, from two separate points in this project's history:

| Key | Grants update rights on | Used by |
|---|---|---|
| `externaldns` | `public.k8s.int.itlusions.com` only | An older, now-superseded external-dns deployment (pre-dates this chart) |
| `externaldns-int` | `k8s.int.itlusions.nl` only | **This chart** |

Don't mix them up - the old key has no access to the zone this chart actually targets, and vice versa.

## Scoped to just `k8s.int.itlusions.nl` - and why that's not "every zone this key can reach" anymore

Briefly widened to all three zones (`int.itlusions.nl`, `k8s.int.itlusions.nl`, `infra.int.itlusions.nl`) on 2026-10-24, reverted the next day. The reason is structural, not a preference: on BIND, **any zone with an `update-policy` grant becomes a "dynamic zone"**, and a dynamic zone refuses `rndc reload` for its own file entirely - confirmed live, not theoretical. `int.itlusions.nl` and `infra.int.itlusions.nl` are meant to stay plain static zones that [`ITL.Bind.Config`](https://github.com/ITlusions/ITL.Bind.Config)'s git-file + CI-deploy pipeline manages directly; granting this key on them broke that pipeline's ability to manage them at all, and - worse - the pipeline's own unconditional "copy every zone file" step would have silently wiped any real record this chart ever wrote dynamically, on every single deploy of any other zone.

`k8s.int.itlusions.nl` is the only zone actually meant to be dynamic in the first place, and is the only one `domainFilters`/`--rfc2136-zone` target now. Its file in `ITL.Bind.Config` (`zones/db.k8s.int.itlusions.nl`) is kept only as a historical record of its one-time seed content - that pipeline no longer deploys it at all.

## Deploy

The TSIG secret must exist before installing - this chart doesn't create it:

```bash
kubectl create namespace external-dns
kubectl create secret generic rfc2136-tsig -n external-dns \
  --from-literal=secret=<the real externaldns-int TSIG value - see ITL.Bind.Config's .secrets-local, never commit it>

cd itl.externaldns
helm dependency update
helm upgrade --install external-dns . -n external-dns
```

Using the release name `external-dns` in namespace `external-dns` (matching the original raw-values install) lets Helm adopt the existing Deployment/resources in place rather than creating a parallel second instance.

## Verify

```bash
kubectl get pods -n external-dns
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns --tail=40
```

Look for `Configured RFC2136 with zones '[k8s.int.itlusions.nl]'` and no `NOTAUTH`/`REFUSED` errors - those would mean the TSIG key/secret doesn't match what's actually granted on the server.
