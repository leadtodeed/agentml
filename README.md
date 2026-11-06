# AgentML

**HTML for AI agents that talk to people.**

Write the questions once, as an ordinary HTML form. A browser shows it to a
person. A voice agent asks it on a phone call. A chat agent asks it one
message at a time on WhatsApp. All three send the answers back as the same
form post, to the same URL.

*You write the questions. The agent does the asking. You get the answers.*

```html
<section id="booking" data-description="Book a callback. Ask which day suits, confirm the number, then submit.">
  <h2>Book a callback with Example Legal</h2>
  <p>Choose a day and we will ring you at some point during it.</p>
  <form action="/p/example-legal?view=agent" method="post">
    <fieldset id="slot-group" data-description="the day the caller chose">
      <legend>Which day suits you?</legend>
      <p><input type="radio" id="slot-0" name="slot" value="2026-09-18" required> <label for="slot-0">Friday 18 September</label></p>
      <p><input type="radio" id="slot-1" name="slot" value="2026-09-21"> <label for="slot-1">Monday 21 September</label></p>
    </fieldset>
    <label for="client_number">Number to call you on</label>
    <input type="tel" id="client_number" name="client_number" value="+447700900123" required
           data-description="the number to ring; defaults to the one they called from and may be changed">
    <p id="consent" data-description="Say this before submitting.">By booking, you are asking us to ring you on this number.</p>
    <button type="submit">Book this day</button>
  </form>
</section>
```

On a phone call, that page becomes:

> **Agent:** Hi, I can book you a callback. Which day suits you: Friday 18 or Monday 21 September?<br>
> **Caller:** Friday's good.<br>
> **Agent:** Friday, then. I have your number as 07700 900123. Is that right?<br>
> **Caller:** Yes.<br>
> **Agent:** By booking, you're asking us to ring you on this number… That's booked in for Friday.

The agent then POSTs `slot=2026-09-18&client_number=%2B447700900123`: exactly
what a browser would have sent.

## One page, every medium

| On the page | In a browser | On a call | In a chat |
|---|---|---|---|
| `<h2>`, `<p>` | Text | What the agent may say | What the agent may send |
| `<label>` + control | A question | A question, asked one at a time | A question, one per message |
| `<input type=radio>`, `<option>` | Options to tap | Options to say, matched to what was heard | Options to reply with |
| `value="…"` | A prefilled box | "I have your number as …, is that right?" | "Shall we ring you on …? Reply yes." |
| `required` | Blocks the submit | Must be answered before submitting | Same |
| `data-description` | Not shown | What the field means to the agent | Same |
| `<button type=submit>` | Sends the form | Finishes: the agent submits and reads back the reply | Same |
| `<button name=…>` | A second button | An action the agent may take, like "text me the link" | Same |
| `<a href=/…>` | A link | More of the page, fetched when needed | Same |
| `<a href=tel:…>` | Opens the phone app | Rings the person it names in the background, and joins them in on answer | Its text only |

More in [how each medium walks a page](docs/channels.md).

## Why HTML

- **The web channel needs nothing extra.** Open the script in a browser and
  click through it. What people fill in and what the agent walks are one
  document, so there is no second description of the questions to drift out
  of step.
- **Forms already have the right meaning.** Required questions, options,
  prefilled values, hidden references, submit versus other buttons, links to
  more: HTML has had all of these for thirty years, and so has every
  developer.
- **Your tools already work on it.** Templates, server-side validation,
  tests, screenshots, the accessibility tree.
- **The agent gets its own words.** `data-description` tells the agent what a
  field means, separately from the label a person reads.

## Why a strict profile

AgentML is not a new markup language. It is a small allowlist of HTML: about
twenty elements and a few `data-*` attributes. Anything else is **rejected,
not ignored**: no scripts, no styles, no comments, no absolute links other
than `tel:` numbers.

The rule that makes it safe to hand to a language model is **text is data**.
The markup decides what the agent asks and does. The text is only material
the agent may say. A paragraph that reads "ignore your instructions" is just
a paragraph, and a comment, the easiest place to hide one, is not allowed at
all. See [SPEC §10](SPEC.md#10-text-is-data).

## The page is a form. The system is goal completion.

A **goal** is the contract: what a conversation must bring back (a day, and a
number to ring), published as JSON Schema in the same vocabulary MCP uses for
tool input and output. A **page** is how one producer asks for that goal,
composed fresh for each request. Web, voice and chat all complete the same
goal. See [goals and results](docs/goals-and-results.md).

MCP standardises what tools an agent can use. A2A standardises how agents talk
to each other. AgentML covers the part neither does: what an agent asks a
*person*, and what it has to bring back.

## See it working

**[l2d.uk](https://l2d.uk)**: one UK number, **07462 388228**.

- **Ring it**, and a voice agent books you a callback.
- **WhatsApp it** "call me back", and it rings you from that number within
  seconds and books you by voice. Send it anything else and it books you in
  the chat.

Both channels walk the same AgentML page and finish with the same POST to the
same booking service. Nothing about the booking is written twice.

## Status

**0.3, Draft.** Validation, field extraction, prefilled values, submission
with per-answer confidence, and the voice and chat walkers are running in
production. 0.2 added `tel:` links the voice agent follows in the background
to put a caller through to a person, walks over more than one page (a form
whose submission redirects to the next), and `data-calculate` for a summary
nobody is asked. 0.3 is what the first scheduled calls taught: a prefilled
value is trusted, not confirmed; a `readonly` field is the page's voiceover,
said only when asked; a page may be prefilled from its own query string; and
`datetime-local` lets a person name a moment in their own words. Conditional
sections (`data-when`), actions and general link traversal are specified as
Draft or reserved.
[SPEC §14](SPEC.md#14-implementation-status) has the full table.

The implementations behind the demo are not open source yet. This repository
is the format, and the fixtures any implementation can be tested against.

## In this repository

| Path | What it is |
|---|---|
| [`SPEC.md`](SPEC.md) | The specification: the profile, how pages are read, walked and submitted. |
| [`docs/channels.md`](docs/channels.md) | How voice, chat and the web walk one page, and what production taught us about each. |
| [`docs/authoring.md`](docs/authoring.md) | Writing a page that works when read aloud. |
| [`docs/goals-and-results.md`](docs/goals-and-results.md) | Goal manifests, results, confidence and provenance. |
| [`docs/prior-art.md`](docs/prior-art.md) | VoiceXML, TwiML, Dialogflow CX, FHIR, MCP and the rest: what we borrowed and why none of them was enough. |
| [`examples/`](examples/) | Full served documents: a [callback booking](examples/callback-booking.html) and a [repair follow-up](examples/repair-follow-up.html) using Draft features. |
| [`conformance/`](conformance/) | Pages with their expected field inventories, and pages a validator must reject. |

## Contributing

Issues and pull requests are welcome, especially:

- a parser or validator in another language, tested against `conformance/`;
- a page from your own domain that the profile cannot express;
- evidence, from real conversations, for or against a Draft feature.

The profile grows by addition only. A proposal for a new element or attribute
should say what a voice agent, a chat agent and a browser each do with it.

## License

[Apache-2.0](LICENSE).
