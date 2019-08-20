- 0000 Experimental Code Protocol
- Start Date: 2019-08-19
- RFC PR: (leave this empty)
- Chisel-lang Issue: (leave this empty)

# Summary
[summary]: #summary

Question: **Should new APIs that might change be placed in a package named *experimental* which requires some user code refactoring
when those APIs become settled, or should they be identified in some other way that does not require such refactoring.**

Comments: **Please comment if you have a take on this.**

Vote: **After the comment period we plan to have a vote on which plan to use.**

The Chisel development team is seeking community feedback on this question as begin working on our next release.

# Motivation
[motivation]: #motivation

The traditional protocol has been to place API code within a `experimental` Scala package.
Users electing to use the new API are alerted to it's possible ephemeral nature by having to import the code using `experimental` package.
While this achieves its goal it presents a problem when the API becomes settled.
At that point in time we would like to place the API classes and methods from the experimental package and move them to the regular codebase.
At a subsequent API changing release the experimental versions will be eliminated.
This can force early adopters who with these API's in their codebase to have to edit many source files in order to continue compiling successfully.
The underlying question here is:
**Is the strong identification of code as experimental code, warning of its potential to change worth the additional work it requires of users when the
code is no longer experimental?**

# Background
[background]: #background

Traditionally some percentage of new development is produces with an API that is deemed possible to change.
Our experience is that only a small percentage of that code actually does change.
This places a burden on early adopters to have to fix their code when everything goes well and we get the API right the first time.
If we do change the API then they obviously need to change their relatively recent code.

# Proposals
[proposals]: #proposals

## Proposal 1 : Continue placing new changeable APIs in to the experimental package

- Users will be very aware when they are using volatile APIs
  - Will generate compile warnings, these can be in-effective given overall number of warnings in typical compile
  - IDEs can also show this
- Users will have to go in and remove and/or change templates when experimental status is dropped
- ScalaDoc will prominent warnings experimental status
- Users have to update code if APIs change

## Proposal 2 : Stop using the experimental package and warn users in some other way
- Users may be less aware (or more likely to forget they are using experimental APIs)
  - Warnings are hard to generate, can also be lost in the noise.
- Users do not need to change the code if the API doesn't change.
- ScalaDoc will prominent warnings experimental status
- Users have to update code if APIs change

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is there a better way to indicate experimental status.
  - A code annotation like @deprecated would seem ideal, but no evidence Scala can do this or be extended to do it.
- Many current users use the wildcard in `import chisel3.experimental._`, lessening the utility of experimental
  - Maybe there's a way to limit this?
