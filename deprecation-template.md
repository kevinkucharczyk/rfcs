---
stage: accepted
start-date:
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: # update this to the PR that you propose your RFC in
project-link:
---

<!---
Directions for above:

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
-->

# <RFC title>

## Summary

> One paragraph explanation of the deprecation.

## Motivation

> Why are we doing this? What are the problems with the deprecated feature?
What is the replacement functionality?

## Transition Path

> This is the bulk of the RFC. Explain the use-cases that deprecated functionality
covers, and for each use-case, describe the transition path.
Describe it in enough detail for someone who uses the deprecated functionality
to understand, for someone to write the deprecation guide, and for someone
familiar with the implementation to implement.

## How We Teach This

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?
Does it mean we need to put effort into highlighting the replacement
functionality more? What should we do about documentation, in the guides
related to this feature?
How should this deprecation be introduced and explained to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.
There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
