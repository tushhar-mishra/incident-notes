# 81 · Drill: "certificate has expired" on an Internal Service

> **Level:** ★★ intermediate
>
> **You will practise:** an mTLS/internal TLS failure at an exact time: finding **which** certificate (server, client, CA, webhook), checking it with `openssl`, renewing it (cert-manager or manual), making the service **reload** it, and finding the automation that silently failed.
>
> **You need:** topics 17, 27, 76.
>
> **Part 4 · On-call drills** · Previous: [80 · CrashLoopBackOff after a ConfigMap change](80-drill-crashloop-after-configmap.md) · Next: [82 · Intermittent DNS failures](82-drill-dns-intermittent.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📟 **00:02 — PAGE:** `orders-svc error rate 100% — upstream inventory-svc: x509: certificate has expired or is not yet valid`

**The idea.** Errors started at **00:00 UTC exactly**, nothing deployed. One service can't talk to another over TLS. Find the certificate, renew it, make the server serve the new one, and find out why renewal didn't happen.

```
exact time + x509 error → certificate (or clock)
openssl s_client → which cert, notAfter → renew (cert-manager / manual) → reload server
→ why didn't automation renew it? (cert-manager Certificate not Ready, missing Issuer, expired ACME/Vault token)
```

**The one thing to know.** Check **both sides** and **the clock**: expired server cert, expired **client** cert (mTLS), expired **CA**, or a skewed clock (topic 17) all produce similar errors.

> 📖 Read only **Symptoms**, then write your first three checks.

---

## Symptoms

- 100% errors on calls from `orders-svc` to `inventory-svc`; other services fine (or every service calling inventory-svc fails).
- Errors began within seconds of a round time (00:00 UTC).
- Logs: `x509: certificate has expired or is not yet valid: current time ... is after ...`.

## Analogy

A staff ID badge that stopped working at midnight. The door reader isn't broken and the employee is the same person — the badge's date passed. Somebody was supposed to print new badges.

## How it happens

Internal certificates (service mesh, mTLS between services, database TLS, webhooks) are issued by an internal CA (cert-manager with a CA/Vault issuer, a mesh CA). Renewal fails silently when the issuer breaks (expired Vault token, deleted Issuer, RBAC change), or a service **doesn't reload** the renewed certificate (topic 27).

### Lab setup (break it)

```bash
# Create a CA and a server cert that expires in 1 day, then test with a client clock moved forward:
openssl req -x509 -newkey rsa:2048 -nodes -keyout ca.key -out ca.crt -subj "/CN=lab-ca" -days 365
openssl req -newkey rsa:2048 -nodes -keyout srv.key -out srv.csr -subj "/CN=inventory"
openssl x509 -req -in srv.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out srv.crt -days 1
# Run a TLS server with it, then test with a client clock moved forward (or wait a day):
openssl s_server -accept 8443 -cert srv.crt -key srv.key -quiet &
faketime '+2 days' curl --cacert ca.crt https://localhost:8443 -v 2>&1 | grep -i expired   # needs libfaketime
```
(On Kubernetes with cert-manager: create a Certificate with `duration: 1h`, `renewBefore: 5m`, then break the Issuer's secret so renewal fails.)

## How to find it

**First — which certificate is presented, and is it expired?**

```bash
echo | openssl s_client -connect inventory-svc.shop.svc:8443 -servername inventory-svc 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```
```
subject=CN = inventory-svc
issuer=CN = internal-ca
notBefore=Oct 21 00:00:00 2026 GMT
notAfter=Nov 20 00:00:00 2026 GMT       ← expired at 00:00
```

Also check the **client** certificate the caller uses (mTLS) and the **CA** itself:

```bash
kubectl -n shop get secret orders-client-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -enddate
kubectl -n shop get secret internal-ca -o jsonpath='{.data.ca\.crt}' | base64 -d | openssl x509 -noout -enddate
date -u; chronyc tracking | grep "System time"      # rule out clock skew
```

**Second — why wasn't it renewed?**

```bash
kubectl get certificate -A
kubectl -n shop describe certificate inventory-tls | sed -n '/Status/,$p'
kubectl -n shop get certificaterequest
kubectl -n cert-manager logs deploy/cert-manager --tail=50 | grep -i -E "error|inventory"
```
```
NAMESPACE  NAME           READY  SECRET         AGE
shop       inventory-tls  False  inventory-tls  30d
  Message: Issuing certificate ... error: failed to sign: Vault returned 403: permission denied
```
- Renewal has been failing (e.g. the Vault issuer's token/role changed).

**Third — is the server serving the latest secret?** Compare the Secret's `notAfter` with what `s_client` shows; if the Secret is new but the served cert is old, the app needs a reload.

## How to fix it

### Immediate

1. Fix the issuer (restore the Vault role/token, recreate the Issuer secret) and **force renewal**: `cmctl renew inventory-tls -n shop`.
2. If the issuer can't be fixed quickly: issue a short-lived cert manually from the CA and put it in the Secret.
3. **Reload** the server: `kubectl -n shop rollout restart deploy/inventory-svc` (or SIGHUP if supported).
4. If it's a clock problem: fix NTP on the affected nodes (topic 17).

### Permanent

- Alerts: `certmanager_certificate_ready_status == 0`, `certmanager_certificate_expiration_timestamp_seconds - time() < 7d`, blackbox probes on internal TLS endpoints.
- Apps hot-reload certificates (or use a sidecar/mesh that does).
- Issuer credentials monitored and rotated themselves.
- Inventory of all internal certificates and CAs, including CA expiry dates.

## How to verify the fix

- `openssl s_client` shows the new `notAfter`; client calls succeed; error rate 0.
- `kubectl get certificate`: `READY=True`, renewal time in the future.
- A test renewal (`cmctl renew`) succeeds end to end.

## Prevention

Expiry and readiness alerts, short-lived certs renewed often (so failures appear early, not once a year), runbook "renew + reload".

## Follow-up questions

1. **Q: Root cause?**
   A: cert-manager couldn't renew because the Vault issuer lost permissions weeks ago; no alert on Certificate readiness, so the old cert simply expired.
2. **Q: How did you detect it? Why not earlier?**
   A: Error-rate page at expiry. A readiness/expiry alert would have fired as soon as renewal first failed.
3. **Q: What did you check first?**
   A: The certificate actually served (`s_client`) and the clock — to confirm "expired" and which cert.
4. **Q: How did you stop impact?**
   A: Fixed issuer permissions, forced renewal, restarted the service to load the new cert.
5. **Q: Permanent fix?**
   A: Readiness and expiry alerts, issuer credential monitoring, cert hot-reload.
6. **Q: How did you confirm?**
   A: New notAfter served, calls succeed, Certificate Ready.
7. **Q: What if renewal worked but errors continued?**
   A: The server didn't reload, another replica/LB still serves the old cert, the client cert or CA expired, or clock skew. Check each side.
8. **Q: How would you prevent it?**
   A: Monitor the automation, not just the certificates; short lifetimes; inventory.
9. **Q: What would you do differently?**
   A: Keep a "manual issue" runbook ready for when the issuer is broken during an expiry.
10. **Q: What if the internal CA itself expired?**
    A: Every cert it signed fails at once. You need a new CA, re-issue all certs, and distribute the new CA bundle to all clients — plan CA rotation years ahead and monitor CA expiry.
