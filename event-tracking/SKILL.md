---
name: event-tracking
description: Use when instrumenting code for behavioral analytics, whether adding, changing or auditing tracked events and their properties, with any analytics tool.
metadata:
  sources: >-
    PLGeek's Event Tracking Style Guide (written for Segment and Amplitude Data), made tool-agnostic,
    and Ben Williams, "The importance of account level analytics in B2B" (plg.news, June 2023).
    Vendor notes checked against Amplitude, Mixpanel, PostHog, Segment and GA4 docs in November 2026.
---

# Event tracking

A **tracking plan** is the story of what users do in the product. Many people add to it in small
pieces over time. It stays readable only if every event is written in the same voice, so a person
or an automation can guess an event's name, trust what it carries, and join it to the events
around it. Every rule below serves that, whatever tool receives the events.

The plan is the single source of truth. Code implements the plan, and the analytics tool mirrors
it. A tracked call that isn't in the plan, or that differs from it, is a bug.

## Steps

### 1. Survey what exists

Find, and write down before designing anything:

- **Tools.** Every analytics SDK, CDP (Segment, RudderStack) and tag manager in the code, and
  every place each is initialised. Compare the inits: the client and server must send to the same
  project, key and region or host. A mismatch quietly splits one user's events across two
  projects. Then read that tool's section in
  [references/tools.md](references/tools.md). It maps each rule below onto the tool and covers the
  tool's naming limits and debug mode.
- **Plan.** Where the tracking plan lives today: a file in the repo, a governance tool (Amplitude
  Data, Avo, Segment Protocols, Mixpanel Lexicon, PostHog schema management), a spreadsheet, or
  nowhere. Generated tracking code (Ampli, Avo Codegen, PostHog schema types) means the plan lives
  in that tool. Edit the plan there and regenerate; hand edits to generated files are lost.
- **Calls.** Every existing tracking call (`track`, `capture`, `logEvent`, `gtag('event'`,
  `analytics.`), and what autocapture, pageview and session-replay settings collect on their own.
  For each call, note its event name, properties, file and whether it runs client- or server-side.
- **Identity.** Which ID identifies users on the client and on the server, where identify is
  called, and whether both sides use the same ID.
