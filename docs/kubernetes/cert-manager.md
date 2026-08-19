# cert-manager

Every service in the cluster wants HTTPS, and none of them want to think about it. cert-manager is the controller that makes certificates a thing you declare rather than a thing you renew.

You describe an Issuer once, add an annotation to an Ingress, and cert-manager requests the certificate, proves you own the domain, stores the result in a Secret, and reissues it at 30 days remaining. Nothing on your calendar, no expired-cert Monday.

This assumes a running K3s cluster with [Helm configured](helm-install.md).

## Why not just self-sign?

Because you will spend the next two years clicking through browser warnings and adding exceptions on every phone in the house.

Free certificates from Let's Encrypt work fine for internal services, with one catch: to prove you own the domain, you have to answer a challenge.

| Challenge | How it proves ownership | Needs public inbound? | Wildcards? |
|---|---|---|---|
| **HTTP-01** | Serves a token at `http://host/.well-known/acme-challenge/…` | Yes, port 80 open to the internet | No |
| **DNS-01** | Writes a `_acme-challenge` TXT record via your DNS provider's API | No | Yes |

**For a homelab, use DNS-01.** Your services are not exposed to the internet, and they shouldn't have to be just to get a certificate. DNS-01 also gets you a single wildcard cert for `*.lab.example.com`, which means adding a service is one Ingress and zero certificate work.

Use HTTP-01 only for the handful of things genuinely published to the world.

!!! note "Prerequisites"

    - A real domain you control (`example.com`) — `.local` and `.lan` cannot get public certs
    - DNS hosted somewhere with an API. Cloudflare is free and cert-manager supports it natively
    - Helm working against the cluster, with `KUBECONFIG` exported

## 1. Install cert-manager

Helm chart, own namespace, CRDs installed by the chart:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

!!! warning "`crds.enabled` vs `installCRDs`"

    The flag was renamed in chart v1.15. On older charts it is `--set installCRDs=true`. Setting the wrong one is silently ignored, the CRDs never install, and every `ClusterIssuer` you apply fails with `no matches for kind`. Check your chart version with `helm search repo jetstack/cert-manager`.

Three deployments should come up:

```bash
kubectl get pods -n cert-manager
```

```text
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5d849b8f4c-7xk2p              1/1     Running   0          45s
cert-manager-cainjector-6f9b4c8d5-mn4qz    1/1     Running   0          45s
cert-manager-webhook-79c4f8b6d9-p2vlt      1/1     Running   0          45s
```

The webhook takes a few extra seconds to become ready. Anything you apply before it does gets rejected with a connection-refused error — wait for it:

```bash
kubectl wait --for=condition=Available deployment --all -n cert-manager --timeout=120s
```

## 2. Give it DNS API access

Create a **scoped** API token at Cloudflare → My Profile → API Tokens → Create Token → *Edit zone DNS*. Restrict it to the one zone you are using. Do not use a Global API Key; it can do anything to every zone on the account, and it ends up sitting in a Secret in your cluster.

```bash
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token='YOUR_TOKEN_HERE'
```

The Secret must live in the `cert-manager` namespace — a `ClusterIssuer` reads its credentials from the namespace cert-manager runs in, not from wherever your app lives. This is the single most common wiring mistake.

## 3. Create the issuers

Start with staging. Let's Encrypt production limits you to **5 duplicate certificates per week**, and a typo will burn through that before lunch. Staging has generous limits and issues from an untrusted root — the browser will still warn, but a certificate arriving at all is what you are testing.

`clusterissuer-staging.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

`clusterissuer-prod.yaml` — identical apart from the server URL and the two names:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

Apply both and check they registered with the ACME server:

```bash
kubectl apply -f clusterissuer-staging.yaml -f clusterissuer-prod.yaml
kubectl get clusterissuer
```

```text
NAME                  READY   AGE
letsencrypt-prod      True    8s
letsencrypt-staging   True    8s
```

`READY: False` here means the account registration failed — almost always a bad email or no outbound internet from the pod. `kubectl describe clusterissuer letsencrypt-staging` prints the reason.

**ClusterIssuer or Issuer?** `Issuer` is namespaced and only serves Certificates in its own namespace. `ClusterIssuer` works cluster-wide. In a homelab you want one ClusterIssuer, not one Issuer per namespace.

## 4. Request a wildcard certificate

One certificate, every service. `wildcard-cert.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-lab
  namespace: default
spec:
  secretName: wildcard-lab-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  commonName: "*.lab.example.com"
  dnsNames:
    - "lab.example.com"
    - "*.lab.example.com"
