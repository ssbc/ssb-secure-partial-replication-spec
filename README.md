# SSB secure partial replication

Status: Design phase

This document will outline the steps needed to convert an existing SSB
identity to make it ready for partial replication. First a root meta
feed will be generated from the existing SSB feed as described in
[ssb-meta-feed]. The main feed is linked with the meta feed which
means an application can start using the feeds linked from the meta
feed for partial replication.

The new meta feed should contain the following entries:

```
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'main', id: @main },
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'claims', id: @claims }
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'linked', id: @linked }
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'trust', id: @trust }
```

Claims is a meta feed of claims or indexes, meaning feeds consisting
of a subset of messages in another feed. These can be used for partial
replication. The feeds inside this meta feed should only contain
hashes of the original messages as their content. These linked
messages can be replicated efficiently as auxiliary data during
replication as described in [subset replication].

Applications should create at least two index feeds:

```
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', id: '@claim1', query: 'and(type(contact),author(@main))' }
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', id: '@claim2', query: 'and(type(about),author(@main))' }
```

FIXME: do we need a rotational feed for the latest messages?

Trust is a feed that contains trust ratings within a number of areas
as defined in [trustnet] about other feeds. One areas where this will
be used is for verifications of claims. An auditor verifies claims and
writes those to a new feed within the meta feed:

```
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'claimaudits', id: @claimaudits }
```

The messages should be of the following form:

```
{ type: 'trustnet/claims/verification', latestid: x, id: @claim1, status: 'verified' }
```

Because feeds are immutable once you have verified it up until
sequence x, the past can never change. In order not to create too many
verification messages, a new message should only be posted if claim is
no longer valid. How often claims should be verified is at the
discretion of the auditor.

Linked is a meta feed that contains links to other feeds. The use case
for this is same-as where other SSB ids can be linked. This allows
applications to use this information to create a better user
experience, such as showing notifications for linked feeds, showing
the linking feeds on a profiles page etc. If the linked feed links
back, the feeds are considered transitivily linked.

```
{ type: 'about/same-as-link', from: '@mf', to: '@otherid' }
```

We here should expand on how feeds are replicated within a system with
meta feeds. Existing replication is taking care of by [ssb-friends]
that replicates feeds based on follow or block relations as described
in that repo.

Existing contact messages on the main feed still form the basis for
feed replication. From this basis the parts of the meta feed needed
for the application should also be replicated. Linked feeds should
still be followed, this is to ensure backwards compatibility with
existing clients.

Assuming one wants to do partial replication of a subset of a feed,
one looks in trusted claimaudit feeds to find any that can be
used. Trusted is defined as:

A target feed is trusted if:

 -  One has assigned any positive, non-zero amount of trust to the
    target feed if the trustnet calculation returns it as trusted

A trustnet calculation is performed as:

 -  All first order (i.e. direct) trust assignments are regarded as
    most trusted

 -  All nonzero rankings are clustered using ckmeans clustering on
    their resulting appleseed scores (ranks). The most trusted group
    is calculated by breaking the rankings into 3 clusters, and
    discarding the cluster with the lowest values. This is true if and
    only if there exists at least one direct trust assignment that is
    high enough, as defined by the option trustThreshold
    [0..1]. trustThreshold is defined as having a default value of
    0.50, if unspecified.

If any direct trust assignments are not included in the top 2
clusters, then they are added to the concatenation of the top 2
clusters before returning the result of the trustnet calculation.

If no verified claims are available one should fall back to full
replication of that main feed.

## Open questions

- Should pubs also use meta feeds?
- How do we handle other feed types?
- What initial trust should be assigned and to what?

[ssb-meta-feed]: https://github.com/ssb-ngi-pointer/ssb-meta-feed
[trustnet]: https://github.com/cblgh/trustnet
[ssb-friends]: https://github.com/ssbc/ssb-friends
[subset replication]: https://github.com/ssb-ngi-pointer/ssb-subset-replication
