---
title: "More Instant Messaging Interoperability (MIMI) Policy"
abbrev: "MIMI Policy"
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

This document specifies an authorization policy framework for the More Instant
Messaging Interoperability (MIMI) working group's transport protocol. It makes
use of a Role-Based Access Control (RBAC) mechanism to grant/deny permissions
to users, clients, and servers.

--- middle

# Introduction

This document relies on the concepts described by {{!I-D.barnes-mimi-arch}} and
{{!I-D.ralston-mimi-protocol}}.

Policy within MIMI defines who or what is allowed to take a certain action, and
what the allowable actions are. Some examples include whether a given user is
able to send messages or promote/demote other users within the room. This
document outlines the minimum permissions required for interoperability, and how
the Role-Based Access Control (RBAC) mechanism works.

Some actions are enforceable by the hub server or local follower server in a
room, however other actions can only be handled by end clients. Whether a server
can enforce the policy largely depends on the server's visibility of the message
being checked: MLS Private Messages cannot be inspected, and therefore cannot
have policy applied to them by the server. Such messages will need to be checked
by the clients instead.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Terms from {{!I-D.barnes-mimi-arch}}, {{!I-D.ralston-mimi-terminology}}, and
{{!I-D.ralston-mimi-protocol}} are used throughout this document.
{{!I-D.barnes-mimi-arch}} takes precedence where there's conflict.

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

# Types of Senders

> **TODO**: Figure out non-user permission structures. https://github.com/turt2live/ietf-mimi-policy/issues/2

# Permissions {#int-permissions}

Groups of permissions are known as roles. These roles are then assigned to a
user or server as needed. Permissions cannot be assigned without being part of a
role.

Roles do not currently carry aesthetic characteristics, such as a name, badge
color, or avatar.

Roles, and their assignees, are persisted through the AppSync MLS extension.
Changes are proposed with MLS Proposals, and confirmed with MLS Commits. This
uses the `mimiRoomPolicy` `applicationId` defined by {{!I-D.ralston-mimi-protocol}}.

> **TODO**: Define actual example/schema once AppSync is more reviewed by the MLS
> working group. Initial indications are unclear if a diff or "irreducible" blob
> is preferred.

A role *notionally* looks like the following:

~~~
enum {
   /* Iterated later in the document. */
} Permission;

struct {
   select (Permission) {
      /* cases defined later in the document. */
   } permission;
} PermissionValue;

struct {
   PermissionValue permissions<V>;
   IdentifierUri assignees<V>;
   int order;
} Role;
~~~

`IdentifierUri` is as defined by {{Section 5.2 of !I-D.ralston-mimi-protocol}}.

## Calculating Permissions {#int-calc-permissions}

An entity's permissions is the sum of the permissions described by their assigned
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

These permissions are then used to define whether a user can perform the action.

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
   // Whether other users can be added to the room by the role.
   // Default: false.
   add(1),

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

   // The actions this role can take against roles. For example, adding or
   // removing permissions.
   // Default: None.
   roles(5),

   // Whether the assigned entities can send messages to the room.
   // Default: true.
   sendMessages(6), // TODO(TR): This likely needs breaking out.
} Permission;

struct {
   select (Permission) {
      case invite: BooleanPermission;
      case kick: BooleanPermission;
      case ban: BooleanPermission;
      case redact: BooleanPermission;
      case roles: RolePermission;
      case sendMessages: BooleanPermission;
   } permission;
} PermissionValue;

struct {
   // When false, the permission is explicitly not granted.
   byte granted;
} BooleanPermission;

struct {
   // The role IDs that can be affected by this role. This includes adding,
   // removing, and changing permissions.
   // TODO(TR): We might want something more comprehensive.
   opaque affectRoleId[];
} RolePermission;
~~~

> **TODO**: Determine which permissions are needed to fulfill {{?I-D.ietf-mimi-content}}.

## Role Changes

> **TODO**: I believe we need words to describe how to use the role permissions
> described above. Probably something using effective power levels and talking
> about what "add", "remove", and "change" actually mean.

> **TODO**: We also need to specify that the creator has superuser permissions
> until a role is defined/assigned.

# User Participation {#int-user-participation}

> **TODO**: Needs updating considering participation state changes are proposed
> through AppSync now. The concepts around the rules of state changes still apply.

> **TODO**: "Invite" likely needs swapping for "Add".

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

> **TODO**: This is where we'd put server ACLs - https://github.com/turt2live/ietf-mimi-policy/issues/3

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
({{int-join-rules}}).

If the join rule is `invite` or `knock`, the user MUST already be in the joined
or invite state.

If the join rule is `public`, the user MUST NOT already be in the banned state.

Otherwise, reject.

## Knock Conditions

The target and sender of a knock MUST be the same.

If the current join rule ({{int-join-rules}}) for the room is `knock`, the
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

## Join Rules {#int-join-rules}

> **TODO**: Convert to an AppSync-style policy flag. It will need an associated
> permission.

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
} PolicyJoinRule;
~~~

# Security Considerations

> **TODO**: Verbosely describe the security considerations throughout the doc.

# IANA Considerations

None.

--- back
