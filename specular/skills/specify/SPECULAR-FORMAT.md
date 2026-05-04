# SPECULAR.md format

`SPECULAR.md` is a per-folder glossary of semantic definitions: domain terms, the concepts they name, and how they relate. It is committed to the repo and shared by the team.

`SPECULAR.md` is hierarchical, like `CLAUDE.md`:

- Any folder can contain a `SPECULAR.md`.
- A `SPECULAR.md` in a child folder takes precedence over a `SPECULAR.md` in any parent folder when terms collide.
- A repo without any `SPECULAR.md` is fine - it just means no canonical vocabulary has been recorded yet.

When a skill needs vocabulary for a folder, it walks from that folder up to the repo root, collecting every `SPECULAR.md` it finds. Closer files win on conflict.

## Structure

```md
# {Glossary name - usually the folder or bounded context}

{One or two sentence description of what this glossary covers and why it exists.}

## Language

**Order**:
{A concise description of the term}
_Avoid_: Purchase, transaction

**Invoice**:
A request for payment sent to a customer after delivery.
_Avoid_: Bill, payment request

**Customer**:
A person or organization that places orders.
_Avoid_: Client, buyer, account

## Relationships

- An **Order** produces one or more **Invoices**
- An **Invoice** belongs to exactly one **Customer**

## Example dialogue

> **Dev:** "When a **Customer** places an **Order**, do we create the **Invoice** immediately?"
> **Domain expert:** "No - an **Invoice** is only generated once a **Fulfillment** is confirmed."

## Flagged ambiguities

- "account" was used to mean both **Customer** and **User** - resolved: these are distinct concepts.
```

## Rules

- **Be opinionated.** When multiple words exist for the same concept, pick the best one and list the others as aliases to avoid.
- **Flag conflicts explicitly.** If a term is used ambiguously, call it out in "Flagged ambiguities" with a clear resolution.
- **Keep definitions tight.** One sentence max. Define what it IS, not what it does.
- **Show relationships.** Use bold term names and express cardinality where obvious.
- **Only include terms specific to this context.** General programming concepts (timeouts, error types, utility patterns) don't belong even if the project uses them extensively. Before adding a term, ask: is this a concept unique to this context, or a general programming concept? Only the former belongs.
- **Group terms under subheadings** when natural clusters emerge. If all terms belong to a single cohesive area, a flat list is fine.
- **Write an example dialogue** when it helps clarify boundaries between related concepts. Skip it for thin glossaries.

## Where to put SPECULAR.md

- **Single-context repo:** one `SPECULAR.md` at the repo root.
- **Multi-context repo:** put a `SPECULAR.md` in each bounded context (e.g. `src/ordering/SPECULAR.md`, `src/billing/SPECULAR.md`). Optionally keep a thin root `SPECULAR.md` for terms that span all contexts.
- **Subfolder overrides:** if a deeper folder uses a term differently from its parent, define the term in that folder's `SPECULAR.md`. The deeper file wins.

Skills that need vocabulary for a file at `path/to/file.ts` walk upward: `path/to/SPECULAR.md`, `path/SPECULAR.md`, `SPECULAR.md`. Closest match wins on conflict; non-conflicting terms accumulate.
