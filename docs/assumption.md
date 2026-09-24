# Identity Assumption

## Scope

This design covers identities that hold no credential — groups and machine
identities — and how an authenticated identity takes one on.

It does not cover the policy engine that authorizes assumption, or enrollment
into a realm.

## Credentialable Memberships

An identity belongs to a realm through a membership, and a membership states
whether a credential may reference it. A credential's foreign key pins that
value to true on its side, so a membership with the flag unset can hold no
credential, and the flag cannot be cleared while credentials exist.

Three kinds of membership follow from one column:

| Kind | Credentialable | Members |
| ---- | -------------- | ------- |
| Person | yes | none |
| Group | no | one or more |
| Machine | no | none |

Nothing else distinguishes them. A group is an identity that appears as the
group in a membership row of `identity_group`; a machine identity is one that
appears in none.

## Why a Credential-Less Identity Authenticates by Assumption

A credential belongs to an authenticator someone holds. Giving a group a
credential means the people in it share one authenticator, which removes the
per-person attribution that credentials exist to provide.

Assumption keeps credentials personal. An identity authenticates as itself,
then asks for a grant on an identity it is entitled to act as. The evidence of
who acted is the assumption itself, not the credential.

## Assume

`Assume` takes the realm and the identity to assume. The caller's own identity
arrives in the request, verified on the way into the system and trusted by the
handler, the same way `Delete` receives it. What distinguishes `Assume` is that
the grant it writes is for an identity other than the caller's.

The handler:

1. Reads the caller's identity from the request.
2. Asks policy whether that identity may assume the named one in that realm.
3. Generates a grant, writes its digest to the target membership, and answers
   with a proof naming the target.

The proof is exchanged with the issuer exactly as a completed WebAuthn login's
is. Nothing downstream distinguishes the two.

## Grant Contention

A membership holds one grant. Two members assuming one group concurrently both
write it, and the last commit wins; the caller holding the replaced grant is
refused at redemption and assumes again.

This is the same property a person's membership has, met more often because a
group has many members and a machine identity has many workloads. Bounded state
is the reason: one digest per membership, whatever the traffic.

## Audit

The token names the assumed identity as its subject and carries no record of
who assumed it. The chain is reconstructed from the `Assume` and `Issue` spans,
which name the caller, the target, and the realm.

Carrying the chain in the token would put unbounded, recursive state in a value
that is meant to be a fixed set of claims, and policy reads the subject alone,
so nothing evaluates it.

## Privileged Access

Assumption is how a privileged identity is held: nobody carries its authority
continuously, and taking it on is a distinct, authorized, audited act. That is
what privileged access management asks for, and the pieces map onto what
exists.

| Requirement | How it is met |
| ----------- | ------------- |
| Eligibility separate from authority | A group membership row plus a policy permitting assumption. Neither confers anything until `Assume` is called. |
| Activation | `Assume`, which writes a grant and yields a proof. |
| Bounded activation | The token's expiry, bounded by the realm's maximum lifetime. |
| Re-activation rather than extension | A grant is single use, so continuing means assuming again and being authorized again. |
| Time-bound eligibility | A condition on the document granting the assumption, which expresses a lapse date, a schedule, or a recurring window. |
| Approval | A document written by the approver, scoped to one subject, method and target, whose condition bounds how long it stands. |
| Freshness | Policy conditioning assumption on the caller's `last_authenticated_date`. |
| Audit | The `Assume` and `Issue` spans, naming caller, target and realm. |

`last_authenticated_date` advances on the membership that receives the grant,
which is the target's on an assumption and the caller's on a login. A policy
asking how recently the caller authenticated therefore reads a value that only
their own authentication moves.

An approval is a policy document and needs no separate mechanism: the approver
writes one permitting that subject to assume that target, conditioned on a
window, and the requester's next `Assume` succeeds because policy now says so.
What has no home is a *pending* request — asking is out of band, and nothing
records that it was asked.

## Transitive Membership

A group is an identity, so a group may be a member of a group. Membership is
therefore a graph, and whether an identity may assume a group is a question
about reachability rather than a single row.

Two consequences for the engine that answers it: applicable policy accumulates
over every group reachable from the caller, and the walk must tolerate a cycle,
which the schema permits. The only pairing the schema refuses is an identity
that is its own member.

## Machine Identities

A machine identity is created, given no credential, and assumed by whatever is
entitled to run as it. The workload never registers and never holds an
authenticator.

Whether it may assume itself — renewing its own token indefinitely — is a
policy decision rather than a structural one. Denying it is what makes an
expiry mean anything, because the workload must then return to something that
authenticated. Allowing it is supported, and the deployment that wants a
long-lived unattended workload chooses it knowingly.

## Validity

When a membership may be assumed is a policy condition, evaluated before the
commit like every other. Nothing on the membership records it.

That covers what a pair of timestamp columns would — a deprovisioning date, a
start date ahead of which nothing authenticates — and what they could not: a
recurring window, a schedule, or a window that applies to one member of a group
and not the rest.

Enforcement is therefore only as present as the policy engine. Until it exists,
a membership that should have lapsed still authenticates.

## Considered and Rejected

**A credential on a group.** Shared authenticators, no attribution.

**An `act` claim naming the assumer.** RFC 8693 defines it and it nests, so a
chain of assumptions produces a recursive claim whose size is unbounded.
Authorization reads the subject, so the claim would be carried and never
evaluated.

**Grants as rows rather than a column.** It would end contention between
concurrent assumers of one identity, at the cost of unbounded growth in
outstanding grants.
