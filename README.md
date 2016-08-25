# Proposing Changes to go-interpreter

(This is heavily inspired and copy-pasted from [Go's proposal process](https://github.com/golang/proposal).)

## Introduction

The `go-interpreter` project's development process is design-driven.
Significant changes to the interpreter(s), libraries, or tools must be first
discussed, and sometimes formally documented, before they can be implemented.

This document describes the process for proposing, documenting, and
implementing changes to the `go-interpreter` project.

## The Proposal Process

### Goals

- Make sure that proposals get a proper, fair, timely, recorded evaluation with
  a clear answer.
- Make past proposals easy to find, to avoid duplicated effort.
- If a design doc is needed, make sure contributors know how to write a good one.

### Definitions

- A **proposal** is a suggestion filed as a GitHub issue, identified by having
  the Proposal label.
- A **design doc** is the expanded form of a proposal, written when the
  proposal needs more careful explanation and consideration.

### Scope

The proposal process should be used for any notable change or addition to the
interpreter(s), libraries and tools.
Since proposals begin (and will often end) with the filing of an issue, even
small changes can go through the proposal process if appropriate.
Deciding what is appropriate is matter of judgment we will refine through
experience.
If in doubt, file a proposal.

#### Compatibility

Programs written for Go version 1.x must continue to compile and work with
future versions of the Go interpreter(s).
The [Go 1 compatibility document](http://golang.org/doc/go1compat) describes
the promise the Go project has made to Go users for the future of Go 1.x.
The Go interpreter(s) should faithfully follow that heed.

#### Language changes

Go is a mature language and, as such, significant language changes are unlikely
to be accepted.
This should be a good news for the `go-interpreter` project as this means language
churn is unlikely to occur.
A "language change" in this context means a change to the
[Go language specification](https://golang.org/ref/spec).
(See the [release notes](https://golang.org/doc/devel/release.html) for
examples of recent language changes.)

### Process

At this point in time, we don't have much of a lengthy proposal process, 
but we may have to revise it and follow more closely the one articulated by the [Go project](https://github.com/golang/proposal).
For now, this should make do:

- [Create an issue](https://github.com/go-interpreter/proposal/issues/new) describing the proposal.

- Optionally create a design doc:
	- The design doc should be checked in to [the proposal repository](https://github.com/go-interpreter/proposal/) as `design/NNNN-shortname.md`,
	  where `NNNN` is the GitHub issue number and `shortname` is a short name
	  (a few dash-separated words at most).
	- The design doc should follow [the template](design/TEMPLATE.md).
	- Submit a GitHub Pull Request (PR) with the intended design document.
	- New design doc authors may be paired with a design doc "shepherd" to help work
	  on the doc.
	- For ease of review with GitHub, design documents should be wrapped around the
          80 column mark. [Each sentence should start on a new line](http://rhodesmill.org/brandon/2012/one-sentence-per-line/)
          so that comments can be made accurately and the diff kept shorter.
	- Comments on GitHub PRs should be restricted to grammar, spelling, or
          procedural errors related to the preparation of the proposal itself.
          All other comments should be addressed to the related original GitHub
          issue.

- Once comments and revisions on the design doc wind down, there is a final
  discussion about the proposal.
	- The goal of the final discussion is to reach agreement on the next step:
		(1) accept or (2) decline.
	- The discussion is expected to be resolved in a timely manner.

- The author (and/or other contributors) do the work as described by the
  "Implementation" section of the proposal.

## Help

If you need help with this process, please contact the `go-interpreter` contributors by posting
to the [go-interpreter mailing list](https://groups.google.com/group/go-interpreter) or on the `#go-interpreter`
channel on [Slack](https://gophers.slack.com/).