- **Accounts.** Whether the product is B2B: do users work inside a team, workspace, org or
  company? If so, which entity is the account (see [Accounts](#accounts-b2b)), whether a user can
  belong to several, and how the code knows which one is active on each request and screen.
- **Consumers.** Dashboards, automations, CRM or email triggers, and warehouse models that depend
  on current event names. Renaming breaks them silently. Ask the user about any you can't see
  from the code.

Done when you can list every tracking call in scope with its name, properties and runtime, and you
know where the plan lives.

### 2. Start from the questions

Write down the product or marketing questions the instrumentation must answer ("Which onboarding
step loses the most new accounts?"). Mark which ones the user stated and which you inferred. Every
event and every property must serve at least one of them. If the request names none, infer them
from the feature.

For each question, trace the outcome it needs to the code that produces it. When that code isn't
in this repo (billing in another service, a payment provider's webhook), plan the event anyway:
give it `status: planned`, ship its typed function, and report the question as unanswerable until a
human wires it up. An outcome event fires where the outcome happens. A click on the way there
doesn't stand in for it. When the product never captures a value a question needs (a form field
that never reaches the server), report that gap rather than inventing the value.

In B2B, the questions that matter most are about accounts, not users: how many accounts have
adopted a feature, how long new accounts take to activate, account retention at 7, 30 and 90
days, which behaviour predicts an account paying. Include them. Sales (product-qualified account
scoring), customer success (health scores), marketing (channel performance) and support all ask
questions about accounts too, so ask the user which of those teams will use the data.

The questions are also the privacy test (see [Personal data](#personal-data)). New data that
answers no question is not collected. An existing event that answers none is reported as a
candidate to retire; it isn't silently kept or silently deleted.

### 3. Design the plan entries

When existing events follow a different naming scheme, or several, ask the user before naming
anything: adopt this scheme for new events, run in parallel with the old ones (recommended), or
migrate everything now. A project that mixes schemes with no versioning is the outcome to avoid.

For each event, write a plan entry before touching code: name, description, source, status, where
it fires, and each property with its type, required flag and description. Use the format in
[assets/tracking-plan.yaml](assets/tracking-plan.yaml) when the repo has no plan yet. If the plan
lives in a governance tool, draft the entries there or hand them to the user to apply.

Check every entry against every rule in [The rules](#the-rules). Done when each event and property
has a description and passes every rule, or carries a written reason for the exception.

### 4. Implement through one typed module

Send every event through one tracking module per runtime, legacy events included, so the module
and the plan between them list everything that is sent. The module exports the event names as
constants and types each event's properties: a typed function per event, or a typed
`track(name, props)`. Event names never appear as string literals at call sites. With codegen
(Ampli, Avo, PostHog schema types or `createTypedPostHog`), the generated client is that module.

This keeps the plan enforceable in tools that never validate a schema, and it means a rename
happens in one place. When an enum's values already come from a type in the code (a union, an
enum, a config list), derive the property's type from it so a new value can't drift. When the
values live only in markup, write the enum in the module instead of refactoring product code for
it.

In B2B, the module attaches the active account to every call itself, read from the current
workspace context on the client and from the request on the server, so no call site can forget
it or send a stale one.

Place each event on the web or the backend by the [placement rules](#placement-web-or-backend).

### 5. Verify the payloads

Run the code paths and read the events as sent. For each event, check:

- It fires once per real occurrence, from the planned runtime.
- Its name and every property name and type match the plan exactly.
- Required properties are present. No property, URL, element text or identify trait holds
  personal data.
- Client and server events for the same user carry the same user and account IDs, and reach the
  same project.
- In B2B, every event carries the account it happened under. Where a user can belong to several
  accounts, switch accounts and check that the next events carry the new one.

A unit test asserting on a mock client verifies the code. Only the tool's debug mode or live view
(see [references/tools.md](references/tools.md)), or the network panel, verifies delivery. Done
when every implemented event has passed both, or the report says which ones were checked only in
code and why (no credentials, no test environment).

### 6. Report

Point to the plan file or governance tool for the entries, rather than restating them. Report:

- The events added, changed and retired.
- The questions they serve, marked stated or inferred, and any question still unanswerable.
- Rule exceptions, with reasons.
- What was verified only in code.
- Steps left for a human: governance tool edits you could not make, consumers to update, planned
  events to wire up, old events to retire, personal data already stored that should be deleted,
  and, in B2B, an analytics tool that can't analyse at the account level (see
  [Accounts](#accounts-b2b)).

## Auditing or migrating existing tracking

When the work is an audit, or the existing tracking breaks these rules, also:

1. Check each existing event against every rule. Report one table, a row per event: the rules it
   breaks and the planned fix.
2. Take personal data out of legacy events straight away. The name and the remaining properties
   stay unchanged, unless the name itself carries personal data (an email or a customer name built
   into it). Stop sending that event now, whatever depends on it, and list its consumers in the
   report.
3. When names must change, add the new event **in parallel** with the old one. Keep both firing
   until every consumer found in the survey has moved, even when the old name breaks the rules. Renaming in
   place breaks history and the dashboards built on it, with no error. The `Is`/`Are` linking
   verb is what makes running both safe: `Account Is Created` can never collide with a legacy
   `Account Created` or `account_created`, so the two versions stay apart while both are live.
   Mark each legacy entry in the plan with `status: deprecated` and the event that replaces it.
4. Retiring an old event is the user's call. Name its consumers and ask.

## The rules

### Personal data

Champion users' privacy. Personal data that never enters the pipeline can't leak, and it never
needs deleting from five destinations later.

- **Do you need it?** Collect personal data only when a stated question cannot be answered
  without it. Leaving it out at the source beats filtering it out downstream.
- Use internal IDs, never the values themselves. Email addresses, names, phone numbers, free-text
  input, search strings, file names and URLs carrying tokens or emails stay out of properties,
  identify traits and user IDs.
- **Check what is collected automatically.** Pageview URLs, autocaptured element text and session
  replay record whatever is on the page. A form that submits by GET puts its fields into the URL.
  Mask or exclude these at the SDK, and make sure forms post.
- **Minimise the spread.** A CDP or SDK may forward events to many destinations. Check where these
  events go, and send customer data to no more third parties than the questions need.
- Company data-handling policy overrides this guide. When the repo or the user names one, follow
  it.

### Event names

**Subject + linking verb + past-tense action**, in Title Case:

- **Subject**: the singular or plural noun for the record or thing acted on, usually matching the
  record type in the code (`Account`, `User`, `Project`, `Projects`, `Email`).
- **Linking verb**: `Is` for a singular subject, `Are` for a plural one.
- **Action**: a past-tense verb for what happened.

Examples: `Email Is Sent`, `Projects Are Created`, `Account Is Upgraded`.

This structure lets anyone predict an event's name from the behaviour, and it groups an object's
events together when the list is sorted. It also keeps new events apart from older ones during a
migration (see above). A tool that can't store Title Case with spaces gets the same words in its
own form (GA4: `account_is_upgraded`). See [references/tools.md](references/tools.md).

**Choose the verb deliberately.** It states what the user's action was. Use the standard verbs:

| Runtime | Verb | Fires when |
|---|---|---|
| Web | `Clicked` | A CTA, button or link is clicked |
| Web | `Sent` | A form submission is sent to the server |
| Backend | `Created` | A new record (the subject) is created |
| Backend | `Invited`, `Generated`, `Opened`, `Upgraded`, ... | A record-specific action is processed |

On the web, the subject is the thing acted on (`Signup Form Is Sent`). When many similar elements
share one purpose, name the element class and put which one in a property, so the event list
doesn't grow with every new button.

**No parameters in names.** Variable content in a name splits one behaviour across many event
types. That fills the namespace, breaks charts and automations keyed on a name, and uses up tool
limits on event types. Move the variable into a property:

| Wrong | Right |
|---|---|
| `100 Emails Sent` | `Emails Are Sent` with `countSent: 100` |
| `User Created 10 Projects` | `Projects Are Created` with `countProjects: 10` |
| `Acme Imported Projects` | `Projects Are Created` with `customerId: "<id>"` |

The variable becomes an ID or a count, never a name. When the code has only a customer's name, use
an ID it already holds or leave the property out.

### Properties

Give each event a set of **flat, top-level properties** carrying the context its questions need.
Most analytics tools can't filter or group by values nested inside an object.

Name properties in camelCase. Prefix counts with `count` (`countProjects`) and booleans with `is`
(`isReadOnly`). End IDs with `Id` (`accountId`). A property means the same thing, with the same
type, on every event that carries it.

**Choose each type** by three questions. How well do we know the values in advance? How often will
the set of values change? How much control do we want over what gets sent?

| Type | Use when | Examples |
|---|---|---|
| String | Text whose values change over time, or have no manageable set | `accountId`, `featureType` |
| Enum | Every possible value is known, and this code controls the set | `scope`, `projectType` |
| Number | A numeric value. When the unit matters, add a companion enum or string giving it | `countProjects`, `sectionNumber` |
| Boolean | A true on/off state. Prefer a type with more context where one fits (`accessLevel: "read"` over `isReadOnly: true`) | `isReadOnly` |
| Const | A static, hard-coded value | `eventSource` |
| Array | Several values of one type. Type the items; set min/max items and uniqueness where the tool allows | `enabledFields`, `selectedIntegrations` |
| Object | Avoid. Keys and values can't be typed, so arbitrary data collects in them. Avoid objects as array items too | |

Arrays and objects need more care, and many tools hide them from analysis. When a tool supports
arrays of objects for item-level analysis (cart contents, for example), use one only for a
question that needs it, and name that question in the property's description.

**Balance integrity against flexibility.** A tighter schema gives cleaner data but costs upkeep.
Take an `integration` property. As an enum it rejects junk like `"Hello World"`, but every team
that adds an integration must also update the schema, and with many teams in one codebase they
won't know to. As a string it accepts anything. Pick enum when the values are owned here and
change rarely, or when the property is derived from the code's own type (step 4). Pick string
otherwise.

**Property groups.** Define a group (a shared type in code, and a property group in tools that have
them) when two or more properties go on more than one event, or when one complex property may
spread to more events. The group is then managed in one place.

### Descriptions

Every event and every property gets its own description. An event's description says why it
exists, where it fires (the screen, the component or the endpoint) and how it behaves in ways that
matter to analysis: whether it fires once or on every retry, and what the user did just before it.
A property's description says what the value means and where it comes from. Push descriptions into
the analytics tool's catalogue too, so analysts see them where they work.

### Sources

A **source** is one runtime or codebase that sends events: the web app, the marketing site, the
API, an iOS app.

- Give each separate runtime or codebase its own source. When a source sends a different set of
  events from the rest of the plan, that is the cue to split it off. Schema validators that check
  a source against its expected events fail when the two drift.
- Assign one event to several sources rather than defining it twice. When a source needs extra
  properties, first try to make them generic enough for every source. Define a distinct event only
  when no reasonable abstraction exists.
- In tools without a source concept, send a const `eventSource` property on every event, with the
  source's ID as its value. SDK-level fields such as PostHog's `$lib` name the library, not the
  codebase: two Node services both report `posthog-node`.

### Placement: web or backend

Ad blockers and privacy extensions drop client-side events. Expect up to about 30% less volume on
the web. Server-side events always arrive but sit further from what the user actually did. A server
can't tell which of several CTAs was clicked, has no browser context (user agent, location) and
can't see page views accurately.

| The event... | Send from |
|---|---|
| Triggers a product or marketing automation (email, CRM, billing, lifecycle) | Backend, always |
| Records an outcome on a record (created, upgraded, invited) | Backend |
| Captures intent or UI context only (which CTA, which page, what was viewed) | Web |
| Is intent that leads to a server outcome | Both: a web event for the intent (`Signup Form Is Sent`) and a backend event for the outcome (`Account Is Created`) |

When both fire, give them the same user and account IDs so they can be joined, and let the
backend event alone drive automations. This rule covers automations that a tool or destination
triggers from an event. An automation the code calls directly (a handler that sends the welcome
email itself) needs no event to drive it. Track its outcome (`Email Is Sent`) only when a question
needs it.

### Identity

Events from the web and the backend describe one user only if they carry the same identifiers.

- **The user ID** is the stable internal ID of the user record: the database primary key, never
  an email address or username. Server events send it as their user ID, and the client identifies
  with the same value.
- **Identify on the client** as soon as the user is known: after signup and login, and again
  whenever a signed-in session is restored on page load. Events sent before signup stay anonymous
  until then. Most tools merge them into the user on the first identify, so identify once, with the
  real ID. Reset identity on logout.
- **The account** in B2B follows [Accounts](#accounts-b2b).
- Identify and group traits follow the same personal-data rules as properties.

### Accounts (B2B)

In B2B, every user event happens inside a team, workspace, company or account. Users work
together, and different users complete different steps of one account's journey: one connects the
integration, another gets the value. User-level analytics alone can't say whether an account
activated, retained or is ready to buy, so without the account on every event the most important
B2B questions go unanswered.

- **Associate each event with an account, not the user.** A user can belong to several
  workspaces, and the account in force changes when they switch. Every event carries the account
  it happened under. A user property naming "their" account records only the last one set and
  misattributes everything else.
- **The account is the entity the business sells to and measures:** usually the workspace or org
  that holds the subscription. When there are several levels (workspaces inside a company), track
  each as its own group type where the tool allows, and name each in the plan.
- **Send it two ways.** Use the tool's group call (Segment `group`, Amplitude `setGroup`, PostHog
  `group`, Mixpanel `set_group`), which account-level analysis depends on. Also put an `accountId`
  property on every event, so a warehouse, or a tool without groups, can still roll events up by
  account. Server SDKs don't remember the group, so every server call names it explicitly.
- **Account properties live on the account.** Plan, seat count, created date, lifecycle stage and
  acquisition channel are set through the group call's traits and updated whenever they change,
  rather than copied onto each event. The company name is fine as a trait. People's names and
  emails are not.
- **Plan the account milestones.** Track the primary KPIs at the account level: signups (distinct
  accounts), activation (the setup, aha and habit moments), engagement and monetisation. Each
  milestone needs an event that carries the account, so a funnel can count the accounts that
  reached it, whichever user took the step. Keep the user-level events too. Each perspective
  explains the other: 800,000 weekly active users means one thing spread across 9,000 accounts and
  another across 1,000.
- **The tool must analyse accounts, not only group users by them.** Grouping user analytics by
  account still gives user retention within each account. Account retention needs a tool that
  treats an account's users as one. In some tools this is a paid add-on (Amplitude Accounts,
  Mixpanel Group Analytics, PostHog group analytics). Without it, B2B analysis is severely
  impaired. Send the group calls anyway, and report the missing capability as a step for a human.
