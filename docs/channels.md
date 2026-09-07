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
4. Submit once every required field is answered, then tell the person what
   the response page says.

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
