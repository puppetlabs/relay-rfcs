# Nebula RFCs

RFCs (requests for comments) provide a semi-formal mechanism to bring about
change in a product. In a democratic team where all voices are held in equal
regard, RFCs ensure that anyone can propose changes and have them considered in
a rigorous, structured environment.

* [Accepted RFCs](INDEX.md)
* [Template](0000-template.md)

## Why RFCs?

On the Nebula team, we had been using ad-hoc Google Docs to share ideas and
proposals. Often, these were viewed once and then forgotten or integrated
without an appropriate amount of consideration.

Using RFCs gives our change proposals two additional useful properties:

* Persistence: Git never forgets, and now neither will we.
* Visibility: Lots of people look at the Nebula repository, and pull requests
  have higher visibility than comments on a Google Doc.

## What is an RFC?

RFCs have a somewhat silly name, but for good reason. In the early days of the
internet, lots of people had lots of opinions they were unsure about, and they
circulated memos specifically requesting feedback from their peers on internet
architecture. [Take a look at the very first ARPANET
RFC](https://tools.ietf.org/html/rfc1) for context.

RFCs are not supposed to be overly declarative or proclamatory in tone. They are
a way of gathering consensus for a change. It is okay to have product and
technical questions, to not be sure about importance, and to need the help of
several others to complete an RFC.

When an RFC is accepted, it means we have confidence that an underlying issue
affects our product as described, that we have a promising solution, and that we
actually want to do the work associated with the solution. We don't expect
perfection.

## What types of changes need an RFC?

Generally, changes that affect multiple platform components or that cross many
feature teams deserve an RFC. Backward-incompatible changes also usually deserve
an RFC.

## How to create an RFC

* Create a new branch for your RFC. Something like `rfcs/your-rfc-name`.
* Copy `content/0000-template` to `content/0000-your-rfc-name`. Do not assign a
  number yet; it will be assigned just prior to merging.
* Fill out your RFC with as much detail as you feel is needed. You may choose to
  exclude some sections entirely if you decide they are not relevant. However,
  reviewers may request that you complete removed sections at their discretion.
* Build consensus around an initial set of stakeholders and add them to your
  RFC. If you aren't sure who stakeholders should be for a given change, reach
  out to your manager for recommendations. Note that there may be overlap
  between stakeholder categories (for example, an agreer may also be a decider).
* Open a pull request against `master` and add all stakeholders as reviewers.
* Continue to iterate on the RFC according to review feedback from stakeholders.
* When a quorum is reached among the agreers, one or more of the deciders will
  make a final decision on including the RFC in the product. This should be a
  fairly objective process based on the feedback given.
* If the RFC is approved, a decider will give the RFC the next sequential number
  available, rename the file, update the INDEX, and merge it.
* Once merged, the changes will be included in a reasonable time in the product
  and/or engineering backlog and prioritized as described in the RFC.

## Stakeholder considerations

* There must be **at least two agreers**, and ideally three, for every RFC.
* Although it should go without stating, recommenders cannot also be agreers or
  deciders.
* Diversity of opinion is important, but we aren't trying to implement design by
  committee. You shouldn't be adding the entire team to every RFC.
* Stakeholders are welcome to request additional team members' input as they see
  fit.
* If you are not a stakeholder on an RFC, feel free to leave informal comments.

## Credits

The structure of the template file is based on the [Rust RFC
process](https://github.com/rust-lang/rfcs/). See the [LICENSE](LICENSE.md) file
for more information.
