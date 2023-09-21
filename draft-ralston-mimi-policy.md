---
title: "MIMI Policy Envelope"
abbrev: "MIMI Policy Envelope"
category: std

docname: draft-ralston-mimi-policy-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "More Instant Messaging Interoperability"
keyword:
 - mimi
 - policy
 - envelope
 - messaging
venue:
  group: "More Instant Messaging Interoperability"
  type: "Working Group"
  mail: "mimi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mimi/"
  github: "turt2live/ietf-mimi-policy"
  latest: "https://turt2live.github.io/ietf-mimi-policy/draft-ralston-mimi-policy.html"

author:
 -
    fullname: Travis Ralston
    organization: The Matrix.org Foundation C.I.C.
    email: travisr@matrix.org
 -
    fullname: Matthew Hodgson
    organization: The Matrix.org Foundation C.I.C.
    email: matthew@matrix.org

normative:

informative:
   RFC8446:


--- abstract

The MIMI Policy Envelope describes a *policy control protocol* and
*participation control protocol* for use in a room, applied at the user
participation level, as described by {{!I-D.barnes-mimi-arch}}.


--- middle

# Introduction

The primary objective for the More Instant Messaging Interoperability (MIMI)
working group is to specify the needed protocols to achieve interoperability
among modern messaging providers. The protocols which make up the "MIMI stack"
are described by {{!I-D.barnes-mimi-arch}}.

In the stack are a policy control protocol and a participation control protocol.
These two control protocols are described by this document, supported by
**TODO(TR): Link to I-D.ralston-mimi-signaling**.

Policy control is handled through permissions, while participation is managed
primarily through the rules governing `m.room.user`. Together, these control
protocols create this policy document.

When an action is impossible for a server to enforce, such as when a client
operated by a user sends an encrypted instant message, the receiving clients
are responsible for enforcing the remainder of the policy. This may mean, for
example, decrypting a message but not rendering it due to a policy violation.

The concepts of permissions and participation state for a user are deliberately
separated in this policy document. A user's participation state might affect
which permissions they can use, but a user's permissions do not change their
participation in a room.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Terms from {{!I-D.barnes-mimi-arch}} and {{!I-D.ralston-mimi-terminology}} are
used throughout this document. {{!I-D.barnes-mimi-arch}} takes precedence where
there's conflict.

Terms from **TODO(TR): Link to I-D.ralston-mimi-signaling** are used throughout
this document.

Other terms include:

*Rejected*: The action being performed ceases to continue through the remainder
of the send/rendering steps. For a hub server, this means the event being sent
is not added to the room and is not sent to any other server. For a client, this
equates to not rendering or respecting the action.

*Allowed*: The opposite of Rejected. The action is expressly permitted to occur.

*"Engaging with the room"*: The user is able to take some actions and send
messages in the room, provided the remainder of the policy allows them to do
that. The encryption/security layer MAY further restrict a user's ability to
take action. For example, the user might need 1 or more clients to be able to
successfully send a message.

## Permissions Definitions

*Action*: Something a user does in the context of a room. For example, invite
another user or send a message to the room.

*Permission*: A flag which allows (or rejects) execution of an action.

*Role*: A user-defined set of permissions. Users are added to roles to gain the
included permissions.

## Participation Definitions

*Target*: The user affected by a participation state.

*Sender*: The user affecting a target user with a participation state.

*Invited*: The target is given the choice to accept the invite (join the room)
or decline (leave the room).

*Joined*: The target is capable of engaging with the room.

*Left*: The target has either voluntarily chosen to leave the room, or has been
removed with a kick.

*Banned*: The target is kicked and cannot be invited, joined, or knock on the
room until unbanned.

*Knocking*: The sender is requesting an invite into the room. They can either
be welcomed in (invited) or declined (kicked).

*Kicked*: Involuntary leave. The target and sender are not the same user.

# Event Authorization

