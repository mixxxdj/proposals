# Crate Hierarchies / Nested Crates / Subcrates

* **Owners:**
  * @cr7pt0gr4ph7

* **Implementation Status:** `Not implemented` (in Mixxx main branch) / `Implemented` (in @cr7pt0gr4ph7's code)

* **Related Issues and PRs:**
  * Nested Crates / Crate Folders
    * PR: mixxxdj/mixxx#16835 (utilities for sidebar item drag & drop) &mdash; ca. 2026: Short summary of the discussion from @cr7pt0gr4ph7's point of view, and the reason this proposal document was written.
    * Discussion: mixxxdj/mixxx#5634 (multi level crates) &mdash; ca. 2010 - 2024: Last preceding discussion between @cr7pt0r4ph7, @daschuer and @rons0 about the approach.
    * Discussion: mixxxdj/mixxx#7481 (hierarchical library display)  &mdash; ca. 2010: Another mention of hierarchical crates.
    * PR: mixxxdj/mixxx#10997 by @ywwg (Crate archives) &mdash; ca. 2022: One of the initial implementations of a single archive folder, abandoned by the author @ywwg in 2025 in favor of having user-defined hiearchies.
    * PR: mixxxdj/mixxx#13978 by @Eve00000 (Grouped crates) &mdash; ca. 2024: Alternative implementation based on storing the crate's full path in its name. Stalled since the start of 2025.
    * PR: mixxxdj/mixxx#4182 by @Eve00000 (SearchCrates) &mdash; ca. 2025: Tangentially related, orthogonal proposal of adding dynamic crates whose content is determined by a search query.
   * Nested Playlists / Playlist Folders
     * PR: mixxxdj/mixxx#16669 (Subfolder for Playlist) &mdash; 2026: Implementation of folders for playlists.
     * Issue: mixxxdj/mixxx#3413 (Subfolder for Playlist) &mdash; 2024: Feature request for playlist folders.
     * PR: mixxxdj/mixxx#14352 by @Eve00000 (Grouped playlists) &mdash; ca. 2024: The same as "Grouped crates", but for playlists. As the proposal you're reading is focused on crates, we'll ignore playlists for now.

* **Other docs or links:**
  * `<Links…>`

> We need a way to organize crates into a hierarchy, so we can deal with large numbers of crates in a more useful way.
> There's a reason why filesystem have folders instead of a flat hierarchy.
> 
> The demand & usefulness of this feature is evident from the fact that there has been continued discussion & proposals for the last 16(!) years,
> that always got stuck in limbo for one reason or the other. This proposal is an attempt to _finally_ get something out the door,
> to be able to iteratively improve it afterwards.


## Why

We need a way to organize crates into a hierarchy, so we can deal with large numbers of crates in a more useful way,

### Pitfalls of the current solution

The current solution boils down to manually naming crates in a hierarchy.

* There is **no easy way to _move_ multiple crates** at once within the hierarchy. Every change of the hierarchy requires manually changing the title of each crate one-by-one.
* There is **no way to _temporarily hide_ crates** that are completely irrelevant to the DJ set you are currently playing. This gets kind of bad if you play on very different occasions:
  * For a wedding, I may have 15 different crates for different stages of the evening & mood.
  * For a second wedding, I may have another 10 crates, as they have requested different music.
  * I also play music for West Coast Swing, and have 20 crates for different moods, difficulty levels and genres.
  * And I also play Ballroom & Latin music.
  * And I also play Salsa music.
  * Due to this, I have over 150 different crates.
* There's **no way to _manipulate_ a number of crates** at once (e.g. export and then delete all crates associated with a wedding after that wedding is over).

> There are also some more UX / screen real estate problems once the number of crates hits a certain threshold:
> * The crate tree in the sidebar becomes large & unwieldy (which this proposal would solve), making drag & drop as well as finding the right crate a nightmare.
> * The "Crate"/"Move to Crate" context menu item for tracks becomes unwieldy & takes up a lot of screen space (which this proposal would not make worse that it is right now anyway, and is probably best solved in a separate PR/proposal).
> * A potential "Move to Parent Crate" context menu item for crates has the same problem, but could be solved using the same approach, once one has been found.
  
## Goals

Goals and use cases for the solution as proposed in [How](#how):

* Provide the most basic implementation of hierarchical crates that gets the job done.
* Provide drag & drop support for editing crate hierachies.
* Allow expanding/collapsing nodes in the crate hierarchy.
* Defer every decision that can be deferred until after this MVP.

This is also the time to decide on "Crates + Folders" vs. "Nested Crates"/"Subcrates".
This is more of a personal preference thing than anything else, so let's just have a vote and get it over with.

### Audience

The target audience that this change relates to:

* End users that want to organize their crate library.
* Maintainers that care about the backend organization of the code.

## Non-Goals

Explicit non-goals for the solution as proposed in [How](#how), which could all also be called "Future work":

* Deciding on the **final user-facing name** for what to call those crates nested inside another crate. Let's just go with "Sub-Crate", and then have separate vote whether we should call it "Child Crates", "Nested Crates", "Inner Crates" or something else instead. The author of this proposal is not partial to any particular name.

* Solving the **non-sidebar-related UX problems** that occcur once the number of crates hits a certain threshold:
  * The "Crate"/"Move to Crate" context menu item for tracks becomes unwieldy & takes up a lot of screen space (which this proposal would not make worse that it is right now anyway, and is probably best solved in a separate PR/proposal). We can have a look at Spotify for a potential solution, as they are solving are very similar problem, and are solving it by combining a search box with a "most recently used" list.

  * A potential "Move to Parent Crate" context menu item for crates has the same problem, but could be solved using the same approach, once one has been found.

* Making a final decision on **whether tracks from nested crates should or should not be shown** in the parent crate, or whether the user should be able to make that choice.

* Deciding what it should mean to **mark a parent crates as an "Auto DJ source"**. The answer is likely to be the same as for the point above.

* Finding a final solution (or deciding it is not needed) for **how to indicate the currently selected track is part of a not-currently-visible subcrate** in the sidebar.
  For now: You are likely to have the crates you are currently working with visible anyway, and likely to not care about the ones you cannot see right now,
  but that is a separate discussion.

  (Context: Crates and playlists are displayed in bold when they contain the track that is currently selected in the table view).
* Making the solution generalizable to playlists (though it likely is).
* Providing "Export All Sub-Crates to Playlist" or similar features that build upon this.

* Finding a better solution for the deletion of parent crates that: A) Allows users to delete a whole crate hierarchy if desired, but B) prevents them from accidentally deleting the "wrong" 50 crates, or at least gives them an easy way to restore them for now.
  * We could evaluate the option moving deleted crates to a recycling bin by marking them with a `deleted` flag, which would also allow us to implement an undo.


## How

* Crates can be nested inside other crates (like a file inside a folder).
* Such nested crates are displayed in the sidebar tree in the obvious way, i.e. as children of their parent crate.
* Nested crates (which we'll call "Sub-Crates") can be created the following ways:
  * By dragging an existing crate (B) onto a new parent crate (A) in the sidebar.
  * By selecting an existing crate (A) in the sidebar, and choosing "Create New Sub-Crate" from the context menu to create a sub-crate (B).
  * By selecting an existing crate (A) in the sidebar, and choosing "Import Sub-Crate" from the context menu to create a sub-crate from a playlist file (B).
* Selecting a crate in the sidebar will (for now) only show the tracks in that crate, but not those from the parent crates.
  * The suggested next step for better UX would be to provide a checkbox "Hide tracks from nested crates" in the header section
    of the tracks table. By "header section" I mean the area where e.g. the Auto DJ view places its controls.
    This would both make it apparent to the user what is going on, expose them to the existence of that option,
    and allow everyone to use their preferred workflow.
* Deleting a parent crate is only possible when it has no subcrates (for now).
  * This avoids the possible damage accidentally deleting the wrong parent crate could cause.
    We could evaluate the option moving deleted crates to a recycling bin, which would also
    allow us to implement an undo.

On the implementation side:

* The **`crates` table gets a _new_ `parent_id` field** that points to the parent crate, or is NULL for crates at the root of the hierarchy.
* The **uniqueness requirement on `crates.name` is _fully removed_** (i.e. we do NOT place a new uniqueness requirement on the tuple `(crates.id, crates.name)`)
  to allow easier implementation of e.g. the recycling bin, undo, deletion as well as a few other scenarios, and to avoid issues during drag & drop.

  Basically, there's no reason why we should have a "Do you want to skip / rename / overwrite ...?" dialog when we do not actually need the names to be unique internally.
  Or why we should unecessarily incur the hassle for scenarios like: Delete Crate "A/B", Create a new Crate "A/B", Restore the previously deleted crate from the recycling bin into its original location.

## Alternatives

### Alternative 1: Implicitly grouping crates by name

One alternative proposal, as put forward by @Eve00000 in mixxxdj/mixxx#13978, is to be able to use separators in the name of crates
that are treated as a path. I.e. a crate named `Root A/Parent B/Crate C` is displayed as follows in the sidebar:

```
> Root A
  > Parent B
    > Crate C/
```

Pros:
1. Requires no DB schema change.

Cons:
1. This makes the internal implementation of moving crates around in the tree harder than it needs to be,
   while not providing a discernible benefit.
2. It is basically equivalent to either "Subcrates" (this proposal) or "Crates + Folders" (Alternative 2).


### Alternative 2: Crates + Folders

The initial suggestion put forward by the author of this proposal (@cr7pt0gr4ph7) was 

Pros:
1. The differentiation between "Leaf Nodes" and "Folders" may (or may not) allow us to do better organize
   the crates in the UI when displaying them in the context menu.

Cons:
1. The DB schema is more complicated due to the separate `crate_folders` table.
2. The implementation is more complicated due to having to deal with two different entity types in the sidebar tree,
   and there's a lot of duplication going on (e.g. `CrateId`/`CrateFolderId`/`CrateOrFolderId`).
4. The user has to mentally deal with two different entity types ("crates" and "folders") instead of only one.

## Action Plan

The tasks to do in order to migrate to the new idea.

- [ ] Decide on "Nested Crates"/"Sub-Crates" vs. "Crates + Folders" (@all maintainers)
* [ ] Provide the infrastructure for working with a nullable `parent_id` column
  * [ ] mixxxdj/mixxx#16829 (DbId: Add `DbId::toVariantOrNull()` method for working with nullable database IDs)
* [ ] Provide the infrastructure for drag & drop within the sidebar
  * [ ] mixxxdj/mixxx#16835 (TreeItemModel: Add `TreeItem::getUrl`/`setUrl`, `TreeItemModel::kUrlRole` and `SidebarModel::UrlRole`)
  * [ ] mixxxdj/mixxx#16869 (Sidebar: Provide infrastructure for drag & drop of sidebar items) &mdash; the current implementation only allows for D&D _onto_ sidebar items, but not for the D&D _of_ sidebar items. This will also fix some bugs/issues with the behavior of the sidebar tree related to D&D.
  * [ ] mixxxdj/mixxx#16870 (Sidebar: Use different times for auto-expand vs. auto-collapse of sidebar items) &mdash; Improvements to the auto-expand/auto-collapse behavior of sidebar items.
* [ ] Provide the infrastructure for restoring the expansion state of sidebar child items (necessary when we rebuild the tree after the list or hierarchy of crates has changed)
  * [ ] mixxxdj/mixxx#16833 (WLibrarySidebar: Add `setChildIndexExpanded`/`isChildIndexExpanded`)
* [ ] Provide an implementation of nested crates after those foundations have been merged
  * [ ] mixxxdj/mixxx#16871 (Feature: Sub-Crates / Nested Crates)

Future work:
* [ ] Update the user-manual with information on "Sub-Crates"
* [ ] Decide on a final user-facing name for "Nested Crates" / "Sub-Crates" / "Inner Crates" / "Child Crates" / ...
* [ ] Decide on a final user-facing name for "Parent Folder" / "Parent Crate" / "Folder" / "Containing Crate" / ...
* [ ] Add a "Move to Parent Crate" context menu item
* [ ] Improve the UX of "Move to Parent Crate" for large numbers of crates
* [ ] Improve the UX of "Move to Crate" for large numbers of crates
