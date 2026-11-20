# 57 · Certificate Expiry Outages: Ericsson 2018 and the Let's Encrypt Root 2021

> **Level:** ★ foundation
>
> **Pattern:** Something **expires** at a fixed moment → systems that worked yesterday refuse to work today, with **no change** on your side. **Families:** certificates/time (topic 27) · vendor dependency (topic 25).
>
> **You will learn:** two very different certificate expiry events: an **expired certificate inside vendor software** that disrupted mobile networks in several countries (Ericsson, December 2018), and a **root certificate expiry** that broke **old clients** around the world while servers were fine (Let's Encrypt's DST Root CA X3 cross-sign, September 2021); how each was handled; and how to prepare for both.
>
> **You need:** Part 1, topics 17, 27.
>
> **Part 3 · Real outages · Certificates and time** · Previous: [56 · Cloudflare December 2025](56-cloudflare-dec-2025.md) · Next: [58 · Cloudflare leap second 2017](58-cloudflare-leap-second-2017.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Certificates have an end date. When the end date is **inside software you bought**, you may not even know it exists. When the end date is a **root** that clients trust, you can't renew it at all — only prepare clients for the change.

```
Case A: Ericsson 2018
  certificate built into two SGSN–MME software versions (mobile core network) → expires
  → network disturbances for some operators in multiple countries → faulty software decommissioned

Case B: Let's Encrypt 2021
  chain: your cert ← R3 ← ISRG Root X1 ← (cross-signed by) DST Root CA X3
  30 Sep 2021: DST Root CA X3 cross-sign expires
  → modern clients: trust ISRG Root X1 directly → fine
  → old clients that don't trust ISRG Root X1, or OpenSSL 1.0.x quirk → TLS failures
```

**Picture it.** Case A: a new car whose engine software has a hidden "valid until" date — on that day, cars from that batch stop in several cities. Case B: a passport office closes down; passports it stamped stop being accepted at old border posts that never learned about the replacement office.

**The one thing to know.** **Inventory every certificate, including those inside vendor products and those your clients rely on**, and test old clients before root changes.

**You've got it if you can:** explain why the Let's Encrypt event broke clients but not servers, and what an operator could do about it.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Case A · Ericsson, 6 December 2018

### Summary
On 6 December 2018, Ericsson identified an issue in certain nodes in mobile operators' core networks that caused network disturbances for a limited number of customers in multiple countries using two specific software versions of the SGSN–MME (the core node handling mobile data sessions and mobility). Ericsson's initial root cause analysis pointed to an expired certificate in those software versions.

### Impact
| Item | Detail (from Ericsson's press release) |
|---|---|
| Affected | "a limited number of customers in multiple countries" (Ericsson's customers are mobile operators) |
| What | Network disturbances from core network nodes running two specific SGSN–MME software versions |
| Restoration | Most affected customers' services restored during 6 December; work continued with the rest |
| Named operators / subscriber counts / cost | **Not named in Ericsson's release** |

### Timeline
| When | Event |
|---|---|
| 6 Dec 2018 | Issue identified; initial root cause: expired certificate |
| During 6 Dec | Most affected customers restored |
| 6 Dec 15:50 GMT | Ericsson press release |

### How it happened
```
Vendor software (SGSN–MME, two versions) ── contains certificate ── expiry date passes
   → software fails at that moment on every node running those versions
   → operators' mobile data service disrupted
```
The specific certificate and exact failure mechanism are **not publicly disclosed** in the release.

### Why it spread
Every node running the affected versions hit the same date at the same time. Redundant nodes running the same software don't help (topic 24's "same bug + same input" lesson, here "same software + same date").

### How they found it
**Not publicly disclosed** beyond the statement that the issue was identified during 6 December.

### What they did to fix it
Restored most customers during the day. CEO Börje Ekholm: "The faulty software that has caused these issues is being decommissioned."

### Permanent fix
Ericsson's release mentions decommissioning the faulty software and a complete root cause analysis in progress; **further specifics not publicly disclosed** in the release.

---

## Case B · Let's Encrypt, DST Root CA X3 cross-sign expiry, 30 September 2021

### Summary
Let's Encrypt certificates were trusted by older devices through a cross-signature from an older root, **DST Root CA X3**, which expired on 30 September 2021. After that, trust depended on Let's Encrypt's own root, **ISRG Root X1**. Modern devices were fine; older devices that didn't trust ISRG Root X1, and clients with a certificate-verification quirk in OpenSSL 1.0.x, could fail.

### Impact
| Item | Detail (from Let's Encrypt) |
|---|---|
| Who could break | Older devices that don't trust ISRG Root X1 (examples given: "an iPhone 4 or an HTC Dream"); clients using **OpenSSL 1.0.x**, which could fail with the Android-compatible chain even if they trust ISRG Root X1 |
| Who was fine | Clients trusting ISRG Root X1; old Android devices were covered by a special arrangement (below) |
| Total number of failures | **Not publicly disclosed** |

### Timeline
| When | Event |
|---|---|
| Dec 2020 | Let's Encrypt announces an extended cross-sign to keep old Android working |
| 2021 | Default chain changes to: subscriber cert ← R3 ← ISRG Root X1 ← DST Root CA X3 |
| **30 Sep 2021** | DST Root CA X3 expires; Let's Encrypt now uses ISRG Root X1 for trust on almost all devices |
| Early 2024 | The Android-compatibility cross-sign itself approaches expiry (later note on the page) |

### How it happened
```
Server sends: [your cert] ← [R3] ← [ISRG Root X1 cross-signed by DST Root CA X3]
Client checks trust:
  modern client: "I trust ISRG Root X1" → stops there → ✓
  old client without ISRG Root X1: "I only trust DST Root CA X3" → expired → ✗
  OpenSSL 1.0.x: may follow the path to the expired DST root and fail → ✗ (quirk)
  old Android: doesn't enforce expiry of trust anchors → keeps working ✓ (by design, see below)
```

**The Android trick:** Let's Encrypt found that **Android intentionally does not enforce expiry dates of certificates used as trust anchors**. IdenTrust issued a 3-year cross-sign of ISRG Root X1 from DST Root CA X3 that **extended beyond** DST's own expiry, so old Android devices (before 7.1.1) kept working.

### Why it spread
Nothing changed on servers. Clients everywhere evaluated the same date at the same time; any client population with old trust stores or old OpenSSL failed together — often **embedded devices, old phones, and server-to-server API clients** in old containers or OS images.

### How it was found / handled
This was a **planned, announced** expiry. Let's Encrypt published guidance in advance:
- For most websites: nothing to do.
- **API providers:** all API clients must trust ISRG Root X1; OpenSSL-based clients must use **OpenSSL 1.1.0 or later**.
- **ACME client developers:** use the intermediate provided by the ACME API at issuance; don't hardcode or reuse intermediates.

### Permanent fix
Moving trust to ISRG Root X1; old clients needed updated trust stores or software.

---

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| Certificates inside appliances / vendor software / Java keystores you didn't create | Ask vendors for embedded certificate expiry dates; inventory and monitor; include in support contracts |
| Old base images with outdated CA bundles (`ca-certificates`) | Rebuild images regularly; scan for old OpenSSL ([Docker 19](../../Docker/notes/19-choosing-base-images.md), [Docker 43](../../Docker/notes/43-image-supply-chain.md)) |
| Server-to-server API clients (Java truststores, Python `certifi`, Go on old distros) | Test clients against the new chain before a root change |
| Kubernetes internal CAs (kubeadm CA lasts 10 years; leaf certs 1 year) | `kubeadm certs check-expiration`; plan CA rotation long before expiry ([K8s 51](../../CKA-Certification-Course-2025/notes/51-certificates-and-cas.md)) |
| Hard-coded intermediate certificates in configs | Use the full chain supplied by the CA at issuance |

## Follow-up questions

1. **Q: What was the real root cause in each case?**
   A: Ericsson: an expired certificate embedded in two software versions, so every node running them failed at the same moment. Let's Encrypt: a planned root cross-sign expiry; the "root cause" of client failures was old trust stores and an OpenSSL 1.0.x verification quirk.

2. **Q: How were they detected, and why not earlier?**
   A: Ericsson's was found on the day it happened (details not public). The Let's Encrypt change was announced long in advance; failures were detected by those who hadn't tested old clients.

3. **Q: What would you check first if TLS suddenly fails for some clients only?**
   A: The chain the server sends (`openssl s_client -showcerts`), what roots the failing clients trust, and their TLS library versions.

4. **Q: How do you stop the impact quickly?**
   A: For your own certificate: renew and deploy. For a vendor certificate: vendor patch or workaround (sometimes a clock-based workaround isn't safe). For root changes: update client trust stores or, if possible, serve a chain the old clients accept.

5. **Q: What's the permanent fix?**
   A: Inventory and monitoring of all certificates, including embedded ones; regular image rebuilds; tested root migrations.

6. **Q: How do you confirm the fix?**
   A: Test from the oldest supported clients and from strict libraries, not just a modern browser.

7. **Q: What if you can't update the old clients (IoT devices in the field)?**
   A: Keep serving a chain they trust for as long as possible (or a separate endpoint for them), and plan device updates or replacement before the next expiry.

8. **Q: How would you prevent this class of incident?**
   A: Treat expiry dates (certificates, keys, tokens, licences, domains) as capacity items with alerts months in advance, and ask vendors about hidden ones.

9. **Q: What would you do differently?**
   A: Run a client-compatibility test suite against staging whenever the certificate chain changes.

10. **Q: Why did old Android keep working after the root expired?**
    A: Android doesn't enforce expiry dates of trust anchors, so an extended cross-sign from the expired root remained usable there. It's a clever exception, not a general rule.

## Source

- Ericsson, "Update on software issue impacting certain customers" (6 December 2018): <https://www.ericsson.com/en/press-releases/2018/12/update-on-software-issue-impacting-certain-customers>
- Let's Encrypt, "DST Root CA X3 Expiration (September 2021)": <https://letsencrypt.org/docs/dst-root-ca-x3-expiration-september-2021/>
- Let's Encrypt, "Extending Android Device Compatibility for Let's Encrypt Certificates" (21 December 2020): <https://letsencrypt.org/2020/12/21/extending-android-compatibility.html>
