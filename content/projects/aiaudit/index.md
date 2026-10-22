+++
date = '2026-10-22'
draft = false
title = 'AIAudit: an AI assistant for compliance audits'
description = 'An agent that maps evidence to controls, spots gaps and drafts findings for ISO 27001, GDPR and QNRCS audits. The auditor reviews and decides.'
tags = ['AI', 'Compliance', 'ISO 27001', 'GDPR', 'Idea']
+++

**Status: idea, in the making**

## Pitch

An agent that helps me run compliance audits (ISO 27001, GDPR, ...) faster. It maps
evidence to controls, spots gaps, and drafts findings. I review and decide. Name pending 🙃 improvement. 

## Standards in scope

- ISO 27001
- GDPR
- QNRCS: DL 125/2025 and Regulamento n.º 756/2026

## Audit workflow

| Step | AI helps by... |
|------|----------------|
| Evidence request | Asking for specific missing documents |
| Evidence review | Parsing the evidence and checking it matches what was asked |
| Control mapping / gap analysis | Mapping evidence to specific controls |
| Interviews | Writing the meeting minutes |
| Findings & report | Drafting the report with the analysis |

## Inputs

Evidence is triaged by an AI or a human, then bulk-uploaded. The AI maps which
evidence answers which control, and a human validates the mapping.

## Outputs

An analysis per control, for a human to validate.

## How it works

In the making.

## Rules for the AI

- Cite the evidence for every claim.
- Don't hallucinate.
- Help, don't create more work.

## Data & confidentiality

Client data is sensitive. Either a local LLM or some form of anonymization
before anything leaves the machine.
