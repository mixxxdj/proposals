# Template

Feel free to copy this file and feel all the details below.

See [README](README.md) for details.

## Detaching the DB-connection from legacy UI for use in all UI's

* **Owners:**
  * `<@author: single champion for the moment of writing. No need to add contributors
    all will be visible in commit or comments history on GitHub.>`

* **Implementation Status:** `Not implemented / Partially implemented / Implemented`

* **Related Issues and PRs:**
  * `<GH Issues/PRs>`

* **Other docs or links:**
  * `<Links…>`

> In this doc we'd like to describe all necessary steps to split the DB-access from the UI, create a centralized DB-access working with templates so every different UI can access the needed DB-queries.

## Why

*Looking at the future, making it possible to incorporate different UserInterfaces (at the moment mainly for QML), we need to have every UI access the same logic (code) of the different features without the logic (codea) that is particular for an (other) (legacy) UI.*


### Pitfalls of the current solution

*All DB-access is nested everywhere in the code. SQL-queries are build in different ways, some are build with the use of Field-/Table-nameVars, others use the actual Field-/Table-names. In different features procedures with the same logic appear, differing in field-/table-names. There are also different search possibilities that need to use the same centralized query-builder. Every UI should use the same routines to get/set information from/to the database with templates adapted wit a source, target & format identifier.*



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

As we proposed a GSoC project, we need to agree on some basics, like structure, name and place of the future code.
There are 2 big parts in the DB-access: DB-actions for features & DB-actions for tracks.
DB-actions for tracks can be devided in track-metadata kept in the library (title, artist, replaygain..) and data kept in seperated tables (cues, location, analyses).


# Current Situation: Library Features (eg crate)
Every library-feature has
- it's own subdir in the library (except playlist).
- a xfeature.cpp (ev based on another class) in which procedures to generate/populate rootview/sidebarsactions/libraryconnections/trackconnections/sumaries/d&d ...
- a xfeaturehelper in which extra procedures for creating/duplicating/renaming a (new) xfeature-item
- a xfeatureschema in which vars are created to hold the actual field-/table-names
- a xfeaturestorage in which extra query-items/vars are created (for sumary/join) or specific adaptations for db-actions are written (deleting empty items / invalid values)
- a xfeaturetablemodel in which the actual queries are written for track manipulation (select/add/remove) and feature options (lock) + capabilities of the feature (mouse actions / d&d...) + temporary views are made
- a xfeaturesummary in which the summary for the feature is defined

Every library-feature calls
- TracksetTableModel: defining the (cache of) columns
- TrackCollection: defining db-connections / track-manipulation / general feature-actions
- TreeItem: To buid the Feature-SideBar-Tree
- BaseSqlTableModel: actual code to create the table based on the temporary view created in the xfeaturemodel
- Data Access Objects (DAO's)
- LibraryFeature with the defaults for various procedures
- LibraryControl: procedures to manipulate the order of tracks inside a feature + actions called in the features to load the track

# Current Situation: track-manipulation
- handling of meta-data
- handling of cues / analyses / location ...

# Current Situation: Flow in UI
At Start of Mixxx
-> Library is called
-> Library calls every feature activated in the preferences
-> Every feature that has a side-bar-tree calls it's feature -> tablemodel -> BaseSqlTableModel
-> On selecting (activating) an item in the feature's side-bar-tree: library-side-bar -> feature -> tablemodel -> basesqltablemodel // library-side-bar -> feature -> widget tracktableview (and other view widgets) calling  -> libraryview

# Ideal World situation: structure
at the moment we have a subdir in src: database.
in this subdir we should have devisions for features & tracks
in the features we need 
- a class containing the select procedure template for features
- a class containing the insert procedure template for features
- a class containing the delete procedure template for features
- a class containing the change procedure template for features
...
in the tracks we need 
- a class containing the search procedure template for tracks
- a class containing the delete procedure template for tracks
- a class containing the change procedure template for tracks
...
# Ideal World situation: templates
there could ve differend templates needed for the same action, but less if we define the input vars correct.
each template should accept:
- source (depending on the source the inputformat can change)
- target (depending on the target the outputformat can change)
- actiondescription (debug)
- inputdatabasefieldsnameslist + inputdatabasefieldsvalueslist (eg to use crates and crate_tracks)
- inputOptionsActivatedlist (eg: use ' join track_locations')
- sortoptions 
- output model / outputfieldslist / outputformat / outputtarget 


* that would all be “really” “amazing” “great” (sorry)