```

```bash
kubectl apply -f wildcard-cert.yaml
kubectl get certificate -w
```

Issuance takes 1–3 minutes, most of it waiting on DNS propagation. Watch the chain of objects it creates — `Certificate` → `CertificateRequest` → `Order` → `Challenge`:

```bash
kubectl describe certificate wildcard-lab
kubectl get challenge -A
```

When `READY` goes `True`, the Secret `wildcard-lab-tls` exists and contains `tls.crt` and `tls.key`.

Now switch the `issuerRef` to `letsencrypt-prod`, delete the Secret so it reissues, and apply again:

```bash
sed -i 's/letsencrypt-staging/letsencrypt-prod/' wildcard-cert.yaml
kubectl delete secret wildcard-lab-tls
kubectl apply -f wildcard-cert.yaml
```

!!! tip "One cert, many namespaces"

    A Secret only exists in its own namespace, so a wildcard in `default` is useless to an Ingress in `media`. Either request the same Certificate in each namespace, or install [trust-manager](https://cert-manager.io/docs/trust/trust-manager/) or the reflector controller to mirror the Secret. Requesting it twice is fine — Let's Encrypt counts *duplicate* certificates, and cert-manager renews each independently.

## 5. Use it on an Ingress

K3s ships Traefik as its default ingress controller, so the ingress class is `traefik`. Reference the Secret directly:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: default
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - grafana.lab.example.com
      secretName: wildcard-lab-tls
  rules:
    - host: grafana.lab.example.com
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

Or let cert-manager do it per-host. Add the annotation and skip the Certificate resource entirely — cert-manager sees the annotation, reads `spec.tls`, and creates the Certificate for you:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

The annotation is convenient for one-offs. The wildcard is better once you have more than about five services, because every annotated Ingress is a separate certificate counting against the same rate limit.

Point your internal DNS — AdGuard works well for this — at the cluster's load balancer IP for `*.lab.example.com`, and the name resolves internally while the certificate stays publicly valid.

## 6. Confirm renewal is armed

cert-manager renews at two-thirds of the certificate lifetime, so about day 60 of 90. You do not schedule anything. Verify it knows that:

```bash
kubectl get certificate wildcard-lab -o jsonpath='{.status.renewalTime}'
```

Force a renewal at any time to prove the path still works, rather than finding out in two months:

```bash
kubectl cert-manager renew wildcard-lab   # needs the kubectl cert-manager plugin
# or, without the plugin:
kubectl delete secret wildcard-lab-tls
```

Worth adding a check to [Uptime Kuma](../docker-containers/uptime-kuma.md) on one HTTPS endpoint with certificate expiry notification switched on. cert-manager is reliable, but the failure mode is silent and 30 days long.

## Troubleshooting

Work down the chain — the useful error is usually one level below where you are looking:

```bash
kubectl describe certificate <name>
kubectl describe certificaterequest
kubectl describe order
kubectl describe challenge
kubectl logs -n cert-manager deploy/cert-manager
```

**`no matches for kind "ClusterIssuer"`.** CRDs are not installed. See the warning in step 1.

**Challenge stuck in `pending`, reason `DNS record not yet propagated`.** Normal for the first couple of minutes. If it persists past five, the token is wrong or lacks *Zone:DNS:Edit* on that zone. Check the TXT record actually appeared: `dig TXT _acme-challenge.lab.example.com @1.1.1.1`.

**`secret "cloudflare-api-token" not found`.** It is in the wrong namespace. A ClusterIssuer reads it from `cert-manager`, not from the app's namespace.

**`too many certificates already issued for exact set of domains`.** You hit the production rate limit — 5 duplicates per week, and the window is rolling, so waiting is the only fix. This is what staging is for.

**Self-check fails / `presented key authorization does not match`.** Two things are answering for that name. Usually a leftover Challenge from a previous attempt: `kubectl delete challenge --all -A`.

**Webhook errors: `connection refused` or `context deadline exceeded`.** The webhook pod is not ready yet, or a network policy blocks the API server from reaching it on port 10250.

!!! danger "Uninstalling removes your certificates"

    `helm uninstall cert-manager` deletes the CRDs, and deleting the CRDs deletes every `Certificate` object in the cluster along with the Secrets they own. Every service loses TLS at once. If you are moving cert-manager somewhere else, back up the `*-tls` Secrets first:

    ```bash
    kubectl get secret -A -o yaml -l 'controller.cert-manager.io/fao=true' > certs-backup.yaml
    ```
