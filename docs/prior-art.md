# Prior art

Almost nothing in AgentML is new. Each idea worked somewhere first. What no
existing format did was put them together in a document that a browser
renders, that a language-model agent can walk by voice or chat, and whose
meaning can be tested cheaply.

## At a glance

| Format | What it is | What AgentML takes | Why it wasn't enough |
|---|---|---|---|
| **VoiceXML** | W3C language for voice dialogues: forms, fields, grammars. | Form and field as the unit of a dialogue. `noinput` and `nomatch` as reserved attributes. | Voice only, built around recognition grammars, with no browser rendering. Its lesson is below. |
| **TwiML** | Twilio's XML of call verbs (`<Say>`, `<Gather>`), fetched from your server one document at a time. | Fetching the next document from the server as the conversation moves on. | It tells a call what to do next. It doesn't describe what the conversation has to collect. |
| **ConversationRelay** | Twilio's real-time voice layer: speech in and out, interruptions, text to and from your app over a WebSocket. | Nothing to take. It is the layer below. | Complementary: it leaves what the conversation is *for* to your code, and an AgentML page can be that part. |
| **SWML** | SignalWire's call-flow markup, including AI agents with prompts and steps. | `step_criteria` is the model for the reserved `data-complete-when`. | Prompts are instructions to the model, where AgentML keeps text as data. |
| **Dialogflow CX** | Google's conversational agents built from pages, forms, parameters and routes. | The nouns: page, form, parameter. | Lives inside Google's platform, not in a document you serve or render. |
| **Rasa CALM** | Business-logic flows with slots that an LLM fills from their descriptions. | *The description is the model's interface*: `data-description`. | Configuration for one framework, not a format a browser renders. |
| **Adaptive Cards** | JSON cards rendered by host apps, with submit and execute actions and `$when` templating. | Keeping "finish" apart from other actions: submit buttons versus named buttons. | A condition evaluated once at render time, where a conversation needs one re-evaluated after every answer. |
| **FHIR Questionnaire** | Healthcare's standard for forms (`Questionnaire`) and their answers (`QuestionnaireResponse`). | `enableWhen`/`enableBehavior` as the semantics of `data-when`. Results paired with the form they answer. | Clinical data modelling, far heavier than a booking page. |
| **XLSForm / ODK** | Survey forms written as spreadsheets, used for field data collection. | `constraint`, `constraint_message` and `calculate` as reserved attributes. | Built for surveys filled in on a device, not conversations. |
| **MCP** | Model Context Protocol: tools and resources for models. | `inputSchema`/`outputSchema` as the vocabulary of goal manifests. | Covers what a model can call, not what an agent must ask a person. |
| **A2A** | Agent2Agent: how agents delegate tasks to each other. | Task states as the model for a goal's outcome. | Agents talking to agents, not to people. |
| **WebMCP, VOIX** | Proposals for websites to declare actions and context for browser agents. | Explicit, declared affordances instead of agents guessing from pixels. | Actions on arbitrary web apps. AgentML is a whole conversation in one small profile. |
| **llms.txt** | Documentation served in a form language models read well. | Serving a view meant for agents alongside the human one. | Read-only. |

## Research that shaped the design

- **Schema-guided dialogue** (Google's SGD dataset) showed that describing a
  service's intents and slots in natural language lets one model handle
  services it was never trained on. That is the argument for
  `data-description`.
- **Genie Worksheets** (Stanford OVAL) pair a declarative worksheet of what
  to collect with code that decides what happens next, and leave the
  language model to understand and phrase. They report substantially higher
  task completion than agents that leave the whole conversation to the
  model. That split is the voice agent's
  shape: [code walks the page, the model talks](channels.md#voice).

## The VoiceXML lesson

VoiceXML is the closest ancestor and the most useful warning. It specified
its Form Interpretation Algorithm, the loop that decides which field to visit
next, in detail. Moving an application from one platform to another was
still hard in practice, and platforms grew their own extensions.

AgentML draws three rules from that:

1. **Specify meaning, not mechanics.** The interpreter contract is one
   paragraph ([SPEC §8](../SPEC.md#8-walking-a-page)). Speech recognition,
   phrasing and turn-taking belong to the consumer.
2. **Make conformance cheap to test.** The [fixtures](../conformance/) are
   pages with their expected inventories, and both existing implementations
   are tested against the same files.
3. **Reject rather than ignore.** A consumer that silently drops what it does
   not understand cannot be told apart from one that understands it, so the
   allowlist is exhaustive.

## Why HTML and not a new language

A new language needs a renderer for every medium, including the web. HTML
already has one on every device, a meaning for forms that every developer
knows, and an accessibility tree that agents already read. What it lacked
was a boundary: a small, strict subset in which the markup can be trusted to
drive an agent and the text can be trusted to be only text. That boundary is
what AgentML adds.
