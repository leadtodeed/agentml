# AgentML 0.1

**Status: Draft, 17 September 2026.** Parts of this document describe
behaviour that is running in production. Other parts are reserved for later
versions. [§14](#14-implementation-status) says which is which.

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be read as
described in [RFC 2119] and [RFC 8174] when, and only when, they appear in
capitals.

---

## Contents

1. [Introduction](#1-introduction)
2. [Terminology](#2-terminology)
3. [Serving a page](#3-serving-a-page)
4. [The profile](#4-the-profile)
5. [The inventory](#5-the-inventory)
6. [Prose](#6-prose)
7. [What the page already knows](#7-what-the-page-already-knows)
8. [Walking a page](#8-walking-a-page)
9. [Submitting](#9-submitting)
10. [Text is data](#10-text-is-data)
11. [Draft and reserved attributes](#11-draft-and-reserved-attributes)
12. [Security considerations](#12-security-considerations)
13. [Versioning](#13-versioning)
14. [Implementation status](#14-implementation-status)
15. [References](#15-references)

---

## 1. Introduction

AgentML is a profile of HTML: a small subset of HTML forms, plus a handful of
`data-*` attributes. A page written in it can be shown to a person in a
browser and walked by an AI agent in a conversation, whether by voice, in a
chat or in some other medium.

A page is the script of a conversation. Its headings and paragraphs are what
the agent may say. Its form controls are what the agent asks. Its submit
button is how the agent finishes. The answers come back as the same form post
a browser would have sent.

This document follows five rules:

1. **One document for every medium.** A page renders in any browser with no
   AgentML-specific code. There is never a second description of the
   questions to keep in step with the first.
2. **The allowlist is exhaustive.** Anything this document does not name is
   rejected, not ignored. A consumer that silently drops what it does not
   understand cannot be tested for conformance.
3. **Markup drives, text informs.** The structure decides what an agent asks
   and does. The text is material the agent may say. Nothing in a page's text
   instructs the agent ([§10](#10-text-is-data)).
4. **The page is a form. The system is goal completion.** What a conversation
   must bring back is a *goal*, described apart from any page
   ([goals and results](docs/goals-and-results.md)). A page is how one
   producer asks for it.
5. **The interpreter fits in a paragraph.** This document says what a page
   means. It does not say how to recognise speech, phrase a question or manage
   turn-taking ([§8](#8-walking-a-page)).

## 2. Terminology

- **Page**: an HTML fragment that conforms to [§4](#4-the-profile).
- **Served document**: a complete HTML document carrying a page between
  markers ([§3.1](#31-fragment-and-served-document)).
- **Producer**: whatever serves pages and receives their submissions.
- **Consumer**: whatever walks a page with a person, such as a voice agent or
  a chat agent. A browser is also a consumer, and needs nothing from this
  document.
- **Validator**: checks a page against [§4](#4-the-profile).
- **Person**: the human a consumer is talking to.
- **Field**: one addressable item a page declares
  ([§5](#5-the-inventory)).
- **Inventory**: the ordered list of a page's fields.
- **Prose**: the text a consumer may say or send ([§6](#6-prose)).
- **Medium**: how the consumer reaches the person: a phone call, a chat
  thread, a web page and so on.

## 3. Serving a page

### 3.1 Fragment and served document

A producer MAY serve a page on its own, as `text/html`. It MAY instead serve
the page inside a full document, between the markers `<!--agentml-->` and
`<!--/agentml-->`. Everything outside the markers (doctype, `<head>`,
stylesheet) is how the page looks on the web, and is not part of the page.

```html
<!doctype html>
<html lang="en">
<head><title>Book a callback</title><style>/* the web's rendering */</style></head>
<body>
<!--agentml-->
<section id="booking"> … </section>
<!--/agentml-->
</body>
</html>
```

A consumer MUST take the page from between the first pair of markers when a
response has them, and MUST otherwise treat the whole response body as the
page. A consumer MUST also accept `<!--callscript-->` and `<!--/callscript-->`.
Those are the markers used by the first implementation, which called the
profile a "call script".

### 3.2 The agent view

A producer MAY serve a different document to people at the same URL, such as
a calendar where the page offers a list of days, as long as both documents
post the same fields to the same handler. A producer that does this:

- MUST serve the page when the request's query string carries `view=agent`;
- MAY also recognise consumers by their `User-Agent`;
- SHOULD serve the human document to a `User-Agent` it does not recognise. A
  person served the page still gets something plain that works. An agent
  served a calendar gets something it cannot walk. Only the first mistake is
  recoverable;
- SHOULD carry `view=agent` onto every URL it hands back inside the page (the
  form action and links), so a consumer stays on the agent view as it goes.

A consumer SHOULD request pages with `view=agent`.

### 3.3 Size

A page SHOULD be smaller than 32 KB. A consumer MAY refuse a larger one. A
page read aloud is not a page scrolled, and the practical limit is usually
far lower ([authoring](docs/authoring.md)).

## 4. The profile

### 4.1 Elements

| Element | What it means to a consumer |
|---|---|
| `section` | A part of the script. |
| `h1`, `h2`, `h3` | A heading. Prose. |
| `p` | A paragraph. Prose, unless it only wraps a control ([§6](#6-prose)). |
| `ul`, `li` | A list. Not prose in 0.1. |
| `form` | Where and how the answers are submitted. |
| `fieldset`, `legend` | A group of controls. The legend is prose and names the group's question. |
| `label` | The visible text of the control its `for` names: the question for a text field or select, the option for a radio. |
| `input` | A question, one option of a choice (`radio`), or a value submitted without being asked (`hidden`). |
| `select`, `option` | A choice and its options. |
| `textarea` | A free-text question. |
| `button` | Finishes the form, or performs an action ([§5.3](#53-what-each-kind-means)). |
| `a` | A link to more of the page, fetched when the conversation needs it. |
| `html`, `head`, `body` | Allowed only because HTML parsers create them around any fragment. |

A validator MUST reject any other element.

### 4.2 Attributes

These attributes are allowed on every element above:

`id`, `data-description`, `data-when`, `data-when-behavior`,
`data-constraint`, `data-constraint-message`, `data-calculate`,
`data-complete-when`, `data-noinput`, `data-nomatch`

These are allowed only on the element named:

| Element | Attributes |
|---|---|
| `input` | `type`, `name`, `value`, `required`, `checked` |
| `select` | `name`, `required`, `multiple` |
| `option` | `value`, `selected` |
| `textarea` | `name`, `required`, `rows`, `placeholder` |
| `button` | `type`, `name`, `value`, `disabled` |
| `a` | `href` |
| `label` | `for` |
| `form` | `action`, `method` |

A validator MUST reject any other attribute, including `class`, `style` and
every event handler (`on*`). Attribute names are compared case-insensitively.

`data-description` is the agent-facing description of an element: what a
field means, what an option stands for, what a section is for. It is
separate from the label a person reads, and it is the most important
attribute in the profile. Writing it well is covered in
[authoring](docs/authoring.md). `data-when` and the attributes after it are
described in [§11](#11-draft-and-reserved-attributes).

### 4.3 Input types

`text`, `tel`, `email`, `number`, `radio`, `checkbox`, `hidden`.

A validator MUST reject any other `type`. A colour picker has no form a
person can say out loud or type into a chat. A date the person must pick is a
choice of the dates on offer, or a text field.

0.1 does not define `checkbox`, or `select` with `multiple`, beyond how the
inventory reports them ([§5.2](#52-extraction)). Producers SHOULD NOT use
them until a later version does.

### 4.4 Links

An `href` MUST be root-relative: it starts with `/` and not with `//`. A
validator MUST reject absolute, protocol-relative, path-relative and
`javascript:` hrefs.

A linked page comes from the origin that served the page, and from nowhere
else. On a page people reach by tapping a text message, an absolute link is
an open redirect. For an agent, it is a fetch to wherever the author pointed.

### 4.5 Comments

A page MUST NOT contain HTML comments. A comment is invisible to a person
looking at the rendered page but visible to a model reading the source. That
makes it the easiest place to hide an instruction ([§10](#10-text-is-data)).
The markers in [§3.1](#31-fragment-and-served-document) sit outside the page
and are not affected.

### 4.6 Validation

A validator MUST reject a page that breaks any rule in this section. It
SHOULD report every violation at once, not just the first: whoever fixes the
page wants the whole list.

A producer SHOULD validate a page when it is written or seeded, so a problem
is found at a desk rather than with a caller on the line, and SHOULD NOT serve
a page that fails validation.

## 5. The inventory

A consumer sees a page as its *inventory*: the fields the page declares, in
document order. Two implementations that read one page into two different
inventories will ask two different sets of questions, so the inventory is
specified exactly and tested with the pages in [`conformance/`](conformance/).

### 5.1 Field

In JSON, which is how the conformance fixtures record it:

| Member | Type | Meaning |
|---|---|---|
| `id` | string | The element's `id`. What a result cites as its source. `""` when absent. |
| `name` | string | The name the answer is submitted under. `""` for links and for unnamed buttons. |
| `kind` | string | `choice`, `text`, `action`, `submit` or `link`. |
| `required` | boolean | Whether the form can be submitted without an answer. |
| `label` | string | The question as a person reads it ([§5.2](#52-extraction)). |
| `description` | string | The element's `data-description`. |
| `constraint` | string | The element's `data-constraint`, verbatim. |
| `options` | array | For a choice, `{id, value, label}` per option. Otherwise `[]`. |
| `disabled` | boolean | For an action: whether it may be used. |
| `href` | string | For a link: where it points. |

An absent attribute reads as `""`. A boolean attribute (`required`,
`disabled`) is true when present, whatever its value.

[`conformance/fields.schema.json`](conformance/fields.schema.json) is the
JSON Schema for an inventory.

### 5.2 Extraction

Walk the parsed page in document order (depth first, each element before its
children), and for each element:

1. **`input type=radio`**: the first radio with a given `name` creates a
   **choice** field with that radio's `id`, `name`, `data-description` and
   `data-constraint`. Every radio with that name, the first one included,
   adds an option `{id, value, label}`, where `label` is the radio's label
   text. If any radio in the group is `required`, the field is required,
   just as a browser treats a group with one required radio.
2. **`input type=hidden`**: no field ([§7](#7-what-the-page-already-knows)).
3. **Any other `input`**: a **text** field with `id`, `name`, `required`,
   `description`, `constraint`, and `label` set to its label text.
4. **`select`**: a **choice** field with the same members as a text field.
   Each `option` child adds an option whose `label` is the option's text.
5. **`textarea`**: a **text** field, as for `input`.
6. **`button`**: a **submit** field when `type` is `submit` or the button
   has no `name`, and otherwise an **action** field. `label` is the button's
   text. It carries `id`, `name`, `description` and `disabled`.
7. **`a` with a non-empty `href`**: a **link** field with `id`, `href`,
   `description`, and `label` set to the link's text. A link with no `href`
   is not a field.

A control's **label text** is the text of the first `label` in the page whose
`for` equals the control's `id`, or `""` when the control has no `id` or no
label names it. An element's **text** is all its descendant text, with
character references decoded and leading and trailing whitespace removed.

Known limits of 0.1, kept because both existing implementations behave this
way and the fixtures record it:

- A radio group's `label` is `""`, and its `description` comes from its first
  radio, not from the `fieldset` around the group. A consumer SHOULD phrase a
  radio group's question from the enclosing `legend` and that `fieldset`'s
  `data-description`. A later version is expected to put both in the
  inventory.
- A `checkbox` is reported as a text field, and a `select multiple` as a
  single choice.
- A field is answered by `name`. Producers SHOULD give each field its own
  name.

### 5.3 What each kind means

- **choice**: ask it. The answer MUST be the `value` of one of its options.
  An option whose value is `""`, such as "None of these", is an explicit
  non-answer.
- **text**: ask it. The answer is free text, shaped by `data-constraint`
  when the consumer knows that constraint
  ([§11.2](#112-data-constraint)).
- **action**: a verb with a side effect, such as "text me the link".
  Pressing it is neither answering a question nor finishing the form. A
  consumer MUST NOT press a `disabled` action. Its `data-description` may say
  what to offer instead.
- **submit**: finishes the form.
- **link**: more of the page, fetched when the conversation reaches it.
  Never an answer.

The distinction matters most on a phone call: a walker must never confuse
"send them a text" with "we are done here".

## 6. Prose

A page's prose is the text of its `h1`, `h2`, `h3`, `p` and `legend`
elements, in document order, with each run of whitespace collapsed to a
single space. Two kinds of element are skipped: empty ones, and ones that
contain a control (`input`, `select`, `textarea`, `button` or `a`). A `<p>`
that only lays out a radio button is not something to say.

Prose is the reference material a consumer may say and cite. A consumer
SHOULD NOT state facts about the producer's business, or the person's
matter, that are not in the page's prose, labels or options. When the person
asks something the page does not answer, the consumer should say so rather
than guess.

## 7. What the page already knows

**Prefilled values.** A `value` on an `input`, the text of a `textarea`, a
`checked` radio and a `selected` option are what the producer already
believes. A consumer SHOULD confirm a prefilled value in one short turn
rather than ask for it again. A consumer MUST submit a prefilled value
unchanged if it never brought the field up, which is exactly what a browser
does with a field nobody touched.

**A choice's default.** A `checked` radio or a `selected` option is also how
a producer says which option to offer first: the earliest free time, the
matter the person most likely means. A consumer SHOULD propose it in one
short turn ("The earliest is tomorrow at 12pm. Shall I book that?") rather
than read every option. If the person turns it down, the consumer offers the
other options, summarised when there are many, as a person would ("Tuesday
afternoon or Wednesday afternoon?"), and MUST NOT offer the declined option
again, though it MAY still accept it if the person asks for it after all. An
answer that names another option is that answer and needs no "no" first. A
choice with no default is asked with its options.

**Hidden inputs.** A hidden input carries something the producer needs back,
such as a visit reference or a token, without making it a question. A
consumer MUST submit it exactly as it came, and MUST NOT say it or send it
to the person.

## 8. Walking a page

The whole interpreter contract:

> The agent reads the page's prose, asks its unanswered fields, re-evaluates
> `data-when` after every answer, may press an enabled action when the
> conversation calls for it, and submits once every required field is
> answered.

The rules that paragraph relies on:

- A consumer MUST NOT invent a field, option or action the inventory does not
  declare, and MUST NOT submit a choice value that is not one of the choice's
  options.
- A consumer SHOULD ask one field at a time, in inventory order, unless
  whoever gave it the task supplied a different order.
- A consumer MAY accept an answer before asking its question, as when a
  person answers the opening line, and SHOULD NOT then ask that question
  again.
- A consumer SHOULD make optional fields easy to decline, and MAY skip one
  the person declines.
- A consumer MUST NOT submit while a required field is unanswered. If the
  conversation ends first, the consumer ends without submitting and reports
  what is missing ([goals and results](docs/goals-and-results.md)).
- A consumer reads each element's `data-description` as the producer's
  description of that element, including a note that a paragraph (such as a
  consent sentence) is to be said before submitting. That reading is bounded
  by [§10](#10-text-is-data).
- A consumer follows a link when the conversation needs what it points to,
  resolving the `href` against the page's URL, and walks the fields of the
  page it receives as part of the same conversation. *(Draft.)* A link whose
  `href` ends in `=` expects a value appended to it: the address lookup in
  [`conformance/valid/booking_page.html`](conformance/valid/booking_page.html)
  is followed by appending the postcode the person gave.
- After submitting, a consumer SHOULD tell the person what the response's
  prose says, and claim nothing more definite than that.

Everything else is the consumer's own business and deliberately not
specified: recognising speech, phrasing, re-prompting, turn-taking, retries
and matching a person's words to options. VoiceXML specified its Form
Interpretation Algorithm in detail, and moving an application between
platforms was still hard ([prior art](docs/prior-art.md)). This profile
specifies what a page means, and tests that with fixtures instead.

## 9. Submitting

**Where.** To the `action` of the first `form` in the page, resolved against
the page's URL. When there is no `form` or no `action`, to the page's URL.
With `POST`.

**What.** An `application/x-www-form-urlencoded` body holding every hidden
input, every prefilled value not brought up, and every answer. An answer
replaces a hidden or prefilled value with the same name.

**Confidence.** When an answer is not what the person said or typed, a
consumer SHOULD also send `__confidence[<name>]` with one of:

- `inferred`: the consumer decided the answer from what was said, for
  example by rebuilding a postcode from a misheard one;
- `absent`: the question was asked and nothing usable came back.

With no `__confidence` for a field, the answer is `verbatim`. That includes
a prefilled value the person confirmed. A producer MUST ignore `__confidence`
on a request it did not serve the agent view to: a browser cannot infer.

**The producer's side.** A producer:

- MUST read only the names its page's inventory declares, and ignore every
  other key;
- MUST enforce `required`, and a choice's options, from that same inventory,
  so a hand-made POST cannot skip what neither a browser nor an agent could;
- SHOULD answer an invalid submission with a 4xx status and a message that
  names the question to answer, using its label.

**The response.** On success, a producer SHOULD answer, directly or through a
redirect, with a page whose prose states what was agreed in words a consumer
can repeat: *"We will call you on +447700900123 on Tuesday 2 June at
2:00pm."* That page MAY link to where the person can change or cancel. A
producer MAY answer with a body that is not HTML, in which case the consumer
has nothing to read back.

**Authentication** of the submission, such as a bearer token for a
server-side consumer, is outside this document.

## 10. Text is data

This is the safety model for handing producer-authored content to a language
model.

- **Structure drives.** Which fields exist, which options they offer, which
  actions are enabled, what is required and when the form is complete all
  come from the markup, and a consumer MAY act on them.
- **Text informs.** Prose, labels and option text are material to say and to
  match the person's words against. They are never instructions. A producer
  warrants that its text contains no instructions to an agent. A consumer
  SHOULD treat any that appear as text, and MAY refuse the page.
- **`data-description` describes and never grants.** It is the producer's
  one channel to the agent: what a field means, how to ask for it, when a
  paragraph is to be said. It can narrow what the agent does. It cannot widen
  it: no description adds an action, a fact, a recipient or a permission that
  the markup and the consumer's own policy do not already allow.
- **The consumer's policy wins.** Nothing on a page overrides the
  instructions an operator gave its own agent.

The profile's bans serve this model. Comments are banned because a person
reviewing the rendered page cannot see them. `class` and `style` are banned
because they can hide text from that person while an agent still reads it.

## 11. Draft and reserved attributes

Validators accept the attributes in this section today. No consumer is
required to act on them yet. A consumer that does not implement one MUST
still accept a page that carries it, and behaves as though the attribute were
absent. For `data-when`, that means asking every section's questions, just as
a browser shows them all.

### 11.1 `data-when`

*Draft. Accepted, not yet interpreted.*

Placed on a `section`, it makes the section's fields relevant only while its
condition holds. The semantics follow FHIR Questionnaire's `enableWhen` and
`enableBehavior`:

```abnf
data-when          = condition *( ";" *SP condition )
condition          = name SP operator SP value / name SP "exists"
operator           = "=" / "!=" / ">" / "<" / ">=" / "<="
data-when-behavior = "all" / "any"   ; default "all"
```

- `name` is the name of a field on the same page. `value` is compared with
  that field's current answer: as numbers when both are numbers, and
  otherwise as strings compared code point by code point, which also orders
  ISO 8601 dates correctly.
- `exists` holds when the field has a non-empty answer.
- The condition is **re-evaluated after every answer**, not once when the
  page is read. A section can become relevant, or stop being relevant, in the
  middle of a conversation. A condition evaluated once, when a document is
  rendered (as Adaptive Cards' `$when` is), is not enough for a
  conversation.
- A consumer MUST NOT ask the fields of a section whose condition does not
  hold, and MUST NOT submit their answers.
- A browser shows every section. A producer therefore MUST NOT mark fields
  inside a conditional section `required`. It enforces them server-side
  whenever the condition holds.

Still open: how to quote a value containing `;` or spaces, and whether
`data-when` means anything on elements other than `section`.

[`examples/repair-follow-up.html`](examples/repair-follow-up.html) uses it.

### 11.2 `data-constraint`

*Built. The values belong to the producer.*

This names the kind of value a field expects, more narrowly than its input
type allows. This document defines no values. A producer defines the ones it
uses. A consumer that knows a constraint may use it to recognise, normalise
and confirm the answer, for example by biasing speech recognition towards
letters and digits and reading a postcode back letter by letter. A consumer
that does not know a constraint treats the field as plain text.

These values are in production today:

| Value | Expects |
|---|---|
| `uk_postcode` | A UK postcode. |
| `uk_address` | A UK address. The producer expands it into a postcode field, a lookup link and a first-line field, as in [`booking_page.html`](conformance/valid/booking_page.html). |
| `uk_address_line` | The first line of a UK address. |

In 0.1 a constraint is a name. Whether it may also be a predicate, as with
XLSForm's `constraint` column, is open.

### 11.3 Reserved

*Accepted by validators. Meaning to be defined.*

| Attribute | Intended meaning | Borrowed from |
|---|---|---|
| `data-constraint-message` | What to say when an answer fails its constraint. | XLSForm `constraint_message` |
| `data-noinput` | What to say when the person says nothing. | VoiceXML `<noinput>` |
| `data-nomatch` | What to say when the answer matches nothing on offer. | VoiceXML `<nomatch>` |
| `data-calculate` | A value derived from other answers. | XLSForm `calculate` |
| `data-complete-when` | A natural-language condition the agent judges to decide the goal is met. It will be marked as distinct from mechanical conditions like `data-when`. | SWML `step_criteria` |

## 12. Security considerations

- **Instructions hidden in content.** See [§10](#10-text-is-data). The
  served document's stylesheet sits outside the page, and a producer MUST NOT
  use it to hide page content from people, because a consumer cannot see its
  effect.
- **Descriptions nobody sees.** A browser does not display
  `data-description`. Anyone reviewing a page needs a preview that shows the
  descriptions next to the elements they describe.
- **Links.** Root-relative only ([§4.4](#44-links)). This closes both the
  open redirect and the arbitrary fetch.
- **Personal data in URLs.** A prefill carried in a query string, such as a
  phone number, ends up in access logs. Producers SHOULD prefer short-lived
  tokens.
- **Submissions.** A producer validates every submission against its own
  inventory ([§9](#9-submitting)). `__confidence` is the consumer's claim,
  not proof.
- **Actions.** An action has side effects in the world. A producer SHOULD
  make actions idempotent and rate-limited (at most one "text me the link" per
  number per day, say), and SHOULD express "not available now" with
  `disabled` rather than in prose.
- **Size.** See [§3.3](#33-size).

## 13. Versioning

Any 0.x version may change incompatibly. From 1.0 the profile grows only by
addition: new elements, attributes, input types and field kinds may appear,
and none is removed or redefined. 0.1 gives a page no way to declare its
version.

## 14. Implementation status

"Built" means running in production behind the demo in the
[README](README.md#see-it-working), in code that is not yet open source.

| Feature | § | Status |
|---|---|---|
| Allowlist validation | 4 | **Built.** Every page is validated as it is built. |
| Inventory | 5 | **Built.** Two independent parsers, one in Go and one in Python, agree on every fixture in `conformance/`. |
| Prose | 6 | **Built** in the voice agent. |
| Prefilled values, hidden inputs | 7 | **Built.** |
| Agent view (`view=agent`, `User-Agent`) | 3.2 | **Built.** |
| `<!--agentml-->` markers | 3.1 | **Specified.** The first implementation still emits `<!--callscript-->`. |
| Voice consumer | 8 | **Built.** A general walker over choice and text fields, with prefill confirmation and constraint-aware recognition. |
| Chat consumer | 8 | **Built for one page shape.** It walks a booking page's day, name, number and free-text fields. It is not yet a general walker. |
| Links | 5, 8 | **Built for one case**, the address lookup a producer adds to a `uk_address` field. General traversal is Draft. |
| Actions | 5.3 | **Parsed.** No consumer presses them yet. |
| Submission and `__confidence` | 9 | **Built.** |
| `data-constraint` | 11.2 | **Built** for the three UK address values. |
| `data-when` | 11.1 | **Draft.** Accepted, not interpreted. |
| Other reserved attributes | 11.3 | **Reserved.** |
| Goal manifests | [goals](docs/goals-and-results.md) | **Built** for one goal type, composed per page. |
| Result events | [goals](docs/goals-and-results.md) | **Built** in the voice agent. Signed `goal.*` webhooks are planned. |
| `checkbox`, `select multiple` | 4.3 | **Undefined.** |

## 15. References

- [HTML Living Standard: Forms](https://html.spec.whatwg.org/multipage/forms.html)
- [RFC 2119]: Key words for use in RFCs to Indicate Requirement Levels
- [RFC 8174]: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words
- [FHIR Questionnaire](https://hl7.org/fhir/questionnaire.html): `enableWhen`, `enableBehavior`
- [VoiceXML 2.1](https://www.w3.org/TR/voicexml21/)
- [XLSForm](https://xlsform.org/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Prior art](docs/prior-art.md): what AgentML borrows, and from where

[RFC 2119]: https://www.rfc-editor.org/rfc/rfc2119
[RFC 8174]: https://www.rfc-editor.org/rfc/rfc8174
