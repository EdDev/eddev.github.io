---
title: "GitHub Review and Merge"
date: 2020-07-20
categories:
  - blog
tags:
  - git
  - github
  - review
  - merge
---

# Overview
Landing on [GitHub](https://github.com) after several years with
[Gerrit](https://www.gerritcodereview.com) is challenging.

To be precise, performing reviews on GitHub compared to Gerrit is a pain.

Software projects and teams look to collaborate by engaging a development flow
which includes proposing code changes, reviewing them and finally accepting
them into the code base.

The first two steps are executed in a loop until the author and the reviewers
are reaching an agreement on the correctness and quality of the change.
Immediately after, comes the merge where the changes are included in the
project code base.

> **_NOTE:_** Git is the SCM tool and GitHub is the source code hosting that
includes view and review services.

> **_NOTE:_** Testing will not be discussed in this post (which is no less
important).

# The Need
The development flow of coding, reviewing and merging should be supported
by the tooling of a project to ease and help its implementation.

This post is looking to discuss the last two steps:
- Reviews are essential to a healthy codebase, providing correctness, quality
  and a learning opportunity with focus on team collaboration.
- Merging combines work into the collaborative project, providing a sync point
  and keeping the work history.

> **_NOTE:_** Codebase change history is crucial for maintaining and
understanding the code. It provides context to changes with proper reasoning
and relevant data.

# Reviewable Change
Write code with your reviewers in mind.

- Small is greater than Big:
  Small changes are easy to review and simple to explain.
  - Keep PR/s small:
    Discussions are performed on PR level in GitHub, therefore, less commits
    in a PR encourages focused discussions.
    This obviously comes with a cost: Increased test runs and stacking PR/s
    one over the other in a dependency chain.
  - Keep commits small:
    Small commit messages are focused on a specific intent (described in the
    commit message), giving the reviewer a smaller domain to focus on. It also
    provides an insight of the change direction, telling a story.
- Commit messages provide intent and context to changes for the reviewer.
- Changes performed due to review requests, should be easily tracked by the
  reviewer (this is one of the main pain points with GitHub review flow).
  This issue is elaborated in the
  [Track Requested Changes](#track-requested-changes)

# Merge Methods
The method chosen to merge changes into the collaborative branch (e.g. master)
impacts mainly the change history.

GitHub (through git) provides 3 main
[methods](https://docs.github.com/en/github/administering-a-repository/about-merge-methods-on-github)
of merging a PR:
- Merge
- Squash
- Rebase (my favorite)

It is worth mentioning that in GitHub, a PR represents a collection of changes
that are merged together. Individual commits cannot be merged (unless they
are wrapped by a PR, in a 1:1 relation).

The merge and rebase methods are preserving the PR commit history intact.
The squash method on the other hand is combining all PR commits into a single
commit (recording all commit messages into a single commit message)
automatically.

The merge method preserves the feature branch, allowing future usage of the
changes included in it (e.g. backporting or sharing with multiple branches).
On the other hand, the rebase method places the changes one by one on the
target branch, keeping a cleaner linear log history but without the changes
original grouping.

If you value your change history, do not use squash.

If you intend to merge changes into multiple branches frequently or your
individual commits are not self-contained (i.e. they cannot stand alone without
the other changes in the PR when tested), use the merge method.

If you want a linear clean change history and your commits are stand alone,
use the rebase method.

> **_NOTE:_** On GitHub, a PR contains multiple commits but checks (CI) occur
against the last commit (and not per commit). Therefore, a rebase has a risk
of containing unchecked commits (which may affect their correctness).

# Track Requested Changes 
The GitHub review tool has one major disadvantage, especially when compared to
Gerrit. It is extremely hard to both track changes that respond to review
comments and keep a proper change history in the project.

The difficulty to track changes in a PR is mainly due to the fact that PR/s
may include more than one logical change. While helping with reducing tests
(run once on the top commit instead on each and every one of the commits), it
does not provide proper means for a reviewer to track his requested changes.

To both support the review process (tracking the changes) and keeping the
change history in good shape, we can consider the following points:

- Answer each review comment with "Done" or other text, helping the reviewer
  see that the comment has been addressed.
- One PR, One commit: Whenever possible, keep one logical commit per PR.
  - One could use both transient commits to answer review requests or use
    the pushed changes diffs on the PR discussion page.
  - When using commits to answer review requests, the commits need to be
    squashed by the author before a final approval for merge is given.
- One PR, multiple commits:
  - If the PR is simple and small enough, using the provided diffs between
    pushes may be enough for a proper review.
  - When the PR is not small or when there are multiple requested changes,
    temporary commits may be used, ordered under the logical commit they
    belong to. It is up to the author to squash them at the end before
    the final approval. 
- When using forced pushes to answer review requests, consider leaving
  a comment about the changes included in the push.
- Do not mix your own changes with a rebase, push each separately and consider
  the previous point.
- The reviewer can confirm no changes have occurred from his last review by
  checking the last push diff. This is useful post a squash operation.

