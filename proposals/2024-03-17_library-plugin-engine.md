## Library plugin engine

* **Owners:**
  * `<@acolombier>`

* **Implementation Status:** `Not implemented`

> TL;DR: This design doc proposes a solution to allow plug-n-play plugins to provide external integration to third party services.

> Note: A PoC is available [here (Mixxx implementation)](https://github.com/acolombier/mixxx/tree/poc/lbrary-module) and [here (Demo plugin)](https://github.com/acolombier/mixxx-plugin-librespot?)

## Why

To provide a way to allow external services or software to be integrated in Mixxx in a ad-hoc way, in order to:
- include software and solution with license that prevent distribution as part of the Mixxx software
- support library provider which relies on opinionated data model, incompatible with the Mixxx one
- allow more integrations without requiring Mixxx Core Team to provide support and maintenance for them

### Pitfalls of the current solution

The current solution requires implementing the integration into the Mixxx codebase, and implies that the Mixxx Core team will provide efforts to maintain and test the integration going forward. This also limits integration to rely on GPLv2 licenses only, as well as be implemented in C/C++ as it stands.

## Goals

The goal of this proposal is to:

* Create a generic standard API to provide integration between Mixxx library and a third party.
* Provide a way to manage plugins in the easiest way possible for the user

### Audience

The audience of this change is:
- Developers or organisations that want to integrate a solution or service in Mixxx.
- Non-technical user who want to use solutions or services for which the latter have provided plugins.

## Non-Goals

* Migrate existing third party integration such as Serato or RekordBox features from Mixxx codebase to plugins.
* Provide any other feature than sound library sourcing.
* Create a built-in "plugin store" managed by Mixxx (search, install, download, ...).
* Create interaction on Mixxx -> Plugin direction (e.g custom library export, streaming/recording).

## How

### API

Since Mixxx already embeds protobuf, the most appropriate choice appears to be gRPC with Protobuf. The latter pair provide solid solution to make strongly typed interfaces, which are easy to control over time, as the specification involve, and will provide a strongly typed models, with memory efficiency. It will provide asynchronous operations out of the box. Finally, there is many supported clients for many languages, which will allow the community to build plugin using languages they want to.

On this API, Mixxx will act as a client of it, and the plugin will provide a server. In production, connectivity could be made via standard input/output, which provide the benefit of being fairly standard across different operating systems. Additionally, in development mode, Mixxx will be able to interface with plugin via Unix socket or named pipe on Windows.

#### Specification

> Note: this is a state of the spec after the PoC. There is likely additions to come, but this states the bare minimum and the current topology

<details>

<summary>gRPC spec</summary>

```protobuf
syntax = "proto3";

package mixxx.plugin;

// Plugin service definition.
service PluginService {
  // Perform service discovery
  rpc Manifest(ManifestRequest) returns (ManifestReply);
  // Browse a node of the plugin tree
  rpc Browse(BrowseRequest) returns (BrowseReply);
  // Propagate a user event and fetch side effects
  rpc Event(UserEvent) returns (stream SideEffect);
}

// Service discovery request.
message ManifestRequest {}
// Service discovery reply.
message ManifestReply {
  // The friendly name of the "feature" displayed in Mixxx sidebar. This allows dynamic title to be provided
  string name = 1;
  // The protobuf version this plugin was written with. THis allows Mixxx to detect incompatible plugin
  string proto_version = 2;
  // The icon of the "feature" displayed in Mixxx sidebar. This allows dynamic icon to be provided
  bytes icon = 4;
}

// Node browsing request. If no node, browse the root node.
message BrowseRequest {
  optional Node node = 1;
}
// Node browsing reply. If no node, browse the root node.
message BrowseReply {
  repeated Node children = 1;
  oneof brose_reply_oneof {
    Playlist playlist = 2;
    string view = 3;
  }
}

// Extra context menu item to be added for the current node
message ContextMenuItem {
  enum State {
    CHECKED = 1;
    UNCHECKED = 2;
    NORMAL = 3;
  }
  string ref = 1;
  string label = 2;
  bytes icon = 3;
  State state = 4;
}

// Tree node header definition
message Node {
  string id = 2;
  string label = 3;
  bytes icon = 4;
  repeated ContextMenuItem contextMenuitems = 5;
}

// Events to be sent by Mixxx to indicate user interactions with the plugin views and resources
message UserEvent {
  // A click on a plugin view. (TBC)
  message Click {
    int32 x = 1;
    int32 y = 2;
  }
  // A form submission from the plugin view. (TBC)
  message Submit {
    string id = 1;
    bytes payload = 2;
  }
  // A context menu item was triggered
  message ContextMenu {
    string ref = 1;
    ContextMenuItem.State state = 2;
    Node node = 3;
  }
  oneof view_event_oneof {
    Click click = 1;
    Submit submit = 2;
    ContextMenu contextMenu = 3;
  }
}
// Side effects of an event
message SideEffect {
  // Invalidate a given node
  message Invalidate {
    Node node = 1;
    BrowseReply data = 2;
  }
  // Send an arbitrary message the plugin view
  message Message {
    bytes data = 1;
  }
  oneof side_effect_oneof {
    Invalidate invalidate = 1;
  }
}

// An ordered collection of track
message Playlist {
  // Unique reference for this playlist. There is no constrains of the string format, it can be a UUID, a path or a custom resource name
  string ref = 2;
  // This indicates if Mixxx should manage the search locally or if search queries should be handled by the plugin
  enum SearchMode {
    NONE = 0;
    DELEGATED = 1;
  }
  SearchMode search = 4;
}

// Playlist service
service TracklistService {
  // Fetch a playlist
  rpc Get(PlaylistRequest) returns (PlaylistResponse) {}
  // Fetch the content of a playlist
  rpc FetchContent(FetchContentRequest) returns (stream Track) {}
}

// Playlist fetching request
message PlaylistRequest {
  string ref = 1;
}

// Playlist fetching reply
message PlaylistResponse {
  Playlist playlist = 1;
}

// Playlist content fetching request
message FetchContentRequest {
  // The playlist to fetch from
  Playlist playlist = 1;
  // The maximum number of track to fetch
  int32 limit = 2;
  // The offset from where to fetch
  int32 offset = 3;
  // Search query to filter this playlist. Only relevant if the playlist uses SearchMode=DELEGATED
  string query = 4;
}

// Track service
service TrackService {
  // Fetch a track
  rpc Get(TrackRequest) returns (TrackResponse) {}
  // Request opening of a track with a delegated source
  rpc Open(OpenRequest) returns (OpenResponse) {}
  // Read data from a track with a delegated source
  rpc Read(ReadRequest) returns (stream  ReadChunk) {}
  // Seek in a track with a delegated source
  rpc Seek(SeekRequest) returns (SeekResponse) {}
  // Request closing of a track with a delegated source
  rpc Close(CloseRequest) returns (CloseResponse) {}
}

// Track fetching request
message TrackRequest {
  string ref = 1;
}
// Track fetching reply
message TrackResponse {
  Track track = 1;
}
// Track opening request
message OpenRequest {
  Track track = 1;
}
// Track opening reply
message OpenResponse {
  // A unique file descriptor allocated by the plugin
  int32 fd = 1;
  // The file size in bytes
  int64 filesize = 2;
  // The expected MIME type of the file
  string mime = 3;
}

// Track read request
message ReadRequest {
  // A unique file descriptor allocated by the plugin when opening
  int32 fd = 1;
  // Preferred chunk size to use for streamed response
  int64 chunk_size = 2;
  // The maximum number of bytes to read
  int64 max_size = 4;
}

// Track read stream response
message ReadChunk {
  // The bytes read
  bytes data = 1;
}

// Track seek request
message SeekRequest {
  // A unique file descriptor allocated by the plugin when opening
  int32 fd = 1;
  // The seek start position
  uint64 position = 2;
}

// Track seek response
message SeekResponse {
  // The new start position after seeking
  uint64 position = 1;
}
// Track close request
message CloseRequest {
  // A unique file descriptor allocated by the plugin when opening
  int32 fd = 1;
}
// Track close response
message CloseResponse {}


// The track definition
message Track {
  message DelegatedSource {}
  message LocalSource {
    string path = 1;
  }
  oneof track_source_oneof {
    DelegatedSource delegated = 10;
    LocalSource local = 11;
  }
  string ref = 1;
  string title = 2;
  string artist = 3;
  string album = 4;
  bytes artwork = 5;
  string albumArtist = 6;
  string genre = 7;
  string composer = 8;
  string grouping = 9;
  string year = 10;
  string trackNumber = 11;
  string trackTotal = 12;
  int32 timesPlayed = 13;
  string comment = 14;
  double bpm = 15;
  string bpmText = 16;
  string keyText = 17;
  double duration = 18;
  string info = 19;
  string titleInfo = 20;
  int32 sourceSynchronizedAt = 21;
}

```

</details>

### Plugin process lifecycle

In production mode, the plugin process will be spawned (forked) by Mixxx when it starts the plugin. The executable will be chosen in the `extension` settings path subdirectory, and its standard input/output will be piped to interface with Mixxx gRPC client. Standard error will be used for logging.

**TBD: Security/Sandboxing of the process**

### Distribution and filesystem

Plugins should be managed via the Preference window. Options should include:
- Installing/Deleting a new plugin (unpack or delete the `extension` directory)
- Enabling/disabling a plugin (mark the plugin as enabled/disabled in UserSettings)

Future options could include:
- Clearing cache
- Restarting
- Custom settings

> In order to help with the overall QML effort, a "nice to have" would be to implement the new dialog in QML and embed it using `QQuickWidget`

> Note: this is inspired on how Terraform works with providers

Plugins should be distributed as `.zip` and it's contains should look like the following:

```
manifest.yaml
bin/linux_amd64
bin/linux_arm_v7
bin/linux_arm64
bin/windows_amd64
```

All the binaries referenced in the `manifest.yaml` should be packed.

Upon install, the content of the `.zip` should be unpacked in `<SETTING_PATH>/extensions/<PackageFQDN>/`, for example `/home/user/.mixxx/extensions/org.mixxx.demo/`. Irrelevant binaries may be omitted, so a Mac will not have to unpack Linux or Windows executable.

**TBD: Signing plugin archive**

#### Manifest

> To be completed
> Note: this is inspired on how Docker works with multi arch images

```yaml
schemaVersion: v1beta1
package: org.mixxx.demo
name: Demo Plugin
author: Antoine C.
url: https://github.com/mixxxdj/mixxx-plugin-demo
description: |-
  A nice little demo that showcase plugin capabilities.
  See more info on <a href="https://mixxx.zulipchat.com/">Zulip</a>
dist:
- os: linux
  architecture: arm
  variant: v6
  digest: sha256:...
  size: 12048

```

### Mixxx DAO and model change to support non-local files

Currently, interaction with track files is performed using `fileinfo.h`, which is a shim around QFileInfo. Interaction with files is fully synchronous, sometimes performed in the main thread.

### Library unidirectional integration

#### Remote files

One change that will be required to make that available is the change of location to rely on URL instead of local file path. Currently, the URL format suggested is:

```
mixxx.plugin://<PackageFQDN>/<UniqueIdentifier>
```

Example:

```
mixxx.plugin://org.mixxx.demo/my+resource+id
mixxx.plugin://org.mixxx.demo/my:resource:id
mixxx.plugin://org.mixxx.demo/my/resource/id
```

This will enable integration with ephemeral files, which cannot be read directly from the filesystem. Example usage can be cloud storage or services.

#### User interface

Some integration may want to provide minimal user interface (examples include: settings, login, summary). Overall, everything should be made to encourage integration to make the most of Mixxx mechanics (e.g search, browsing, playlist building,...)

The UI is built on top of QML, which will allow the future Mixxx a better integration with plugin, and the ability for plugin developers to reuse Mixxx UI components and create an even more seamless integration

#### Events

Mixxx integration will provide a list of events that will be propagated to plugins in order to add more integration with Mixxx functionality. A non exhaustive list is:

- Context menu interaction
- Add/remove plugin track for playlist or crate
- Play/load track
- Message from UI view (e.g form submission)
- Click in UI view (e.g link or button)

### Mixxx effort on plugin bootstrap

It has been expressed a few time the will to start writing part of Mixxx using other languages, such as Rust. It would be a nice to have to provide some bootstrap repo allowing an easy way to create and automate the distribution of plugins by the community

## Alternatives

### API

JSON RPC was also suggested during the initial discussion, however no further research were pursued in that direction due to:

1. Performance impact from serialisation/deserialisation
2. No built in schema or object specification
3. No builtin versioning solution and breaking change prevention

### Distribution and filesystem

XNL, JSON and TOML were also considered for manifest. There is no major reason for choosing YAMl over those other format. YAML was chosen because of the user friendliness and the formatting imposed by the language.

### User interface

Webviews were also considered to provide UI, however they were quickly disqualified for the following reasons:
- Qt support for QtWidget WebView is reduced
- Interfacing with the view will require a consequent Javascript layer
- Not enough constrains and risk that developers get to rely on a wide range of frameworks for UI rendering, leading to performance issue and difficulties to maintain compatibility

## Action Plan

> To be completed

* [ ] Refactor the `FileInfo` shim to support further file sources
* [ ] Refactor library and DAO to rely on URL instead of local file path
* [ ] Create a feature flag
* [ ] Create a new library feature to wrap plugin integration
* [ ] Implement a demo plugin in Rust
* [ ] Implement the plugin client in Mixxx
* [ ] Implement plugin management in Mixxx preferences
