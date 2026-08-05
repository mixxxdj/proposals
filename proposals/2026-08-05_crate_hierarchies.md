# Template

Feel free to copy this file and feel all the details below.

See [README](README.md) for details.

## Crate Hierarchies / Nested Crates / Subcrates

* **Owners:**
  * @cr7pt0gr4ph7

* **Implementation Status:** `Not implemented` (in Mixxx main branch) / `Implemented` (in @cr7pt0gr4ph7's code)

* **Related Issues and PRs:**
  * PR: mixxxdj/mixxx#16835 (utilities for sidebar item drag & drop) &mdash; ca. 2026: Short summary of the discussion from @cr7pt0gr4ph7's point of view, and the reason this proposal document was written.
  * Discussion: mixxxdj/mixxx#5634 (multi level crates) &mdash; ca. 2010 - 2024: Last preceding discussion between @cr7pt0r4ph7, @daschuer and @rons0 about the approach.
  * Discussion: mixxxdj/mixxx#7481 (hierarchical library display)  &mdash; ca. 2010: Another mention of hierarchical crates.
  * PR: mixxxdj/mixxx#10997 by @ywwg (Crate archives) &mdash; ca. 2022: One of the initial implementations of a single archive folder, abandoned by the author @ywwg in 2025 in favor of having user-defined hiearchies.
  * PR: mixxxdj/mixxx#13978 by @Eve00000 (Grouped crates) &mdash; ca. 2024: Alternative implementation based on storing the crate's full path in its name. Stalled since the start of 2025.
  * PR: mixxxdj/mixxx#4182 by @Eve00000 (SearchCrates) &mdash; ca. 2025: Tangentially related, orthogonal proposal of adding dynamic crates whose content is determine by a search query.
  * PR: mixxxdj/mixxx#14352 by @Eve00000 (Grouped playlists) &mdash; ca. 2024: The same as "Grouped crates", but for playlists. As the proposal you're reading is focused on crates, we'll ignore playlists for now.
  

* **Other docs or links:**
  * `<Links…>`

> We need a way to organize crates into a hierarchy
> idea of proposer's view on proposed changes.
>
> *For example: This design doc is proposing a consistent design template for “example.com” organization.*

## Why

Put here a motivation behind the change proposed by this design document, give context.

*For example: It’s important to clearly explain the reasons behind certain design decisions in order to have a consensus
between team members, as well as external stakeholders. Such a design document can also be used as a reference and
knowledge-sharing purposes. That’s why we are proposing a consistent style of the design document that will be used for
future designs.*

### Pitfalls of the current solution

What specific problems are we hitting with the current solution? Why it’s not enough?

*For example, We were missing a consistent design doc template, so each team/person was creating their own. Because of
inconsistencies, those documents were harder to understand, and it was easy to miss important sections. This was
causing certain engineering time to be wasted.*

## Goals

Goals and use cases for the solution as proposed in [How](#how):

* Allow easy collaboration and decision making on design ideas.
* Have a consistent design style that is readable and understandable.
* Have a design style that is concise and covers all the essential information.

### Audience

If not clear, the target audience that this change relates to.

## Non-Goals

* Move old designs to the new format.
* Not doing X,Y,Z.

## How

Explain the full overview of the proposed solution. Some guidelines:

* Make it concise and **simple**; put diagrams; be concrete, avoid using “really”, “amazing” and “great” (:
* How you will test and verify?
* How you will migrate users, without downtime. How we solve incompatibilities?
* What open questions are left? (“Known unknowns”)

## Alternatives

The section stating potential alternatives. Highlight the objections reader should have towards your proposal as they
read it. Tell them why you still think you should take this path.

1. This is why not solution Z...

## Action Plan

The tasks to do in order to migrate to the new idea.

* [ ] Task one <GH issue>
* [ ] Task two <GH issue> ...
