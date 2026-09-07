# Conformance fixtures

Pages, together with what an implementation must make of them. If you write
an AgentML parser or validator, test it against these files. They are the
same fixtures the two existing implementations (one in Go, one in Python)
are tested against, with business names and addresses swapped for examples.

## `valid/`

Each `<name>.html` is a page that a validator MUST accept. Next to it,
`<name>.fields.json` is the inventory a parser MUST produce from it
([SPEC §5](../SPEC.md#5-the-inventory)). Compare parsed JSON rather than
bytes. [`fields.schema.json`](fields.schema.json) is the schema for an
inventory.

| Fixture | What it covers |
|---|---|
| `booking_page` | A complete page, as a producer serves it: radios collapsing into one choice, prefilled values, a select, a hidden input (absent from the inventory), a `uk_address` field expanded into a postcode, a lookup link and a first line, and a submit button. |
| `booked` | The confirmation returned after a successful submission: prose to read back, and a link. |
| `address_options` | The page a lookup link returns: a select whose first option has an empty value ("None of these"). |
| `radios` | A radio group, a disabled named button (an **action**), and a submit button. |
| `select` | A select whose options carry their own ids, a textarea, and a hidden input. |
| `link` | A link with an `id`, `href` and description, next to a constrained text field. |
| `reserved_attributes` | Every reserved attribute ([SPEC §11](../SPEC.md#11-draft-and-reserved-attributes)): accepted, and passed through as `constraint` where the inventory carries it. |

Some things these fixtures pin down are easy to get wrong:

- a radio group's `label` is `""`, and its `id` is its first radio's `id`;
- `required` on any one radio makes the whole group required;
- a `<button>` with no `name` is a `submit`, even without `type="submit"`;
- hidden inputs never appear in the inventory;
- character references are decoded (`&amp;` in an `href` becomes `&`).

## `invalid/`

Pages a validator MUST reject. [`cases.json`](invalid/cases.json) lists each
file with the reason:

```json
{"file": "comment.html", "reason": "HTML comment inside the fragment", "mentions": "comment"}
```

`mentions` is a string the first implementation's error message contains.
It helps you check that a page was rejected for the right reason, but your
messages don't have to match it. `many_violations.html` breaks three rules
at once, and a validator SHOULD report all three
([SPEC §4.6](../SPEC.md#46-validation)).

## Adding a fixture

A new fixture needs an implementation that produces its inventory, and a
second implementation that agrees with it. Open a pull request with the page,
its `.fields.json`, and a line in the table above saying what it covers.
