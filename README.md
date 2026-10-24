# ITL.ExternalDNS

[external-dns](https://github.com/kubernetes-sigs/external-dns) for the ITL Talos cluster ([`ITlusions/ITL.K8s`](https://github.com/ITlusions/ITL.K8s)) - a thin wrapper chart around the upstream chart, same pattern as [`ITL.Cilium`](https://github.com/ITlusions/ITL.Cilium), [`ITL.Traefik`](https://github.com/ITlusions/ITL.Traefik), and [`ITL.K8s.MetricsServer`](https://github.com/ITlusions/ITL.K8s.MetricsServer). Syncs Kubernetes Service/Ingress/Gateway HTTPRoute hostnames into our self-hosted BIND server via RFC2136 dynamic DNS update - no cloud DNS provider involved.

**Status: deployed and live**, confirmed working end-to-end 2026-10-24 (`RFC2136TSIGKeyName:externaldns-int`, zone `int.itlusions.nl`, "All records are already up to date" on first sync).

## Moved here from `ITL.K8s`

This started as raw values (`manifests/external-dns/values-internal.yaml`) directly against the upstream chart in `ITL.K8s`, then got its own repo and a proper wrapper chart once it was actually working - same evolution `ITL.Cilium`/`ITL.Traefik` went through. See `ITL.K8s/docs/1-cluster/1.6-dns.md` for the DNS hierarchy design this fits into.

## Why RFC2136, not a cloud DNS provider

There's no cloud DNS zone here - `int.itlusions.nl`'s authoritative server is our own BIND instance on `pandora` (`192.168.1.241`), not Route53/CloudDNS/etc. RFC2136 is the standards-based dynamic-update protocol BIND (and effectively every self-hosted DNS server) speaks natively - see [`ITL.Bind.Config`](https://github.com/ITlusions/ITL.Bind.Config) for the server side of this (the zones, the TSIG key, the `update-policy` grants, and the CI/CD pipeline that deploys changes to it).

## The TSIG key, and why it's `externaldns-int` not `externaldns`

Two separate RFC2136 keys exist on `pandora`, from two separate points in this project's history:

| Key | Grants update rights on | Used by |
|---|---|---|
| `externaldns` | `public.k8s.int.itlusions.com` only | An older, now-superseded external-dns deployment (pre-dates this chart) |
| `externaldns-int` | `int.itlusions.nl`, `k8s.int.itlusions.nl`, `infra.int.itlusions.nl` | **This chart** |

Don't mix them up - the old key has no access to the zones this chart actually targets, and vice versa.

## Scoped to all three zones the key is actually granted on

`domainFilters` and `--rfc2136-zone` are both set to all three: `int.itlusions.nl`, `k8s.int.itlusions.nl`, `infra.int.itlusions.nl` - matching exactly what `externaldns-int` is granted on BIND, confirmed via `external-dns --help` (`--rfc2136-zone` can be repeated).

**`k8s.int.itlusions.nl` and `infra.int.itlusions.nl` each need their own `--rfc2136-zone` entry, not just one for the parent.** Both are independently authoritative zones on BIND (separate zone files, separate `type master` blocks in `named.conf.local`) - `k8s.int.itlusions.nl` is NS-delegated from `int.itlusions.nl`'s apex, but a delegation only hands you glue, not the child zone's actual record data. AXFR'ing `int.itlusions.nl` alone would never surface what's inside `k8s.int.itlusions.nl` or `infra.int.itlusions.nl`.

This is deliberately "every zone this key can reach," not "every zone on the server" - `public.k8s.int.itlusions.com` (old `externaldns` key) and the static `.com` zones (no grant at all) stay untouched regardless, because the TSIG key itself has no write access there. Narrowing back down later is a one-line config change, no BIND-side change needed.

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

Look for `Configured RFC2136 with zones '[int.itlusions.nl]'` and no `NOTAUTH`/`REFUSED` errors - those would mean the TSIG key/secret doesn't match what's actually granted on the server.
