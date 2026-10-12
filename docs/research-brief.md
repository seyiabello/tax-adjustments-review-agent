# Research brief: corporation tax adjustments

Last updated: 12 October 2026.

## The problem in one sentence

When preparing a UK company's corporation tax computation, tax staff go line by line through the trial balance and nominal ledger to find disallowable and capital items to add back, and mid-tier firms and their SME clients have no tool that does a trustworthy first pass with HMRC citations.

## Why it exists

- Accounts profit is not taxable profit. Items like client entertaining (BIM45020), fines, capital spend, depreciation and some legal fees must be adjusted. The rules sit in HMRC's Business Income Manual, Company Taxation Manual and Capital Allowances Manual.
- Ledgers are messy. The same expense type is coded differently by every client, and descriptions are short.
- It is classic junior work. Accountancy bodies have warned that trainees who stop doing routine ledger work may not build the instinct for spotting errors, so a tool that explains its reasoning also helps juniors learn.
- Computation software calculates tax once adjustments are keyed in. It does not identify them from ledger lines.

## Evidence of demand

- KPMG UK built a bespoke AI solution for a law firm client's annual tax computation. It classifies raw ledger lines with explanations in 10 to 20 seconds, reached 95% accuracy in pilot and saves 4 to 6 weeks. That was one large client.
- KPMG UK lists initial reviews of tax returns and anomaly detection in tax codes on transactions as current AI uses, with return preparation still a gap.
- UK accounting firms are building internal AI engineering teams for tax use cases, and computation software vendors describe "trusted intelligence" as their next stage.

## Current solutions and gaps

| Approach | Gap |
|---|---|
| Manual review in Excel or working papers | Slow, inconsistent, depends on reviewer experience |
| Computation software | Calculates tax once adjustments are entered; does not find them |
| Bespoke Big 4 builds | One client at a time; not available to mid-tier firms |
| General AI assistants | Can cite blogs and repeat web errors; no audit trail |
| Tax research AI | Answers questions; does not work through a ledger |

## The constraint that shapes the design

HMRC's guidance on generative AI in tax software (January 2026) sets five principles: transparent, legally grounded, human-supervised, secure and ethical. Acceptable sources are HMRC publications and manuals, legislation and case law. Software should prompt human review, allow corrections, flag complex cases and remind users they remain responsible for their return.

## What Addback does differently

- Works through a whole ledger, not a single question.
- Cites a specific HMRC manual paragraph for every proposal, and checks that the citation supports the claim.
- Sends anything uncertain to a human, and logs every decision.
- Built for the mid-tier and SME market rather than one large client.

## Sources

- KPMG UK, DAC Beachcroft AI tax automation: https://kpmg.com/uk/en/insights/tax/dac-beachcroft-ai-tax-automation.html
- KPMG UK, AI tax tech landscape: https://kpmg.com/uk/en/insights/tax/ai-tax-tech-landscape.html
- HMRC guidance on generative AI in tax software (summary): https://vatcalc.com/united-kingdom/uk-hmrc-guidance-generative-ai-in-tax-software
- ATT, the hidden risk of AI (May 2026): https://www.att.org.uk/technical/news/hidden-risk-ai-why-you-must-still-check-tax-sources
- Accountex London 2026 panel (Accountancy Age): https://accountancyage.com/2026/05/18/accountex-london-2026-how-accountancy-bodies-are-turning-the-ai-threat-into-a-strategic-advantage/
- Alphatax rebrand (IT Brief UK): https://itbrief.co.uk/story/tax-systems-rebrands-as-alphatax-in-ai-tax-compliance
- HMRC BIM45020: https://www.gov.uk/hmrc-internal-manuals/business-income-manual/bim45020
