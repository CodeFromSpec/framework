# Future Work

The theory behind this methodology
([codefromspec.com/theory](https://codefromspec.com/theory))
creates obligations the framework does not yet meet. This is
the public record that we know what we owe, ordered by how
much it hurts today.

- **Source-level pieces.** Some pieces belong at the source
  level — written and kept as source code, governed by the
  spec tree but not generated. The framework has no way to
  say "this part is decided literally."

- **Generator versioning.** A spec is a delta against the
  prior of the generator that reads it, so the generator's
  identity must be declared in versioned configuration and a
  swap reviewed as a real event. Today the model identifier
  is buried in the subagent definition.

- **The oracle side.** The framework covers aiming in detail
  and confirming barely at all: nothing prescribes how a
  project organizes its verdicts — which are trusted enough
  to feed back, how tests stay independent of the code they
  check. [TESTING.md](TESTING.md) is the seed.

- **Ratification into both destinations.** A durable fix
  enters both the description and the oracle. The framework
  prescribes "the fix goes in the spec"; the test half of
  the pair has no prescribed discipline.

- **The presentation problem.** The spec chain —
  dispositions, previous content — is our current answer to
  presenting spec and artifact together without the artifact
  capturing the draw. It is a bet, not a validated result.

- **Tree-aware chain reading.** Spec authors naturally write
  from their position in the tree — "the parent node", "this
  node" — and chain entry names already encode the hierarchy
  as paths. The generation subagent should be taught to
  resolve such references from the entry names and their
  order, rather than policing the authored form to avoid
  them.

- **Spec-toward-prompt drift.** The generation loop polices
  only what changes the generated output, so specs drift
  toward excellent prompts and away from a legible theory of
  the system. No check in the workflow answers to human
  legibility.

- **Boundary placement.** The spec tree can express any
  decomposition, but the framework offers no guidance for
  placing node boundaries along the theory's axes — settled
  or moving, conventional or unconventional, cheap or
  expensive to verify.
