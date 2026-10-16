# One page, every medium

The page owns **what** is asked. The medium owns **how** it is asked. A voice
agent and a chat agent phrase the same field differently, match answers
differently and recover from mistakes differently. They still agree on the
inventory, never invent an option, and send the same POST at the end.

What every consumer does, whatever the medium:

1. Fetch the page with `view=agent` and read its inventory and prose
   ([SPEC §5–6](../SPEC.md#5-the-inventory)).
2. Ask the unanswered fields, one at a time. Confirm prefilled values rather
   than ask for them.
3. Turn each answer into a value the page offers: an option's `value`, or
   text shaped by the field's constraint.
4. Submit on reaching the submit button with every required field answered,
   then tell the person what the response page says, or walk it if it asks
   for more ([SPEC §8.1](../SPEC.md#81-more-than-one-page)).

The rest of this document covers what each medium adds, and what running them
in production taught us.

---

## Web

A browser needs nothing: the page is HTML. With no JavaScript the whole flow
is form POSTs, so it works on any handset, in any browser, on a bad train
connection.

A producer may serve people a richer document at the same URL, such as a
month calendar instead of a list of fourteen days, and give agents the page
behind `view=agent` ([SPEC §3.2](../SPEC.md#32-the-agent-view)). The two are
genuinely different documents: 24 radio buttons are a long scroll on a
phone, and a calendar is unreadable out loud. Compute what is on offer once
and render it twice, with both documents posting the same names.

Styling lives in the served document around the page, never inside it.

## Voice

The structure that has worked for us: **code walks the page and the model
talks.** A small controller decides which field comes next, whether an answer
matches, when to read back and when to submit. The language model only
phrases each turn from a short note the controller hands it ("Say: Friday,
then. And your name?"). A model left to walk a form on its own tends to skip
questions, ask answered ones again, or confirm things nobody said.

What real calls taught us:

- **The opening can ask the first question.** If the greeting ends "which day
  suits you?", the caller's first words are the answer. Take them rather than
  asking again.
- **Don't read lists.** Offer three options and a range ("Thursday, Friday or
  Monday, or any day up to the 1st of October"). Callers talk over a list, and
  whatever they say while the agent is still speaking is easily lost.
- **Keep turns to a sentence or two.** Short questions do more for turn-taking
  than any matcher.
- **Tell the model every option, even when it should say only some.** What
  is spoken can be short; what the model is told must be whole. A note that
  listed six of seven options and "… (1 more)" left the model to fill the
  gap, and it offered "call marks", a word the caller had just been misheard
  saying, as an option the page does not have. Shortening is the
  controller's decision or the model's phrasing, never a hole in the note.
- **Match words to option labels, not to values.** Weekday names, "next
  Monday", "tomorrow", "the 21st", "the second one". When rules find nothing,
  a model may choose among the labels. The answer is still one of the page's
  values, never a date the model made up.
- **Expect misrecognition of short answers.** Speech recognition turned
  "Friday" into "Finding" and "Heidi". Ignore a single short non-match (often
  a fragment of speech), and on the second, ask again with the options named.
- **Bias recognition from the field.** A day question primes weekday names.
  A `uk_postcode` field primes letters, digits and the phonetic alphabet. Set
  the bias before the caller speaks, not partway through the answer.
- **Keep every reading.** When a hard answer is asked twice, merge all
  readings of both attempts. For spoken UK postcodes we measured 43.6%
  correct on the first attempt's readings and 63.2% with every reading of
  every attempt merged (117 real repeats). Asking again is worth more than
  any decoder tweak.
- **Confirm prefilled values in one breath.** "I have your number as 07700
  900123, is that right?" Don't read numbers back digit by digit unless the
  caller asks.
- **A required field the agent can't fill means no result.** A page once
  required "What is the enquiry about?" with four options callers never quite
  said, so the agent never submitted and nothing was booked. Keep `required`
  to what the goal truly needs ([authoring](authoring.md)).
- **Read the confirmation back as written.** The response page's prose is a
  promise the producer made, so don't paraphrase it into a stronger one.
- **Latency is silence.** A model that takes fifteen seconds to reply leaves a
  dead line. Stream the first sentence, and keep the model's prompt small:
  the page is the script, so the prompt only needs to set the manner.

### Context the consumer derives from an answer

A page asks for what the person can tell it; it does not have to ask for what
the consumer can look up. The support-line agent (meowgent, 2026-09-27) asks
`property_address` (`data-constraint="uk_address"`) and nothing about the case:
from the address the consumer derives, out of its own data, which matter it
is, which team handles it and who leads it, and writes that into the page's
free-text `message` before submitting. The page stays a plain form a browser
can render, the producer never learns a case id from the caller, and the
lookup can move (from a local table to the tenant's API) without the page
changing. Two rules kept it honest:

- **The derived fact is never spoken.** Who handles a matter is written into
  the submission for the team, not read to whoever is on the line — the
  consumer cannot know they are the client.
- **Ambiguity becomes one more question, phrased from the answer.** Two
  matters at one address: "I have two matters at that address — a sale and a
  purchase. Which is it about?" — a choice whose options are single words.

Lessons from the humans' calls that shaped the voice walk (1,333 support
calls read): open with the first question, not "how can I help" (the postcode
was asked on 51% of calls and spelled in NATO on 41%); read the first line of
the address back rather than the postcode a second time; confirm the number
the person is calling from in one breath; the promise after submitting is the
page's own confirmation and nothing more definite.

### Talking to a person: live connect

*Being built (meowgent, October 2026).* The support line above books a call
back. Live connect puts the caller through to the team instead, when someone
is free, and keeps the booking for when nobody answers. It is two plain
pages and a `tel:` link:

1. **The entry form.** The call starts on the tenant's incoming-call form.
   Its first field is the property's address (`uk_address`), and its submit
   button ("Find my team") comes straight after it, the last required field.
   The walk submits as soon as it has the address
   ([SPEC §8.1](../SPEC.md#81-more-than-one-page)).
2. **The producer dispatches.** The form's handler finds who handles that
   address and answers `303` to that team's page, with the address carried
   as prefilled values. The consumer no longer derives the team itself
   (above): it walks the page it is sent. The entry form stays open, like a
   second tab.
3. **The team page offers a person.** When someone on the team is free, the
   page starts with a sentence and a link:

   ```html
   <p>Someone from the Example team is available now.</p>
   <a href="tel:+442046206677;to=01a14495-5600-7f00-bf02-112233445566">Talk to them now</a>
   ```

   The walk reaches the link first and follows it in the background: the
   phone system rings the team while the agent keeps talking ("Someone from
   the Example team is free, I'm trying them now. While it rings, who am I
   speaking to?"). When nobody is free, the page has no link, and the walk is
   the booking flow above.
4. **Details while it rings.** The agent asks the entry form's remaining
   questions (who is calling, their part in the matter), then the team
   page's. The entry form's come after its submit button, so they are never
   posted. They are the call's notes instead, with a
   `data-calculate="conversation"` summary rewritten after each answer, and
   the person being rung reads them on the ringing card before picking up.
5. **On answer, the agent goes quiet.** The team member is joined to the
   call, the agent says one line ("I've got Sam from the team for you now")
   and stays silent until hangup. It stops writing the notes, which are the
   team member's from then on.
6. **No answer.** If nobody has picked up when the walk reaches the time
   question, the agent stops the ring, says they couldn't pick up, and books
   a call back on the same page.

Three rules keep it safe and honest:

- **Only the line's own numbers.** The agent dials only numbers its
  operator configured for the tenant, and resolves `;to=` (here a team) only
  within that tenant ([SPEC §12](../SPEC.md#12-security-considerations)). A
  page cannot make it ring anyone else.
- **One attempt per call.** The link is followed once. A caller who asks for
  a person before the team is known is told the agent will find their case
  first; once a page offers the link, it is followed straight away.
- **The link means someone is free now.** The producer renders it only while
  a team member is online and not on a call, so the agent never promises a
  person the page did not offer.

## Chat

Chat is the same walk, one message at a time.

- **One question per message.** List the options in the message ("Friday 18,
  Monday 21 or Tuesday 22 September?") and accept a reply in words.
- **Make optional questions easy to skip:** "Anything we should know first?
  Reply no if not."
- **The chat identity is a prefill.** When the platform reveals the sender's
  number, confirm it: "Shall we ring you on +44 7700 900123? Reply yes, or
  send the number to use." Some chat identities carry no number, so ask.
- **Say the producer's error back.** If the POST is refused, its 4xx message
  names the question that needs answering.
- **A short typing delay** before a reply reads as a person. Skip it for
  messages that must arrive first ("Ringing you now").

## Switching medium partway through

Because the page is the script, a conversation can change medium without
restating anything. In the [demo](../README.md#see-it-working), a WhatsApp
message asking for a call starts a voice agent that rings back from the same
number. It walks the same page, with the number the person messaged from
already filled in. The booking that results is the same POST the chat would
have sent.

## Media not built yet

These follow from the model but have not been built. They are here to show
the shape, not as a promise.

- **SMS**: like chat, with a hard length limit. Link to the web view for
  anything long.
- **Phone keypad**: options numbered in page order ("press 1 for Friday").
  Text fields fall back to speech.
- **Email**: the questions in one message, or a link to the web view.
- **Other agents and MCP**: a page's inventory maps onto a tool's input
  schema, and its named buttons onto tools. An agent acting for the person can
  fill the form directly. See [goals and results](goals-and-results.md#alignment).
- **Screen readers**: nothing to build. The page is made of real controls with
  real labels.
