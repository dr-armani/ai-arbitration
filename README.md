# AI Arbitration Platform

An automated arbitration system built in **n8n** for contracts, escrow tracking, dispute handling, AI-assisted adjudication, and post-award execution.

It combines:

- contract intake and case creation
- tokenized party actions by email
- escrow verification on Ethereum
- payment / refund request workflows
- evidence and message collection
- AI-generated case investigation and determination
- objection window and delayed award execution
- admin error reporting

The workflows are designed to run independently but share a common case ledger and file store. 

---

## What this repository contains

This repository contains the n8n workflows that implement the arbitration lifecycle:

- `Initialization.json`
- `Transactions.json`
- `Prosecution.json`
- `Adjudication.json`
- `Objection.json`
- `Exceptions.json` 

---

## How the system works

A case begins when a contract is submitted through a web form. The system creates a case record, stores the contract and related files, emails both parties secure links, and waits for acceptance and escrow funding. Once funded, either party can initiate payment or refund requests. If the parties do not resolve the matter, the case can move into dispute, AI adjudication, objection review, and finally automatic award distribution if no objection is sustained. 

---

## Architecture

### Case ledger
Google Sheets acts as the operational database for cases, messages, files, requests, determinations, and payouts. The workflows repeatedly read from and update the same arbitration sheet structure. 

### File storage
Google Drive stores case folders, contract PDFs, uploaded evidence, and generated reports. Each case is assigned its own folder. 

### Notifications
Gmail is used for party communications, confirmations, status updates, determinations, and admin alerts. The fallback error workflow also supports SMTP-based admin notification. 

### Blockchain layer
Ethereum is used for escrow verification and outbound transfers. The workflows query Etherscan for deposit history and use Tatum to broadcast outbound ETH transactions. 

### AI layer
The adjudication flow uses an AI “Magistrate Judge” to investigate the case and prepare a structured report, followed by validation and a downstream final determination stage. The investigation prompt explicitly supports file inspection, external API verification, and strict payout consistency checks in Wei. :contentReference[oaicite:8]{index=8}

---

## Core process

### Case creation
The initialization workflow starts with a public contract registration form that collects seller, buyer, their emails, escrow amount in ETH, contract text, and an optional contract PDF. After submission, it generates a case ID, creates a case folder in Google Drive, stores the case in Google Sheets, generates party tokens, and emails both parties secure action links. :contentReference[oaicite:9]{index=9}

### Contract acceptance
The same initialization workflow handles party responses through tokenized links. Each party can accept or decline the contract. A case becomes signed only when both parties accept; otherwise it remains pending or becomes declined. Duplicate or late responses are explicitly handled. :contentReference[oaicite:10]{index=10}

### Wallet registration
Parties can submit and update Ethereum wallet addresses through a dedicated form link. Submitted addresses are validated against an Ethereum address pattern before being written to the case record, and successful updates trigger confirmation emails. :contentReference[oaicite:11]{index=11}

### Escrow deposit
The transactions workflow verifies whether escrow has been deposited after both parties accept. It looks up the case, derives the relevant starting block from the case timestamp, queries Etherscan for transactions into the escrow address, sums deposits, and compares the result against the required amount. If sufficient funding exists, the case is activated and both parties receive status emails with next-step links. 

### Payment / refund requests
The prosecution workflow allows the seller to request payment and the buyer to request refund. It validates the case, link token, available balance, and request state, then presents a confirmation page. Once confirmed, the request is recorded in the database and the opposing party is notified to approve or dispute within the response window. :contentReference[oaicite:13]{index=13}

### Dispute handling
If a request is disputed, the case moves into the adjudication pipeline. The adjudication workflow periodically selects disputed cases whose deadlines have matured, locks the case for processing, gathers contract text, evidence files, and messages, and builds a structured case packet for AI investigation. :contentReference[oaicite:14]{index=14}

### AI investigation and determination
The adjudication workflow first generates a magistrate report with verified facts, contradictions, unsupported claims, reasoning, and recommended payouts. That output is validated for schema compliance, payout math, and content sufficiency. A downstream final decision is then produced, emailed to both parties, and stored in Drive. Parties are informed of a 7-day procedural objection window. :contentReference[oaicite:15]{index=15}

