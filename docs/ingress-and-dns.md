# One IP for all my apps, with names

What I want:

| name | goes to |
| --- | --- |
| `nurcahyaari.lab.internal` | homepage (in the cluster) |
| `grafana.nurcahyaari.lab.internal` | grafana (on the monitoring VM, `192.168.1.23:3000`) |
| `<anything>.nurcahyaari.lab.internal` | whatever app I add later |

All of them on **one IP**, `192.168.1.200`, on normal port 80. No more `:3000`, `:3001`, `:3002`.

## How it works

Right now every app that needs to be reachable gets its own `LoadBalancer` Service, so its own MetalLB IP. That doesn't scale, and I have to remember IPs and ports.

So instead I put one thing in front of everything, an **ingress controller**. I use [Traefik](https://traefik.io). Only Traefik gets the MetalLB IP. All my apps go back to plain `ClusterIP`.

```
browser: http://grafana.nurcahyaari.lab.internal
   │
   │ 1. DNS: grafana.nurcahyaari.lab.internal → 192.168.1.200
   ▼
192.168.1.200:80  (Traefik, the only LoadBalancer)
   │
   │ 2. Traefik reads the Host header: "grafana.nurcahyaari.lab.internal"
   │    and finds the Ingress with that host
   ▼
Service grafana → 192.168.1.23:3000
```

So DNS gets me to the IP, and the Host header gets me to the app. Many names, same IP. The app's own port (`3000` or whatever) is only used inside, from Traefik to the Service. I never type it.

One thing, this only works for HTTP/HTTPS. Stuff like Postgres or Redis doesn't send a hostname, so those still need their own port or IP.

### Why `.internal`?

`.internal` is reserved for private networks (ICANN made it official in 2024), so nobody can ever buy it, and it's free. I don't use `.local` because macOS uses it for Bonjour and lookups get weird. And I don't use `.dev` because it's a real TLD and browsers force HTTPS on all of it.

The downside is Let's Encrypt can't give certs for `.internal`, so it's plain `http://` for now. If I want real HTTPS later, I'll buy a domain.

## Step 1: Free up 192.168.1.200

Homepage owns `.200` right now. MetalLB gives one IP to one Service, so Traefik can't take it until homepage lets it go.

**Do this as its own push, before installing Traefik.** If I do both in one push, Traefik's Service sits at `<pending>`. `infrastructure` has `wait: true`, so Flux waits on it, `infrastructure` never goes ready, and `apps` (which is where homepage gets released) never runs. It just gets stuck.

`apps/tools/homepage/services.yaml`, back to ClusterIP:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: homepage
  namespace: tools
  labels:
    app.kubernetes.io/name: homepage
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: homepage
  ports:
    - port: 3000
      targetPort: http
      protocol: TCP
      name: http
```

(also `targetPort` with a capital P, I had `targetport` before and it was just ignored)

Push, then check `.200` is free:

```sh
kubectl get svc -A | grep 192.168.1.200   # should print nothing
```

Homepage is unreachable from my browser until step 3. That's fine.

## Step 2: Install Traefik

Same as metallb and external-secrets, I commit the rendered manifest in `infra/`.

```
infra/traefik/
  kustomization.yaml
  namespace.yaml
  values.yaml
  traefik.yaml        # generated, don't edit
```

`values.yaml`:

```yaml
service:
  annotations:
    metallb.io/loadBalancerIPs: 192.168.1.200
ingressClass:
  enabled: true
  isDefaultClass: true
```

The chart's Service is already `type: LoadBalancer` by default, so I only pin the IP. `isDefaultClass` means my Ingresses don't need `ingressClassName`, Traefik picks them up anyway.

Generate it:

```sh
helm template traefik oci://ghcr.io/traefik/helm/traefik \
  --version 42.4.1 \
  --namespace traefik \
  --include-crds \
  -f infra/traefik/values.yaml > infra/traefik/traefik.yaml
```

The chart doesn't make the namespace, so `namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: traefik
```

`kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - traefik.yaml
```

And `- traefik` in `infra/kustomization.yaml`.

To upgrade later, I bump `--version` and run the same command again.

Check:

```sh
kubectl get pods -n traefik             # Running
kubectl get svc -n traefik traefik      # EXTERNAL-IP 192.168.1.200, ports 80 and 443
curl -i http://192.168.1.200            # 404 from Traefik, that's good. It's up, just no Ingress yet
```

## Step 3: Homepage behind Traefik

I already have `apps/tools/homepage/ingress.yaml`, just the host is the example one. Change it:

```yaml
spec:
  rules:
    - host: nurcahyaari.lab.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: homepage
                port:
                  number: 3000
```

Homepage also checks the Host header itself, so add the new name to `HOMEPAGE_ALLOWED_HOSTS` in the deployment. No port, because it's port 80 now:

```yaml
- name: HOMEPAGE_ALLOWED_HOSTS
  value: "$(POD_IP):3000,nurcahyaari.lab.internal"
```

(I can drop `192.168.1.200:3000`, that's not homepage anymore)

Test before DNS exists, by faking the Host header:

```sh
curl -I -H "Host: nurcahyaari.lab.internal" http://192.168.1.200
```

`200 OK` means Traefik found the Ingress and homepage accepted the host.

## Step 4: Grafana, which isn't even in the cluster

Grafana runs on the monitoring VM as a systemd service, not as a pod. But Traefik can still route to it. I make a Service with **no selector**, and tell it the address myself with an `EndpointSlice`. For Traefik it looks like any other Service.

`apps/tools/grafana/grafana.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: tools
spec:
  ports:
    - name: http
      port: 3000
      targetPort: 3000
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: grafana-1
  namespace: tools
  labels:
    kubernetes.io/service-name: grafana   # this links it to the Service above
addressType: IPv4
ports:
  - name: http
    port: 3000
endpoints:
  - addresses:
      - 192.168.1.23
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: tools
spec:
  rules:
    - host: grafana.nurcahyaari.lab.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 3000
```

Plus a `kustomization.yaml` with `- grafana.yaml`, and `- grafana` in `apps/tools/kustomization.yaml`.

Grafana needs to know its own URL too, otherwise the login redirect sends me to `localhost:3000`. On the monitoring VM, in `grafana.ini` (so in the ansible role):

```ini
[server]
domain = grafana.nurcahyaari.lab.internal
root_url = http://grafana.nurcahyaari.lab.internal/
```

Same trick works for anything outside the cluster: Proxmox, Prometheus, the router UI...

## Step 5: DNS

Now I need my devices to know `*.nurcahyaari.lab.internal` → `192.168.1.200`.

### Quick: /etc/hosts

Only for my laptop, and no wildcards, so every name goes in one by one:

```
192.168.1.200  nurcahyaari.lab.internal
192.168.1.200  grafana.nurcahyaari.lab.internal
```

Good enough to test. Not good for the long run.

### Proper: a DNS server with a wildcard

One record, and every future app just works without touching DNS again. Any of these is free:

- **AdGuard Home or Pi-hole.** Add a DNS rewrite for `nurcahyaari.lab.internal` and `*.nurcahyaari.lab.internal` → `192.168.1.200`.
- **dnsmasq.** One line covers the name and all subdomains:

  ```
  address=/nurcahyaari.lab.internal/192.168.1.200
  ```

- **My router**, if it has a local DNS / custom records page and supports wildcards.

Then point devices to that DNS server, either in the router's DHCP settings (whole home) or per device.

### From outside, with Tailscale

`192.168.1.200` is inside the `192.168.1.0/24` that pve advertises, so the IP already works over Tailscale. Only the names don't. In the Tailscale admin, DNS → add a split DNS nameserver for `nurcahyaari.lab.internal` that points to my DNS server above. Then the names work from anywhere on my tailnet.

## Check everything

```sh
dig +short grafana.nurcahyaari.lab.internal     # 192.168.1.200
curl -I http://nurcahyaari.lab.internal         # 200, homepage
curl -I http://grafana.nurcahyaari.lab.internal # 302 to /login, grafana
kubectl get ingress -A                          # all my hosts
```

## Adding a new app later

1. Service as `ClusterIP` (the default, I just don't set `type`)
2. Ingress with `host: <app>.nurcahyaari.lab.internal`
3. Push

No new IP, no DNS change.