When a hub server receives an event, and before it adds it to the room, it MUST
ensure the event passes the policy for the room. In the case of this document,
the server MUST ensure the following checks are performed:

1. The event is correctly signed and hashed.
2. The event's `authEvents` include the appropriate event types
   ({{int-auth-selection}}).
3. The `sender` has permission ({{int-calc-permissions}}) to send the event.
4. Any event type-specific checks are performed, as described throughout this
   document.

## Auth Events Selection {#int-auth-selection}

When a server is populating `authEvents`, it MUST include the event IDs for the
following event types. These SHOULD be the most recent event IDs for the event
types.

Note: `m.room.create` MUST always have an empty `authEvents` array.

* The `m.room.create` event.
* The `m.room.user` event for the `sender`, if applicable.
* The `m.room.role_map` event ({{int-permissions}}), if set.
* The `m.room.role` events ({{int-permissions}}) assigned to the user `sender`,
  if any.
* If the event type is `m.room.user`:

   * The target user's `m.room.user` event, if any.
   * If the `participation` state is `join` or `invite`, the `m.room.join_rules`
     event ({{int-ev-join-rules}}), if any.

**TODO(TR): Restricted joins join_authorised_via_users_server?
([GH issue](https://github.com/turt2live/ietf-mimi-policy/issues/1))**

If an event is missing from `authEvents` but should have been included with the
above selection algorithm, the event is rejected.

If events not intended to be selected using the above algorithm above are
included in `authEvents`, the event is rejected. This extends to events which
aren't known or are malformed in `authEvents`.

If an event uses non-current events in its `authEvents`, it is rejected.

# Types of Senders

**TODO(TR): Do we want to send as not-users?
([GH issue](https://github.com/turt2live/ietf-mimi-policy/issues/2))**

Currently this document only supports `sender` being a user ID.

# Permissions {#int-permissions}

Rooms are capable of defining their own roles for grouping permissions to apply
to users. These roles do not currently have aesthetic characteristics, such as
a display name, badge color, or avatar.

Roles are described by an `m.room.role` state event. The state key for the event
is the "role ID", and is not intended to be human readable.

The content for the event has the following structure in TLS presentation
language format ({{Section 3 of RFC8446}}):

~~~
enum {
   // Iterated later in the document.
} Permission;

struct {
   select (Permission) {
      // cases defined later in the document.
   } permission;
} PermissionValue;

struct {
   PermissionValue permissions[];
} MRoomRoleEventContent;
~~~

Users are assigned to roles using an `m.room.role_map` state event, with empty
string for a state key. The content being as follows:

~~~
struct {
   // The role's ID.
   opaque roleId;

   // The user IDs who are assigned this role.
   opaque userIds[];

   // The power level for the role. This is used in cases of tiebreak and to
   // override permissions from another role.
   uint32 order;
} RoleConfig;

struct {
   RoleConfig roles[];
} MRoomRoleMapEventContent;
~~~

Each role ID MUST only appear once in `MRoomRoleMapEventContent.roles`. Each
`RoleConfig.order` MUST be distinct from all other entries. If either of these
checks fail when a server receives the event, the event is rejected.

## Calculating Permissions {#int-calc-permissions}

A user's permissions is the sum of the permissions described by their assigned
roles. When two roles define the same permission (but with different values),
the higher `order` role takes precedence.

For example, if given the following role structure...

* Role A, order 1.
   * Permission A = true
   * Permission B = false
* Role B, order 2.
   * Permission A = false
   * Permission C = false
* Role C, order 3.
   * Permission B = true
   * Permission C = false

... and a user assigned all three roles, the user's resolved set of permissions
would be:

* Permission A = false (takes Role B's value)
* Permission B = true (takes Role C's value)
* Permission C = false (defined by Role B, no conflict with Role C)

These permissions are then used to define whether a user can "send" the event.

## Effective Power Level {#int-effective-power}

In some cases it is required to know the "power level" for a user to solve
tiebreaks. The power level of a user is the highest `order` role they are
assigned with the desired permission set, regardless of value for that
permission.

Using the example from {{int-calc-permissions}}, a user with all three roles
would have the following effective power levels for each permission in question:

* Permission A = 2
* Permission B = 3
* Permission C = 3

## List of Permissions

The full definitions for `Permission` and `PermissionValue` in
{{int-permissions}} is:

~~~
enum {
   // Whether other users can be invited to the room by the role.
   // Default: false.
   invite(1),

   // Whether other users can be kicked from the room by the role.
   // Default: false.
   kick(2),

   // Whether other users can be banned from the room by the role.
   // Default: false.
   ban(3),

   // Whether another user's events can be redacted by the role.
   // Senders can always redact their own events regardless of this permission.
   // Default: false.
   redact(4), // TODO(TR): Do we need this one?

   // The event types the role is able to send.
   // Default: None.
   events(5),

   // The actions this role can take against roles. For example, adding or
   // removing permissions.
   // Default: None.
   roles(6),
} Permission;

struct {
   select (Permission) {
      case invite: BooleanPermission;
      case kick: BooleanPermission;
      case ban: BooleanPermission;
      case redact: BooleanPermission;
      case events: EventTypePermission;
      case roles: RolePermission;
   } permission;
} PermissionValue;

struct {
   // When false, the permission is explicitly not granted.
   byte granted;
} BooleanPermission;

struct {
   // The event type being gated by a permission.
   opaque eventType;

   // When false, the permission to send the event is explicitly not granted.
   byte granted;
} EventTypePermissionRecord;

struct {
   // The event type restrictions. If there are duplicates, the lastmost entry
   // takes priority.
   EventTypePermissionRecord eventTypes[];
} EventTypePermission;

struct {
   // The role IDs that can be affected by this role. This includes adding,
   // removing, and changing permissions.
   // TODO(TR): We might want something more comprehensive.
   opaque affectRoleId[];
} RolePermission;
~~~

## Event Sending Permissions

The `sender` for an event MUST have permission ({{int-calc-permissions}}) to
send that event type, unless the event type is `m.room.user`. User participation
events are handled specifically in {{int-user-participation}}.

The `sender` MUST also be in the joined state to send such events.

## Role Changes

**TODO(TR): I believe we need words to describe how to use the role permissions
described above. Probably something using effective power levels and talking
about what "add", "remove", and "change" actually mean.**

**TODO(TR): We also need to specify that the creator has superuser permissions
until a role is defined/assigned.**

# User Participation {#int-user-participation}

User participation is tracked as `m.room.user` state events. The `content` for
such an event has the following structure in TLS presentation language format
({{Section 3 of RFC8446}}):

~~~
enum {
   invite,  // "Invited" state.
   join,    // "Joined" state.
   leave,   // "Left" state (including Kicked).
   ban,     // "Banned" state.
   knock    // "Knocking" state.
} ParticipationState;

struct {
   ParticipationState participation;
   opaque reason;  // optional reason for the participation state
} MRoomUserEventContent;
~~~

A user is considered to be "joined" to a room if they have a participation state
of `join`. All servers with users in the joined state are considered to be "in"
the room.

Servers which are in the room can send events for their users directly. The
signaling protocol is able to assist servers (and therefore users) in sending
the appropriate participation events until they are able complete the join
process.

## General Conditions

**TODO(TR): This is where we'd put server ACLs.
([GH issue](https://github.com/turt2live/ietf-mimi-policy/issues/3))**

## Invite Conditions

The target user for an invite MUST:

* NOT already be in the banned state.
* NOT already be in the joined state.

The sender for an invite MUST:

* Already be in the joined state.
* Have permission ({{int-calc-permissions}}) to invite users.

Otherwise, reject.

## Join Conditions

The target and sender of a join MUST be the same.

Whether a user can join without invite is dependent on the join rules
({{int-ev-join-rules}}).

If the join rule is `invite` or `knock`, the user MUST already be in the joined
or invite state.

If the join rule is `public`, the user MUST NOT already be in the banned state.

Otherwise, reject.

## Knock Conditions

The target and sender of a knock MUST be the same.

If the current join rule ({{int-ev-join-rules}}) for the room is `knock`, the
user MUST NOT already be in the banned or joined state.

Otherwise, reject.

## Ban Conditions

The sender for a ban MUST:

* Already be in the joined state.
* Have permission ({{int-calc-permissions}}) to ban users.

Otherwise, reject.

Note that a ban implies kick.

## Leave Conditions

Leaves in a room come in two varieties: voluntary and kicks. Voluntary leaves
are when the user no longer wishes to be an active participant in the room. A
kick is done to remove a user forcefully.

When the target and sender of a leave is the same, it is a voluntary leave.

### Voluntary

The user MUST be in the invited, joined, or knocking state.

Otherwise, reject.

### Kicks

The target user for a kick MUST:

* Already be in the joined state.

The sender for a kick MUST:

* Already be in the joined state.
* Have permission ({{int-calc-permissions}}) to kick users.
* Have a higher (and NOT equal to) effective power level with respect to the
  kick permission ({{int-effective-power}}) than the target user.

If the target user is in the banned state, the sender requires permission to
ban users instead (as to ban means to unban as well). This additionally extends
to the effective power level check.

Otherwise, reject.

## `m.room.join_rules` {#int-ev-join-rules}

**State key**: Empty string.

**Content**:

~~~
enum {
   invite,
   knock,
   public,
} JoinRule;

struct {
  // The current join rule for the room. Defaults to `invite` if no join rules
  // event is in the room.
  JoinRule rule;
} MRoomJoinRulesEventContent;
~~~

**Redaction considerations**: `rule` under `content` is protected from
redaction.

# Event/History Visibility

Unless otherwise specified by the event type, non-state events MUST NOT be sent
to a user's client if the history visibility rules prohibit it. State events
are always visible to clients.

When a server is fetching events it is missing to build history, the returned
events are redacted unless the server has at least one user which is able to
see the event under the history visibility rules. The server must then further
filter the events before sending them to clients.

History visibility rules are defined by `m.room.history_visibility`
({{int-ev-history-viz}}), and can only affect future events. Events sent before
the history visibility rule change are not retroactively affected.

Taking into consideration the `m.room.history_visibility` event that is current
at the time an event was sent, a user's visibility of a that event is described
as:

* If the visibility rule was `world`, show.
* If the user was in the joined state, show.
* If the visibility rule was `shared` and the user was in the joined state at
  any point after the event was sent, show.
* If the user was in the invited state, and the visibility rule was `invited`,
  show.
* Otherwise, don't show.

## `m.room.history_visibility` {#int-ev-history-viz}

**State key**: Empty string.

**Content**:

~~~
enum {
   invited,
   joined,
   shared,
   world,
} Visibility;

struct {
  // The current join rule for the room. Defaults to `shared` if no history
  // visibility event is present in the room.
  Visibility visibility;
} MRoomHistoryVisibilityEventContent;
~~~

**Redaction considerations**: `visibility` under `content` is protected from
redaction.

# IANA Considerations

This document as a whole makes up the `m.0` policy ID, as per
**TODO(TR): Link to I-D.ralston-mimi-signaling**.

This document's descriptions for the following event types are registered to the
event types registry in **TODO(TR): Link to I-D.ralston-mimi-signaling**:

* `m.room.role` ({{int-permissions}})
* `m.room.role_map` ({{int-permissions}})
* `m.room.join_rules` ({{int-ev-join-rules}})
* `m.room.history_visibility` ({{int-ev-history-viz}})

--- back
