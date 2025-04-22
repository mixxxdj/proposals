# Track Similarity Column

<!-- markdown-toc -->

* **Owners:**
  * Daniel Fernandes

* **Implementation Status:** Not implemented

* **Related Issues and PRs:**

  * [Issue 5655](https://github.com/mixxxdj/mixxx/issues/5655)
  * [Issue 7810](https://github.com/mixxxdj/mixxx/issues/7810)
  * [Issue 7599](https://github.com/mixxxdj/mixxx/issues/7599)
  * [Issue 9896](https://github.com/mixxxdj/mixxx/issues/9896)
  * [Issue 12753](https://github.com/mixxxdj/mixxx/issues/12753)

* **Other docs or links:**
  * [GSoC Proposal](https://docs.google.com/document/d/1QAe8mvZFKkFnG--02kuzSagzea6_ed5g3VfjuSyAY7w/edit?usp=sharing)

> Proposal to add a column in the Track Library that shows how harmonically close each track is with a Target
> Track (say, from a deck) if both tracks are played together, at the same tempo.

## Why

When mixing two tracks, we usually play them both at the same BPM. This means that one track may need to be tempo
stretched, relative to the other. When Keylock is off, the tempo stretching will result in a pitch shift.
Because of this pitch shift, we cannot solely look at the Key column to find compatible tracks.
We need to account for the BPM as well. The similarity column will use Key and BPM information of both tracks.

### Pitfalls of the current solution

We have information about Key and also the Keywheel. However, if two tracks are at different BPMs, it becomes
difficult to use this info to predict the compatibility of tracks when they're time stretched to the same tempo.

## Goals

Goals and use cases for the solution as proposed in [How](#how):

* Make it easier to find harmonically compatible tracks to the one playing in deck
* Make it easier to predict the compatibility of two tracks when preparing sets

### Audience

Users interested in creating harmonic mixes.

## Non-Goals

*

## How

### Finding the target track

In the initial version, Mixxx will use the Sync Leader as the target track,
or the track from the Preview Deck if it is present.
The Preview Deck will take precedence over the Sync Leader.

### Computing Compatibility

Python Pseudocode:

```python
import math

# Track1: Owl City - Lucid Dream
key1 = 3 # Eb, if C is 0, C# is 1 and so on...
bpm1 = 128
# Track2: Alan Walker - Alone
key2 = 10 # Bb
bpm2 = 97

def pitch_delta_on_tempo_stretching(old_bpm, new_bpm):
  return math.log2(new_bpm / old_bpm) * 12 # change of pitch in semitones

def normalize_pitch_value(val):
  return (val + 12) % 12 # confine the pitch to the [0-12) range

def pitch_diff(key1, bpm1, key2, bpm2):
  # calculate the pitch delta for each track when stretched to a tempo of 100BPM
  delta1 = pitch_delta_on_tempo_stretching(bpm1, 100)
  delta2 = pitch_delta_on_tempo_stretching(bpm2, 100)

  # get the resulting key for each track at 100BPM
  res_key1 = normalize_pitch_value(key1 + delta1)
  res_key2 = normalize_pitch_value(key2 + delta2)

  # now return the pitch difference when both tracks are played at the same tempo
  return normalize_pitch_value(res_key1 - res_key2)

pd = pitch_diff(key1, bpm1, key2, bpm2)
print("Pitch Difference:", pd) # Output: 0.19895410624553378

```

For the examples above, we find that both these tracks are almost in the same key (with a 20 cent difference)
when they’re played at the same tempo.We would not have known this by looking at the keys or the BPMs independently.
This method of calculating harmonic compatibility gives users a deeper insight on which tracks will work well.

Harmonically compatible tracks do not need to end up in the same key. We can account for this using the Circle of Fifths.
Here’s the continued Python pseudocode:

```python
circle_of_fifths = [
    0,  # C
    7,  # G
    2,  # D
    9,  # A
    4,  # E
    11,  # B
    6,  # Gb
    1,  # Db
    8,  # Ab
    3,  # Eb
    10,  # Bb
    5,  # F
]

def circle_of_fifths_distance(note):
    # distance of given note from note 0
    # it's a circle, so the shortest distance may be from the end of the array
    return min(circle_of_fifths.index(note), 12 - circle_of_fifths.index(note))

def compatibility(pitch_difference):
    rounded_pitch_diff = round(pitch_difference)
    cents = abs(rounded_pitch_diff - pitch_difference)
    # get a rank from 0 to 6, lower means more compatible
    cof_rank = circle_of_fifths_distance(rounded_pitch_diff)
    return cof_rank, cents

comp = compatibility(pd)
print("Rank:", comp[0], "Detune:", comp[1]) # Rank: 0 Detune: 0.19895410624553378
```

This gives us a rank, where 0 means the tracks will be in the same key,
and 6 means the tracks will sound most dissonant together.

We also get a detune value to find out how off tune the tracks will be in cents.

TODO: The algorithm needs to be able to take in tuning information, for non 440Hz tuning standards.

### Displaying The Information

A similarity column will be added, which will allow for sorting and ranking search results.
The values may be encoded as colors taken from a fire, in the range of black to bright yellow.
An option will be provided to display similarity as a percentage string, for accessibility.

The value will be calculated using the detune and rank. It will be a kind of sinus curve with a tip at
a full match and at compatible keys and dips where the key is out of tunes at the 50 cents region at least.

## Alternatives

The section stating potential alternatives. Highlight the objections reader should have towards your proposal as they
read it. Tell them why you still think you should take this path.

1. This is why not solution Z...

## Action Plan

Similarity formula
- extract the library data from mixxxdb.sqlite, import it in python for easy prototyping
- test what values of detune are acceptable, and how this relates to the rank
- create a formula that models the relation of similarity with detune and rank

Similarity Column UI
- create a new column, and a delegate for it
- get a freely licensed (eg. creative commons) Fire icon, and render it in the column
- make the fire change color based on the output of similarity formula
- add an option to keep the column blank (when there is no similarity value)

Write simarity algorithm in KeyUtils. It should take in Key and BPM of two tracks,and return similarity based on formula created above.

Target Track - Sync Leader
- create an `updateTargetTrack` function that is called for these events:
	- sync leader enabled / disabled
	- user changed sync leader deck
	- deck track changed
	- deck pitch shifted using Rubberband
- create a `targetTrack` value, that holds the Key and BPM, and is available to `updateTargetTrack` to modify, and to `BaseTrackTableModel::roleValue` where the similarity value will be computed for each track. It will need to be able to hold the info that no target track is available.

In `BaseTrackTableModel::roleValue`, compute the similarity for the current track with the target track, and return this value to the delegate.
