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

The MIMI Policy Envelope is the initial set of permissions and capabilities for
users to make use of in a room. This policy is made enforceable through a
separate *policy control protocol*, as described by {{!I-D.barnes-mimi-arch}}.


--- middle

# Introduction

The primary objective for the More Instant Messaging Interoperability (MIMI)
working group is to specify the needed protocols to achieve interoperability
among modern messaging providers. The protocols which make up the "MIMI stack"
are described by {{!I-D.barnes-mimi-arch}}.

One of those protocols, the *policy control protocol*, outlines how a policy
envelope is described, shared, and configured. **I-D.TODO.ralston-mimi-signaling**
describes such a protocol. This policy document is described in context of
that signaling document.

This policy document outlines the permissions and capabilities of users. For
example, which users are allowed to invite others to a room or which types of
messages a given user can send.

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

Other terms include:

*Action*: Something a user does in the context of a room. For example, invite
another user or send a message to the room.

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

## Participation

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

# User Participation

User participation is tracked as `m.user` state events in a room. The `content`
for such an event has the following structure in TLS presentation language
format ({{Section 3 of RFC8446}}):

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
} MUserContent;
~~~

**TODO(TR): Continue describing legal participation state transitions. Maybe introduce join rules here?**

# TODO Sections

* History visibility
* Join rules (if not covered above)
* RBAC using MSC4056
* ???

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
