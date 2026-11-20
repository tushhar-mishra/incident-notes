# 27 · Expired TLS Certificates

> **Level:** ★ foundation
>
> **You will learn:** what a **TLS** (Transport Layer Security) certificate is, what **expires** (leaf, **intermediate**, **root**), the many places certificates hide (load balancers, ingress, **mTLS** between services, **Kubernetes control plane**, kubelet, webhooks, Java truststores, IoT devices), the exact `openssl` commands to check expiry and chains, how to renew safely, why **automation that silently fails** is the usual root cause, and how certificate lifetimes are getting **shorter**.
>
> **You need:** topics 01–26. Helpful: [SD 05 · TLS and HTTPS](../../System-Design/notes/05-tls-and-https.md), [K8s 51 · Certificates and CAs](../../CKA-Certification-Course-2025/notes/51-certificates-and-cas.md), [K8s 53 · TLS inside the cluster](../../CKA-Certification-Course-2025/notes/53-tls-inside-the-cluster.md), [K8s 71 · Ingress TLS](../../CKA-Certification-Course-2025/notes/71-ingress-tls-and-host-routing.md), [Docker 39a · TLS and reverse proxies](../../Docker/notes/39a-tls-and-reverse-proxies.md), [Linux 064 · Cryptography basics](../../linux-mastery-rhel/notes/064-cryptography-basics.md).
>
> **Part 2 · Recurring failures · D. Network, DNS, certificates** · Previous: [26 · DNS failures](26-dns-failures.md) · Next: [28 · Network and firewall changes](28-network-and-firewall-changes.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A TLS certificate proves a server's identity and has a **"not after"** date. After that moment, clients **refuse** to connect. The outage starts at an exact second, often with no change at all on your side.

```
Leaf cert (api.example.com)  ──signed by──►  Intermediate CA  ──signed by──►  Root CA (in client's trust store)
expires in 90 days                          expires in years                  expires in decades
   any of these expiring (or missing from the chain) breaks the connection
```

**Picture it.** A passport. The border guard doesn't care that you're the same person as yesterday; if the date has passed, you don't get in. And if the issuing country's stamp (intermediate) is missing, you don't get in either.

**The one thing to know.** Expiry outages are almost never "we forgot to buy a cert". They're "**auto-renewal silently broke months ago**", or "**a cert nobody knew existed**" (internal mTLS, a webhook, the Kubernetes control plane, an old device trusting an old root).

**Try it now:**
```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -dates -issuer
```

**You've got it if you can:** check any endpoint's expiry and chain, renew kubeadm certificates, and design expiry monitoring.

> 📖 **That's the core.** Everything below is the **full reference**. Real cases: [57 · Certificate expiry outages](57-certificate-expiry-outages.md).

---

## Symptoms

| Where | What you see |
|---|---|
| Browsers | `NET::ERR_CERT_DATE_INVALID`, "Your connection is not private" |
| curl | `curl: (60) SSL certificate problem: certificate has expired` |
| Go apps / kubectl | `x509: certificate has expired or is not yet valid: current time 2026-11-20T00:00:05Z is after 2026-11-19T23:59:59Z` |
| Java | `PKIX path validation failed: CertificateExpiredException` / `PKIX path building failed` (missing chain/root) |
| Service mesh / mTLS | Service-to-service calls fail with TLS handshake errors; one direction only |
| Kubernetes control plane | `kubectl` fails with x509 errors; kubelets can't talk to the API server; nodes NotReady |
| Timing | Starts at an exact time (often 00:00 UTC or a round hour) with no deploy |
| Partial | Only **old clients** fail (old phones, old Java, IoT devices) when a **root** changes or expires |

---

## Analogy

The **passport** (Core). A **root expiring** is like a country ceasing to exist: every passport it issued becomes useless, unless your border guard (client) has learned about the new country (updated trust store).

---

## How it happens

| Cause | Example |
|---|---|
| **Renewal automation silently failing** | cert-manager can't complete the ACME challenge (DNS/ingress changed); certbot timer disabled; IAM permission removed; renewal job's own credentials expired |
| **Renewed but not deployed** | New cert on disk, but nginx/haproxy/Java never reloaded it; LB still serves old cert |
| **Unknown certificates** | Internal mTLS certs, webhook certs, database TLS, VPN, SAML signing certs, code-signing, a vendor appliance |
| **Kubernetes (kubeadm)** | Control-plane certs issued by kubeadm are valid for **1 year**. They renew during `kubeadm upgrade`; clusters not upgraded for a year break |
| **Chain problems** | Server sends the leaf without the intermediate; works in browsers (they cache intermediates), fails in curl/Java/apps |
| **Root expiry / change** | Let's Encrypt's old cross-signed root (DST Root CA X3) expired on 30 September 2021; old devices that didn't trust the newer root failed (topic 57) |
| **Clock skew** | Valid cert seen as expired or "not yet valid" because the client's clock is wrong (topic 17) |

### Lifetimes are getting shorter

Public TLS certificates used to last years. In 2025 the **CA/Browser Forum** (the body of certificate authorities and browser makers) approved a schedule (ballot **SC-081**) that steps the maximum lifetime of public TLS certificates down from 398 days to **200 days (from March 2026)**, **100 days (2027)** and **47 days (2029)**. Let's Encrypt already issues 90-day certificates. **Manual renewal stops being an option**; automation and monitoring become mandatory. (Source: [CA/Browser Forum, Ballot SC-081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/).)

---

## How to find it

**First:** which certificate, served by what, and when does it expire?

```bash
echo | openssl s_client -connect api.example.com:443 -servername api.example.com -showcerts 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```
```
subject=CN = api.example.com
issuer=C = US, O = Let's Encrypt, CN = R11
notBefore=Aug 21 00:00:00 2026 GMT
notAfter=Nov 19 23:59:59 2026 GMT
```
- `notAfter` is **yesterday** → expired.
- `-servername` matters: it sends SNI (Server Name Indication); without it you may see a different (default) cert.

Check the chain the server sends:
```bash
echo | openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null | grep -E "s:|i:|Verify return"
```
```
 0 s:CN = api.example.com
   i:C = US, O = Let's Encrypt, CN = R11
Verify return code: 20 (unable to get local issuer certificate)
```
- Only the leaf (`0`) is sent; no intermediate (`1`) → **incomplete chain**.

**Second:** where is that certificate stored, and why wasn't it renewed?

```bash
# Kubernetes + cert-manager
kubectl get certificate -A
kubectl describe certificate api-tls -n shop          # Ready=False? reason?
kubectl get certificaterequest,order,challenge -n shop
kubectl get secret api-tls -n shop -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -enddate
```
```
NAMESPACE   NAME      READY   SECRET    AGE
shop        api-tls   False   api-tls   400d
...
Message: Issuing certificate as Secret was previously issued ... challenge failed: 404 from http://api.example.com/.well-known/acme-challenge/...
```
- Renewal failed because the HTTP-01 challenge path returns 404 (ingress routing changed).

```bash
# AWS ACM
aws acm list-certificates --query 'CertificateSummaryList[].[DomainName,NotAfter,RenewalEligibility]' --output table
# Files on a server
sudo find /etc -name "*.crt" -o -name "*.pem" 2>/dev/null | xargs -I{} sh -c 'echo -n "{} "; openssl x509 -noout -enddate -in {} 2>/dev/null'
```

**Third (kubeadm clusters):**
```bash
sudo kubeadm certs check-expiration
```
```
CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY
admin.conf                 Nov 19, 2026 10:02 UTC   <invalid>       ca
apiserver                  Nov 19, 2026 10:02 UTC   <invalid>       ca
...
CERTIFICATE AUTHORITY      EXPIRES                  RESIDUAL TIME
ca                         Nov 17, 2035 10:02 UTC   8y
```
- Leaf certs `<invalid>` (expired), CA still valid → renew the leaf certs.

---

## How to fix it

### Immediate

- **Renew and deploy** the certificate:
  - cert-manager: fix the challenge problem, then `cmctl renew api-tls -n shop` (or delete the CertificateRequest to retry).
  - certbot: `certbot renew --force-renewal` then reload the web server.
  - ACM: request/validate; for imported certs, re-import.
  - Manual: issue a new cert, install the **full chain** (leaf + intermediate), **reload** the server (`nginx -s reload`, `systemctl reload haproxy`).
- **kubeadm:**
  ```bash
  sudo kubeadm certs renew all
  # restart control-plane static pods (kube-apiserver, controller-manager, scheduler, etcd) so they load new certs,
  # e.g. by moving their manifests out of /etc/kubernetes/manifests and back, then
  sudo cp /etc/kubernetes/admin.conf ~/.kube/config
  ```
- **Incomplete chain:** serve `fullchain.pem`, not `cert.pem`.
- If a client's trust store is the problem (old root): update the client's CA bundle, or serve a chain that the old clients trust (if one exists).

### Permanent

- **Automate renewal everywhere** (cert-manager, ACM, certbot timers, Vault PKI) **and monitor it** — the automation is what fails.
- **Inventory** every certificate (public and internal), with owner and expiry.
- **Short-lived internal certs** issued automatically (service mesh, SPIFFE/SPIRE, Vault) so expiry is continuous and tested daily.
- Keep Kubernetes clusters upgraded (kubeadm renews certs on upgrade); enable kubelet certificate rotation.
- Reload servers automatically when certs change.

---

## How to verify the fix

- `openssl s_client` from **outside** shows the new `notAfter` and a full chain (`Verify return code: 0 (ok)`).
- Test with strict clients (curl, Java, the oldest client you support).
- cert-manager `READY=True`; renewal time set ~2/3 through lifetime.
- Error rate normal; mTLS connections succeed in both directions.

---

## Prevention

- **External expiry monitoring** of every endpoint: Prometheus `blackbox_exporter` (`probe_ssl_earliest_cert_expiry`), alert at **< 21 days** (ticket) and **< 7 days** (page).
- Monitor the **automation itself**: alert on cert-manager `certmanager_certificate_ready_status == 0`, failed renewals.
- `kubeadm certs check-expiration` in a monthly check; alert on control-plane cert age.
- Certificate Transparency (CT) log monitoring finds certificates issued for your domains you didn't know about.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: The trigger was the expiry date. The root cause was that cert-manager's renewal had been failing for 30 days because an ingress change broke the ACME HTTP-01 challenge path, and nothing alerted on a failing renewal or on the expiry date.

2. **Q: How did you detect it? Why not earlier?**
   A: Users and synthetic checks failed at the exact expiry time. We had no expiry alert, and the automation failed silently.

3. **Q: What did you check first?**
   A: `openssl s_client` against the endpoint to confirm expiry and see which cert/chain was actually being served, because the cert in the Secret and the one served may differ.

4. **Q: How did you stop impact quickly?**
   A: Fixed the challenge route and forced renewal; as a faster fallback we could have issued a cert via DNS-01 or uploaded a manually issued cert.

5. **Q: Permanent fix?**
   A: Blackbox expiry alerts on all endpoints, alerts on cert-manager not Ready, and a test in the ingress pipeline that `/.well-known/acme-challenge/` routes correctly.

6. **Q: How did you confirm?**
   A: New notAfter from outside, full chain verified, strict clients succeed, cert-manager Ready.

7. **Q: What if renewing didn't fix it?**
   A: Then the server is still serving the old cert (not reloaded, another replica, a different LB/CDN layer), the chain is incomplete, or clients have a trust store or clock problem. Check what's served from several clients and locations.

8. **Q: The Kubernetes API says x509 expired. What do you do?**
   A: On kubeadm, `kubeadm certs check-expiration`, then `kubeadm certs renew all`, restart the control-plane static pods, update admin kubeconfig, and check kubelet client cert rotation. Long-term: upgrade the cluster at least yearly and monitor cert age.

9. **Q: What would you do differently?**
   A: Monitor the expiry dates and the renewal automation separately, and keep an inventory of internal certs.

10. **Q: What if the expiring certificate was a root CA in clients' trust stores?**
    A: You can't renew someone else's trust store. Plan months ahead: cross-signing, updating CA bundles in your clients and images, and testing old client versions — exactly the Let's Encrypt 2021 lesson.