### Objection window and award execution
The post-decision workflow checks cases in `DECIDED: Locked`, verifies that a determination exists, confirms the 7-day waiting period has passed, validates remaining escrow and wallet availability, and then executes the award transfers. Cases with no remaining escrow or no award are closed accordingly. Successful distributions are recorded and parties are notified. Failed transfers generate admin alerts and payout reversion logic. :contentReference[oaicite:16]{index=16}

### Global exceptions
The exceptions workflow is the global error catcher. Any unhandled workflow failure is converted into a structured admin alert containing execution metadata, the failing node, the error message, raw technical context, and execution URL. If the primary Gmail alert fails, the fallback SMTP alert is used. :contentReference[oaicite:17]{index=17}

---

## Status model

Across the uploaded workflows, cases move through states such as pending acceptance, signed, requested, disputed, processing / locked, decided / locked, and closed. Some close states are specialized, including `CLOSED: No Escrow` and `CLOSED: No Award`, while dispute processing uses explicit lock states to avoid conflicting operations during adjudication or distribution. 

---

## Security model

The system relies on tokenized email links for party actions. Wallet submissions are format-validated. Case actions are tied to case IDs and party-specific tokens. Admin-facing exceptions are separated from user-facing pages. Award execution also checks that payouts have not already been recorded, reducing duplicate disbursement risk. 

---

## Financial model

All core financial values are stored and compared in **Wei**. Adjudication explicitly validates that recommended buyer and seller payouts sum exactly to the escrow balance. The transactions workflow also supports excess-fund handling, including optional seller tipping and buyer withdrawal, both recorded back into the case ledger. 

---

## External services and credentials

To run this system, you need credentials and configuration for:

- Google Sheets
- Google Drive
- Gmail
- SMTP fallback
- OpenAI
- Etherscan
- Tatum
- n8n environment variables such as `BASE_URL`, `ADMIN_EMAIL` and `PRIVATE_KEY` 

---

## Environment Variables

The system relies on several project variables configured in **n8n → Project Settings → Variables**.

These variables should **never be committed to the repository**.  
Use secure secrets management in production.

| Variable | Description | Example |
|---|---|---|
| `ADMIN_EMAIL` | Email address receiving system error alerts | `admin@example.com` |
| `BASE_URL` | Base URL of the arbitration platform used in email links | `https://arbitration.example.com` |
| `ESCROW_WALLET` | Ethereum wallet used as the escrow account | `0x0000000000000000000000000000000000000000` |
| `PRIVATE_KEY` | Private key used to sign payout transactions | `your_private_key_here` |
| `PROCESSING_FEE` | Platform processing fee (in Wei) deducted from escrow | `1000000000000000` |

### Notes

- `PRIVATE_KEY` must be stored securely and **must never be exposed publicly**.
- `ESCROW_WALLET` should correspond to the wallet derived from `PRIVATE_KEY`.
- `PROCESSING_FEE` is defined in **Wei** to avoid floating-point precision issues.

Example usage inside workflows:

```javascript
{{$vars.ADMIN_EMAIL}}
{{$vars.BASE_URL}}
{{$vars.ESCROW_WALLET}}
{{$vars.PRIVATE_KEY}}
{{$vars.PROCESSING_FEE}}
```

----

## Deployment notes

Import the JSON workflows into n8n, connect the required credentials, configure the shared environment variables, and ensure all workflows point to the same case database and storage locations. The initialization workflow is the public entry point; the others operate as background, callback, or scheduled workflows. 

---

## Operational notes

- The objection-related repository file currently combines post-determination validation and distribution logic; it is effectively the enforcement workflow after the objection window expires. :contentReference[oaicite:24]{index=24}
- The adjudication workflow is designed to process eligible disputed cases on a schedule rather than immediately at dispute creation. :contentReference[oaicite:25]{index=25}

---

## Repository goal

This project is a workflow-centric reference implementation of automated arbitration: intake, escrow, dispute resolution, AI-assisted reasoning, objection handling, and execution, all built with n8n and common SaaS infrastructure. 
