## QML mapping requirements

* **Owners:**
  * Daniel Schürmann

* **Implementation Status:** `Not implemented`

* **Related Issues and PRs:**
  * Create a reactive programming API for controllers [#13440](https://github.com/mixxxdj/mixxx/issues/13440) 
  * QML-based components API for controllers [#13459](https://github.com/mixxxdj/mixxx/pull/13459)
  * Respect the Midi timestamp when scratching [#6951](https://github.com/mixxxdj/mixxx/issues/6951)
  * make brake, soft start, and spinback part of the effects system [#8867](https://github.com/mixxxdj/mixxx/issues/8867)

* **Other docs or links:**
  * `<Links…>`

* ** This document shall collect requirements and expectation a new mapping system based on QMl shall fullfill and shall not fulfill. 

## Why

It’s important to clearly explain the reasons behind certain design decisions in order to have a consensus
between team members, as well as external stakeholders. 

It turned out that we have conflicting or unclear requirements. We need to decide the way forward to not end up with a blocked PR or double work.
We need to decide which of them are mandatory optional.

### Pitfalls of the current solution

TODO

## Goals

Here is a fist draft of requirements

1. User Requirements

1.1 Good performance, responsive scratching (Real-Time)
1.2 Easy reassign buttons and sliders
1.3 Easy map a new controller from the scratch

2. Mapping Developer

2.1 Everything at one place, one file, folder, pack
2.2     Easy to share
2.3 Reuse common programming skills
2.4 High abstraction level
2.5 Complete API documentation
2.6 Good IDE support .. code completion.
2.7 On the fly edits, no Mixxx restart.
2.8 Stable: Compatible to future Mixxx versions

## Non-Goals

TODO

## How

### Ideas for discussion 

> we can't make the metadata parsing independent from a QMLEngine

That is correct if we use only one parser. I can image to workaround that by a stripped Mixxx QML parser. reading a file hash, is in this context the minimum version of such parser, but we can read also other info.

> We would need to serialize it into a separate file in a format that we can actually manipulate.

I can imagine QT does exactly that. Maybe we can copy from them and use it for our second parseing.

The other topic is how the data stream is handled. I like to keep the connection factory and the dispatcher in the C++ domain, controlled by QML XML or JS to share concepts between the mapping types. I can also imagine to read the mapping by XML and write them as QML or such for an easy transition.

Another point that bugs me is the weak scratching stability and the programming skills required to map the jogg wheel. Both can be targeted by handling the jog in wheel in the C++ domain and configure it from QML XML or JS.

## Alternatives

The section stating potential alternatives. Highlight the objections reader should have towards your proposal as they
read it. Tell them why you still think you should take this path.

1. This is why not solution Z...

## Action Plan

The tasks to do in order to migrate to the new idea.

* [ ] Task one <GH issue>
* [ ] Task two <GH issue> ...
