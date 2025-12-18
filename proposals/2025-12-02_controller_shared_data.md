# Controller Shared Data

* **Owners:**
  * `@ywwg`
  * `@acolombier`

* **Implementation Status:** `Partially implemented`

* **Related Issues and PRs:**
  * [Ability for controller to share data at
    runtime](https://github.com/mixxxdj/mixxx/pull/12199)

> TL;DR: Allow controllers mappings to set and retrieve variables of different
> data types in order to exchange them between the controller code and the
> engine. Think: ControlObjects of arbitrary type that controllers can declare.

## Why

There are multiple scenarios where controller mapping scripts need to share and
access data outside the container of their own controller script engine. This
includes situations where some controllers expose more than one USB interface
that need to communicate with each other, or when a DJ connects multiple
instances of the same hardware to Mixxx.

### Pitfalls of the current solution

ControlObjects are the normal way we share data, but:

* Controllers can't declare control objects.
* Putting controller-specific data inside Mixxx itself would be bad and lead to
  bloat.
* Controllers need to share more types of data than just double values.

## Goals

Goals and use cases for the solution as proposed in [How](#how):

* Namespacing: Allow different controllers to declare same-named data objects
  without risk of collisions
* Support controllers with screens that require communication from HID to
  separate Bulk USB devices (Traktor S4 MK3).
* Build a data model foundation for users with multiple instances of the same
  controller (CDJ-2000).
* Ensure that the API could support the features we may wish to add in the
  future, such as global namespace, without breaking controller mappings that
  use the API defined here.

### Audience

Users of modern controllers or multiple controllers will appreciate this work.
Specifically, this work is required to fully support the Traktor S4 MK3, which
has separate USB interfaces for the controller and the two screens (two total USB
interfaces).

## Non-Goals

* We do not intend to fully support the multi-device scenario yet, that needs
  further design to associate specific devices with specific controller
  configurations.
* We do not intend to immediately support "global" objects accessible across
  namespaces, like "universal shift".

## "Universal Shift"

"Universal Shift" refers to the idea that a shift button pressed on one
controller can be detected by any and all other controllers. This may be a
useful use-case but creates a lot of difficulties, so for now Universal Shift is
out of scope for this first implementation.

## How

### Data Object

We will create a central object inside Mixxx that contains a triple-keyed map:

* Namespace (string)
  * Entity (string)
    * Key (string)
      * Value ("SafeValue")

#### Namespace

`Namespace` is a string that is unique to each controller **mapping
definition**. All hardware mapping configurations must specify distinct
namespaces.

Multiple device support is still out of scope for this proposal, but we
anticipate that this design can expand to support that use case. For example,
the namespace MAY have a suffix appended to distinguish distinct devices based
on an automatically-detected unique device identifier, e.g. two CDJ-2000's could
have namespaces like `CDJ_2000-ABCDEF` and `CDJ_2000-FEDBCA`. This suffix would
be applied in C++ code outside of the awareness of the controller mapping.

If a unique serial number cannot be determined at runtime, a special controller
preference (defined in Mixxx, not controller mappings) could be used to map
which controller is associated with which device through a new API, and this
value will be passed to controller mappings.

#### Entity

`Entity` is a logical value defined by the controller mapping definition. It can
be like a Mixxx-style group ("`[Channel1]`") but it can be any of the
preselected names listed below. Many controllers will want to define something
like `deck1` to refer to a device that can itself be assigned to multiple mixxx
channels.

The controller mapping decides how these entities behave and Mixxx does not
enforcement of them. To reiterate: even if an "entity" "looks like" a Mixxx
group, it is not.

Here is the proposed initial description for entity definition, which is
anticipated be extended over time.

```typescript
declare Entities {
type Mixer = 'mixer';
type Main = 'main';
type Library = 'library';
type Decks = 'deck1' | 'deck2';
type Channels = 'channel1' | 'channel2' | 'channel3' | 'channel4';
type Controller = 'controller';
}
type Entity = Entities.Mixer | Entities.Main | Entities.Library | Entities.Decks | Entities.Channels | Entities.Controller;
```

Numbered entity names could also be validated with a regular expression such as
`deck[0-9]+`.

'Entity' values are enforced by Typescript library code before function calls
are handed off to C++.

#### Key

`Key` is a logical value defined by the controller mapping definition. It could
refer to a button, light, knob, or abstract name.

The controller mapping decides how these keys behave and Mixxx does no
enforcement of them. Similar to "entity", keys bear no relation to equivalent
Mixxx keys.

#### Value

In the engine, the `value` is stored as a QVariant, however we want to only
support a limited set of types in Javascript / Typescript. For the first
implementation, we will support bool, number, and string, and Arrays of those:

```typescript
type SafePrimitive = string | number | boolean | null;

type SafeValue =
  | SafePrimitive
  | SafePrimitive[];
```

The list of allowable types can be expanded as needed, but we want to be sure
that the shared data system does not become a "bag of bytes" message bus for
large pieces of data like bitmaps or code, nor should it be used to circumvent
intentional limitations or gaps in the overall javascript framework.

#### Example

The shift button on the left side of a Traktor S4MK3 would be stored this way:

pseudocode -- not final naming:

`m_shared_value["S4MK3"]["deck1"]["shift"] = true`

### API

The shared data API should be roughly the same across Controllers, QML, and C++.
The primary difference is that C++ will have access to the namespace value at
all times, whereas controllers and QML will have that value elided.

The controller javascript has access to the shared value object through three
functions:

#### Get

excuse the pseudo-js:

`engine.getSharedValue(entity: Entity, key: string): SafeData?`

`namespace` is set automatically by the engine code, so controllers can't get
that wrong.

This function returns error if the value is not found.

#### Set

`engine.setSharedValue(entity: Entity, key: string, value: SafeData): void`

`namespace` is set automatically by the engine code, so controllers can't get
that wrong.

Calling "set" triggers "updated" signals to all subscribers across the engine,
QML (e.g. controller screen code), and controllers.

Controllers do not get notified about updates they initiated themselves, to
prevent circular signal loops.

#### Updated

Controllers can subscribe to notifications about data updates via a method
similar to how they subscribe to engine Control Object updates:

(not final naming)

`function makeSharedValueConnection(entity: Entity, name: string, callback: CoCallback): ScriptConnection | undefined;`

In this first implementation, controllers can only subscribe to updates for
their own namespace.

### Possible future directions

The following are possible future extensions to this proposal that are currently
out of scope and will not be implemented in the first version, but we want to
make sure to leave room in case we add them in the future:

#### Cross-device communication / subscription

There is a possibility controller authors may want access to signals sent from
other controllers, for instance a "universal shift" button. For this purpose we
may choose to allow controller to "subscribe" to updates from other namespaces,
or all namespaces, in a read-only fashion. In this scenario, controllers would
not have the ability to write updates to namespaces outside their own. This may
be brittle because controllers would need to know about all possible valid
namespaces, so this feature would need more care to make it maintainable.

#### "Global" namespace

Another possibility is that we may want a "global" namespaces that all
controllers can read and write to. This would be another way to support a
"universal shift" button. This would have to be carefully managed to prevent
collisions between controller configs. One way to do this would be to "bless"
specific entities and keys for the global namespace, and controller authors
would have to add their requested global entity/key to Mixxx.

## Alternatives

The original implementation did not have entities and keys and instead had a
single namespaced data blob that controllers had to manage themselves. This
approach requires a lot more work on the part of the controller author to merge
and manage the data object.

## Action Plan

1. Build on the existing PR to implement the desired API
2. Rewrite the Traktor S4 MK3 mapping to support the new API.
