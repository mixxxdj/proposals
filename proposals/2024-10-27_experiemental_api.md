# Experimental CO API Prefixes

* **Owners:**
  * `@acolombier`

* **Implementation Status:** `Implemented`

* **Related Issues and PRs:**
  * #13268
  * #13300

* **Other docs or links:**
  * `<Links…>`

> TL;DR: Introduce a new standard prefix for CO that raises concerns when submitted.

## Why

As we've recently seen in a few PRs, it happens that suggested CO values or names raise concerns about whether they are
perfectly fit for purpose. That is, whether their name is 100% clear to our users and whether the value representation
is usable.

### Pitfalls of the current solution

Currently, our way to address this issue is to get all reviewers involved to agree on what's proposed in a PR. This
often leads to a "too many cooks" problem and an oracle game, where everyone tries to predict the future evolution of
 Mixxx. As we all know, the future rarely
happens as expected, and limitations or design decisions turn out to be outdated after a certain time.

## Goals

The goal of this proposal is to define a process that allows quick iteration on API changes while ensuring the
stability that the Mixxx Development team has fought so hard to keep intact. A requirement for the outcome of this
proposal is to require a minimum of code changes,
ideally none, and focus on establishing a process that contributors can easily align with.

### Audience

The audience of this proposal is strictly targeted at Mixxx core contributors, who may be writing C++ code to the
Mixxx codebase.

## Non-Goals

* Introduce core API code changes
* Allow any CO proposition to be accepted. If there's clear consensus against a CO proposition, it won't be
  eligible for the experimental process

## How

To allow quick changes, COs that don't get consensus at submission can be submitted as an experimental CO. An
experimental CO will always be prefixed with the [experimental marker](#experimental-marker). Typically,
experimentation will never last more than a version (~ 6 months), but the Mixxx Core Team may decide to extend the
experimentation by a second version cycle if there's not enough feedback during the first cycle or if there remain
risks for negative user feedback or lacking use case coverage to confirm that the CO is mature enough to exit the
experimentation sandbox. During the version following the CO stable promotion, an alias should be kept to its former
key to prevent breaking changes. The Mixxx Core Team may decide to extend the alias period by a second version cycle
but should refrain from doing it more than once in order to reduce the amount of COs that have an impact on
performance.

### Experimental marker

The suggested experimental marker is the prefix `x_`, e.g., `x_beats_set_change_marker`.

### Process

The suggested process to ensure proper follow-up of an experimental CO could be the following:

* An issue should be created using a standard template. This issue will remain open during the experimentation period
  and target the milestone `<version>+1`.
* Documentation of the CO should include a note about the experimental aspect of that CO.
* At `<version>+1`, the CO is trimmed from its experimental prefix, and the documentation is updated.
* At `<version>+2`, the CO experimental alias is dropped.

### Standard template

```plaintext
Subject: Experimental CO: <StableCOName>

Body:

Introduced by: #<PRNumber>
Documented by: #<ManualPRNumber>
Usecase:
<Short summary of the scope>
```

Here's an example of issue:

```plaintext
Subject: Experimental CO: beats_set_change_marker

Body:

Introduced by: #13300
Documented by: #702
Usecase:
Sets a new independent marker grid at the current play position. The new marker grid will inherit the BPM from its
predecessor but will be fully autonomous
```

*Note that `Usecase` will largely be inspired of the manual, but may also capture technical implementation details that
may help assessing the stability of the CO during its probation*

### Completion

During the experimentation period, one may suggest dropping a CO for the following reason:

* Bug report due to API complexity or lack of use case support
* Superseding of behavior by newly introduced CO

Once the probation period terminates, the CO should be considered stable, except if an extension is asked by any
member of the core team.

## Alternatives

No alternatives is currently being considered.

## Action Plan

The tasks to do in order to migrate to the new idea.

* Standard template to be captured in the Wiki or Issue template
