# PLGeek skills

Agent skills from [The Product-Led Geek](https://www.plg.news) by Ben Williams. They give coding
agents such as Claude Code, Codex and Cursor the product-led growth practices I use with the teams
I advise.

## Install

```bash
npx skills add devtunehq/plgeek-skills
```

That lists the skills in this repo and asks which agents to install them for. To install one
skill globally without the prompts:

```bash
npx skills add devtunehq/plgeek-skills --skill event-tracking --global --yes
```

Each skill is a folder with a `SKILL.md`, so you can also copy a folder into your agent's skills
directory by hand.

## Skills

| Skill | What it does |
|---|---|
| [event-tracking](event-tracking/SKILL.md) | Instruments a product or website for behavioural analytics, or audits and fixes the tracking it already has, so the tracking plan reads as one story whatever tool receives it. |

### event-tracking

Ask your agent to "instrument signup and upgrade", "audit our analytics" or "add tracking for the
new export feature", and the skill takes over whenever the work touches analytics. Your agent
will:

- **Start from the questions** the data must answer, and leave out anything that answers none of
  them, personal data included.
- **Name events in one voice:** `Subject Is/Are Past-tense-verb` (`Projects Are Created`), with no
  variable content in event names.
- **Keep properties flat, typed and described**, using a table for choosing between string, enum,
  number, boolean, const and array.
- **Place each event deliberately.** Automations trigger from the backend, because ad blockers drop
  web events. Web intent is paired with its backend outcome.
- **Get B2B right:** every event carries the account it happened under, and signups, activation,
  engagement and monetisation are measured per account as well as per user.
- **Send everything through one typed tracking module** backed by a tracking plan, and check the
  payloads before calling the work done.
- **Migrate safely:** new events run alongside the old ones, so dashboards built on the old names
  keep working while you move them.

It works with Amplitude, PostHog, Mixpanel, Segment, RudderStack, GA4 and Heap, with notes on how
each one differs. For any other tool it lists the questions to answer from that tool's docs.

## Licence

[CC BY 4.0](LICENSE). Use and adapt these skills freely, including commercially, and credit
"Ben Williams, The Product-Led Geek (plg.news)".

## Work with me

These skills distil what I help product-led companies do. To work on your growth, analytics or
activation together, visit [plgeek.com](https://plgeek.com).
