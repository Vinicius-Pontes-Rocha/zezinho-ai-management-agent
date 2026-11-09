# Zezinho — AI Management Agent

> 🇧🇷 [Versão em português abaixo](#-português)

An LLM-powered management agent running in production at a Brazilian accounting firm. It reads the team's Google Chat spaces, writes a daily management report, answers @mentions, follows up on open tasks and gives feedback after meetings. **Nothing reaches the company without human approval.**

Built with **n8n** (self-hosted), **Gemini**, the **Google Chat / Drive / Docs APIs** and a **106-note Markdown knowledge base** (Obsidian) used as long-term memory.

## Architecture

```
Google Chat ──► daily digest (Drive) ──► metrics (no AI) ──► trends (no AI)
                                                         │
@mention ──► watcher (no AI, every 5 min) ──► queue ──► ONE Gemini call
                                                         │   + tools: knowledge base, history search,
                                                         │     trends, CRM funnel, attachments
                                                         ▼
                                  rule-based reviewer ──► human approval ──► publish / reject (+ reason)
```

## Design decisions

| Problem | Decision |
|---|---|
| LLM calls are the main cost | Cheap **no-AI watchers** do the polling; the LLM is called **once per mention** with a pre-built context package |
| Runaway loops can burn the budget | **Circuit breaker** every 15 min: disables a looping flow and alerts; hard cap of R$ 70/month, daily cap with queue |
| LLM output can be wrong or sensitive | **Rule-based reviewer** + **double human approval** before anything is posted company-wide; rejections store the reason |
| Need to know if it is getting better | **Eval set ("provão")** extracted from real questions the team already answered + **LLM-as-judge** score tracked weekly |
| Context window and cost | Only the relevant sector note and preferences are sent, not the whole knowledge base |
| Silent failures | Error workflow alerts the owner by DM, rate-limited to 1 alert per flow per hour; a watchdog checks the daily report was produced |

## Numbers (Oct 8–28, 2026)

- 22 workflows in this repo, part of 36 in production
- ~R$ 0.01–0.03 per LLM call
- 3,788 executions across the instance, 99.4% success

## Workflows

| File | What it does |
|---|---|
| `01-daily-chat-digest` | Summarizes every chat space into a daily Doc |
| `02-daily-metrics-no-ai` | Counts messages and reports per person/space |
| `03-mention-watcher-no-ai` | Finds new @mentions and queues them |
| `04-mention-responder-single-llm-call` | Builds the context and calls Gemini once; classifies report / question / record |
| `05-human-approval-flow` | Approve / reject buttons in Chat, publishes on approval |
| `06`–`16` | Agent tools (send for approval, read messages, attachments, knowledge base read/write, history search, trends, CRM funnel, save report, DM) |
| `17-pending-task-follow-up` | Follows up on tasks open 2+ business days (with approval) |
| `18-meeting-feedback` | Post-meeting feedback from the minutes |
| `19-crm-won-deal-handoff` | Won deal in the CRM → new-client handoff to Customer Success |
| `20-alert-report-missing` | Watchdog for the daily report |
| `21-eval-build-test-set` | Builds the eval set from real answered questions |
| `22-weekly-evolution-summary` | Weekly approved/rejected stats, judge score, cost |

## How to import

n8n → *Workflows* → *Import from file*. Credentials, IDs, e-mails, URLs and names were replaced with placeholders (`<FOLDERID>`, `<CREDENTIAL_ID>`, `Redacted`, etc.); set your own before running.

---

## 🇧🇷 Português

Agente de gestão com LLM rodando em produção numa contabilidade. Lê os espaços do Google Chat, escreve o relatório diário de gestão, responde @menções, cobra pendências e dá feedback das reuniões. **Nada sai para a empresa sem aprovação humana.**

**Destaques:** vigias sem IA + **uma** chamada ao Gemini por menção; disjuntor de custos (teto de R$ 70/mês); revisor por regras + aprovação dupla; conjunto de avaliação ("provão") com perguntas reais + LLM como juiz; base de conhecimento de 106 notas como memória.

Os workflows foram sanitizados: credenciais, IDs, e-mails, URLs e nomes de pessoas/clientes foram trocados por marcadores.
