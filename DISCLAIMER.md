# ⚠️ Disclaimer & Liability Limitation

**Repository:** `n8n-clay-webhook-enrichment-credit-guard`
**Author:** DH Robin (dhRobin Systems)
**Last updated:** November 2026

Use at your own risk. The code, workflows, and scripts in this repository (`n8n-clay-webhook-enrichment-credit-guard`) are provided for educational and architectural demonstration purposes.

* **No Liability:** The author and dhRobin Systems accept no responsibility or liability for any direct, indirect, incidental, or consequential damages that result from using, misusing, or deploying these scripts. That includes, without limitation, unexpected Clay credit consumption or enrichment charges, lost or overwritten CRM data, duplicated or missed enrichments, API rate-limit penalties, account suspensions, and any business decisions made from enriched data.

* **No Warranty:** All software, n8n JSON blueprints, JavaScript modules and configuration examples are provided "AS IS", without warranty of any kind, express or implied. This includes, but is not limited to, warranties of merchantability, fitness for a particular purpose, and non-infringement. The domain blocklists, public-suffix rules, ICP defaults and credit-cost estimates are illustrative and are not guaranteed to be complete or accurate for your use case.

* **Production Testing Required:** Always test thoroughly in an isolated staging environment, such as a HubSpot developer test account or sandbox and a dedicated Clay test table, with `SANDBOX_MODE=true` and `DRY_RUN=true` enabled, before you connect live HubSpot or Clay credentials. Move through shadow mode (`SANDBOX_MODE=false`, `DRY_RUN=true`) and a low-budget canary before full production. The daily credit cap is a safeguard, not a guarantee: parallel executions, misconfigured costs or changes in third-party pricing can lead to higher spend than configured. Check actual credit usage in Clay yourself.

* **Third-Party APIs & Compliance:** You alone are responsible for complying with the terms of service, rate limits, data privacy regulations (GDPR/CCPA), and security policies of HubSpot, Clay and any connected services. That includes confirming a lawful basis for sending contact and company data to third-party enrichment providers, telling data subjects how their data is processed where the law requires it, honouring opt-outs and data-subject requests, keeping credentials out of version control, and reviewing every automatically created or updated record.

* **No Affiliation:** This project is independent. It is not affiliated with, endorsed by, or sponsored by HubSpot, Inc., Clay Labs, Inc., or n8n GmbH. All product names and trademarks belong to their respective owners and are used only to describe compatibility.

* **Fictional Sample Data:** All companies, people, email addresses, domains, object IDs and secrets in the pinned sample data and documentation are fictional and exist only for testing. Any resemblance to real entities is coincidental. Placeholder values such as `YOUR_HUBSPOT_PRIVATE_APP_TOKEN` and `YOUR_CLAY_WEBHOOK_URL` must be replaced with your own values, which you keep in environment variables or n8n credentials.

* **Not Professional Advice:** Nothing in this repository is legal, financial, or compliance advice. Consult qualified professionals for decisions in those areas.

## Open-Source License Notice

This project is released under the MIT License. You may use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the software, subject to the conditions in [LICENSE](./LICENSE). The MIT License's own warranty disclaimer and limitation of liability apply in addition to this document. If the two conflict, the terms more protective of the author apply to the extent permitted by law.

---

Full text: DISCLAIMER.md · Licensed under the MIT License. License: MIT © 2026 DH Robin (dhRobin Systems) · Provided as-is, see the disclaimer.
