# Proposals

This repository holds all the past and current proposals for the Mixxx Software. It’s the single place for
reviewing, discovering, and working on the design documents. It is also a record of past decisions and approvals.

## Current Proposals

* The
  [open PRs with the `proposal` label](https://github.com/acolombier/mixxx-proposals/pulls?q=is%3Aopen+is%3Apr+label%3Aproposal)
  show all the pending proposals.
* The [proposals directory](./proposals) shows all the accepted proposals. See the “Implementation Status” for details
  on the implementation.
* The
  [PRs with the `proposal` label that are closed without merging](https://github.com/acolombier/mixxx-proposals/pulls?q=is%3Apr+label%3Aproposal+is%3Aclosed+is%3Aunmerged)
  show all the rejected proposals.

## What’s a Design Document?

It’s essential to clearly explain the reasons behind certain design decisions to have a community consensus. This is
especially important in Mixxx, where every decision might have a significant impact given the high adoption and
stability of the software and standards we work on.

In our world, no decision is perfect, so having a design document explaining our trade-offs is essential.
Such a document can also be used later as a reference and for knowledge-sharing purposes.

Design documents do not always reflect what has been (or will be) implemented. Implementation details
might have changed since a feature was merged. Design docs are not considered documentation and can not define a
standard.
Instead, it should explain the motivation, scope, decisions, and alternatives considered.

A design document shall describe the use-cases the proposal will implement and also the use-cases that are explicitly
not included.

It may also contain requirements and acceptance criteria of a derived Pull Request. This avoid feature creep and adjust
the expectations during a review.

## Proposal Process

Don’t get scared to propose ideas! It’s amazing to innovate in the open and get feedback on ideas.

The process of proposing a change via a design document is the following:

1. Fork `github.com/acolombier/mixxx-proposals`.
2. Create a GitHub Pull Request with a design document in markdown format to the [proposals directory](./proposals).
   Make sure to use the [template](YYYY-MM-DD_template.md) as the guide for what sections should be present in the
   document. Put the creation date (the day you started preparing this design document) as the prefix and some unique
   name as the suffix in the file name. Once the PR is proposed, a maintainer will assign a `proposal` label.
3. An automatic formatter is enabled in the repository. Use `pre-commit` locally to trigger the formatting of all
   markdown documents (requires a working `pre-commit` installation).
4. After a sufficient amount of discussion, the Mixxx team will try to reach a consensus of accepting or rejecting the
   proposal. In the former case, the PR gets merged. In the latter case, the PR gets closed with meaningful reasons why
   the proposal was rejected.
   1. To merge the PR, we need approval (consensus) from the maintainers of the related component(s).
   2. Optionally: Find a sponsor among the Mixxx maintainers to get momentum on a change. You may use
      [Zulip](https://mixxx.zulipchat.com/) to do that.

Once the PR gets merged, the design document can change, but it requires (less strict, but still) a PR with review and
merge by a maintainer.

## Credits

This proposal process was largely inspired by the [Prometheus Proposal Process](https://github.com/prometheus/proposals)
