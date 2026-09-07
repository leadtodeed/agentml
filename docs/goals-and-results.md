# Goals and results

*Draft. Goal manifests, composing a manifest from a page, per-answer
confidence and the voice agent's answer events are built. The result
document and signed goal events are planned, and their shapes below may
change.*

## Why goals are separate from pages

A page changes with every request: today's available days, this caller's
number, this business's own questions. What the conversation must bring back
does not change: a day, and a number to ring. That stable part is the
**goal**.

Keeping the two apart is what lets one goal be met by a web form, a phone
call or a WhatsApp thread without three descriptions of the same thing. A
consumer is given a goal and a page. The page says what to ask, and the goal
says what counts as done.

## Goal manifests

Each producer publishes the goals it can be asked to achieve at
`GET /goal-types`. There is no central registry: publishing manifests is
part of being a producer, so consumers discover goals instead of being
configured with them.

```json
{
  "type": "callback_booked",
  "version": 1,
  "title": "Book a callback",
  "description": "Agree a time for us to ring the caller back, and the number to ring.",
  "inputSchema": {
    "type": "object",
    "required": ["team"],
    "properties": {
      "team": {
        "type": "string",
        "description": "Which team the caller is being booked with. Determines whose availability is offered."
      },
      "client_number": {
        "type": "string",
        "description": "The number the caller was reached on, in +E.164. Prefills the page and may be changed by the caller."
      }
    }
  },
  "outputSchema": {
    "type": "object",
    "required": ["slot", "client_number"],
    "properties": {
      "slot": {
        "type": "string",
        "format": "date-time",
        "description": "The instant the caller chose, RFC 3339. Must be one of the offered options: a time they said in words is not a booking until it matches a slot we offered."
      },
      "client_number": {
        "type": "string",
        "description": "The number to ring at that time, +E.164. Defaults to the number they called from."
      },
      "message": {
        "type": "string",
        "description": "Optional context for whoever takes the call."
      }
    }
  }
}
```

- `inputSchema` is what the producer needs to compose a page. `outputSchema`
  is what a completed conversation must return. The names are MCP's, so a
  manifest can be exported as a tool definition without a translation layer.
- A reference such as `callback_booked@1` pins the version. A bare
  `callback_booked` means the latest.
- **A goal asks only for what the person can tell you.** This one has no
  agent, meeting id or confirmation number, because the conversation cannot
  supply any of them. The producer creates those afterwards.

## Composing a manifest from a page

When a page adds its own questions, the producer can publish a manifest
composed for that page. Each question extends `outputSchema`:

| Field on the page | Property in `outputSchema` |
|---|---|
| text | `{"type": "string", "description": "<label> — <data-description>"}` |
| choice | as for text, plus `"enum": [<option values>]` |
| with `data-constraint` | plus `"x-constraint": "<constraint>"` |
| `required` | its name appended to `required` |

The composed manifest keeps the goal's `version` and adds `form_id` and
`form_version`. `type@version` pins the shape of the contract, and the form
fields pin one business's content. A business editing its questions does not
create a new version of the goal.

## Results

**The form submission is the result.** The producer validates it against the
inventory of the page it served ([SPEC §9](../SPEC.md#9-submitting)). When the
POST succeeds, the goal was met.

Store the page the answers were given against, not a pointer to whatever the
page says today. Six months later, an answer is only interpretable next to
the exact question it answered.

### Confidence

Every answer carries one of three values:

| Confidence | Meaning | Example |
|---|---|---|
| `verbatim` | The person said or typed it, or confirmed a prefilled value. | "Friday", matched to the Friday option. |
| `inferred` | The agent decided it from what was said. | A postcode rebuilt from a misheard reading. |
| `absent` | The question was asked and nothing usable came back. | An optional question the caller did not answer. |

It is an enumeration, not a probability. Nobody has a calibrated number for
most of these decisions. What a reader needs later is the difference between
"the caller said this" and "the agent decided this", and between "not asked"
and "asked, no answer".

A browser always submits `verbatim`. An agent sends the other two as
`__confidence[<name>]` alongside the answers.

### Provenance and events

Each answer cites the element it came from. The voice agent publishes an
event per answer, and one when the form is done:

```json
{"type": "agent.field.answered", "data": {
  "field_id": "slot-0", "name": "slot", "value": "2026-09-18T23:59:00+01:00",
  "confidence": "verbatim", "normalised": "", "external_ref": "",
  "source": {"page": "https://example.test/p/example-legal?view=agent", "element_id": "slot-0"}}}
```

```json
{"type": "agent.form.completed", "data": {"submitted": true,
  "answers": {"slot": "2026-09-18T23:59:00+01:00", "client_number": "+447700900123"}}}
```

```json
{"type": "agent.form.completed", "data": {"submitted": false,
  "missing": ["Number to call you on"]}}
```

`normalised` is the value in canonical form when it differs from what was
heard, such as a postcode with its space restored. `external_ref` is an id
the value resolved to elsewhere, such as an address record.

### Result document (planned)

A typed document, keyed by field name, delivered as a signed `goal.achieved`
or `goal.not_achieved` event. It pairs with the page the way FHIR's
QuestionnaireResponse pairs with its Questionnaire:

```json
{
  "goal": "callback_booked@1",
  "status": "achieved",
  "result": {
    "slot": {
      "value": "2026-09-18T23:59:00+01:00",
      "confidence": "verbatim",
      "verbatim": "Friday's good",
      "source": {"page": "https://example.test/p/example-legal?view=agent", "element_id": "slot-0"}
    },
    "client_number": {
      "value": "+447700900123",
      "confidence": "verbatim",
      "source": {"page": "https://example.test/p/example-legal?view=agent", "element_id": "client_number"}
    }
  }
}
```

`verbatim` holds the person's words when the value was derived from them.

## Alignment

These are deliberate seams with existing standards. None of them is built
yet.

- **MCP tools.** A manifest uses MCP's `inputSchema`/`outputSchema`, so a
  goal can be offered to another agent as a tool whose call means "have this
  conversation". A page's named buttons map onto tools in the same way.
- **MCP Apps.** A page is a self-contained HTML document with no scripts. It
  can be shown as a UI resource to a person supervising or reviewing a
  conversation.
- **FHIR Questionnaire.** `data-when` maps onto `enableWhen` and
  `enableBehavior`, and the result document onto QuestionnaireResponse.
- **A2A.** A goal instance maps onto a task, and achieved or not achieved
  onto its terminal states.

In one line: *A2A standardises how agents talk to each other, MCP what tools
they use, and AgentML what an agent sent to talk to a person must come back
with.*
