---
layout: post
title: "Un-microservicing our storage: what nobody tells you about merging two databases back into one"
date: 2026-07-24
---

A while back, we made a decision that felt obviously correct at the time: split
our data layer into two separate databases, each with its own hosting
provider, each "owned" by a different part of the system. One held the core
application data — accounts, organizations, the day-to-day transactional
stuff. The other held a content-heavy subsystem — documents, templates,
generated artifacts, the kind of data that grows large and benefits from its
own storage characteristics.

It was a clean story on a whiteboard. Two bounded contexts, two databases, two
teams (eventually) who could move independently. Classic microservices
reasoning, applied one layer lower than services — to storage itself.

Then, over a couple of years, reality did what reality does. The "two
independent domains" turned out to reference each other constantly. The
application code became one codebase again long before anyone admitted the
database split wasn't paying for itself. And eventually we looked at our
infrastructure bill, our operational surface area, and our own onboarding docs
(which had grown a whole section just explaining "which database do I need for
this feature") and decided: it's time to put this back together.

This post is about how much harder that turned out to be than splitting it up
in the first place — and the specific ways it was hard, in case it saves
someone else some pain.

## The rehearsal-first rule

The first decision we made, and the one I'd defend most strongly in hindsight,
was: never let a merge script touch real data until it's been proven, over and
over, against a throwaway copy. Every real environment gets dumped, never
mutated in place until the very last, deliberate cutover step. Every script
that writes anything asks for an explicit typed confirmation before touching
anything above a disposable sandbox. It made the whole project slower to build
and much, much less scary to run.

That discipline is also what surfaced most of the problems below — because a
rehearsal against real (copied) data will show you things a rehearsal against
clean, synthetic data never will.

## Problem 1: your bookkeeping and your reality have quietly diverged

Every migration tool worth using keeps a ledger of "what's been applied here."
The theory is that this ledger is the source of truth for what a database
looks like. The practice, once two independently-evolved histories need to
become one, is that the ledger is only as trustworthy as the discipline that
produced it.

We hit this almost immediately: both of our old, independent migration
histories had, by sheer convention, named their very first migration the same
generic thing. Totally unrelated migrations, from two different systems, with
identical names. Any tooling that checks "has this already run?" by name alone
would confidently tell you the wrong thing. The fix wasn't complicated once we
saw it — compare the actual checksum of the migration contents, not the label
— but it's the kind of bug that only announces itself against a real, aged
database. A fresh test database will never coincidentally collide like that.

## Problem 2: the ledger lies on purpose, sometimes

The more uncomfortable discovery was that some of our real environments had
schema changes applied to them that never went through the normal, committed
pipeline at all. Someone, at some point, needed something fixed quickly and
ran a change directly against a live database. Entirely understandable —
everyone's done it under pressure — and also completely invisible to any tool
that trusts the migration ledger at face value.

Reconciling this required treating "what does the ledger say" and "what does
the database actually contain" as two separate questions that don't always
agree, and building a small, explicit, reviewed list of exactly which known
discrepancies were being reconciled and why — never a blanket "just trust
whatever's there." The alternative (silently overwriting or ignoring the
mismatch) is how you turn a one-time forgivable shortcut into permanent,
undetectable schema drift.

## Problem 3: the moment you can finally check your own promises — and don't like all the answers

Here's the thing about splitting a database in two: any relationship that
crosses the boundary can never be a real, enforced constraint. You can pretend
it's a foreign key in your application logic, but the database itself has no
way to guarantee it. For years.

The first time we could actually validate those relationships for real —
because for the first time ever, both sides of the relationship lived in the
same physical database — we found genuine orphaned records. Rows quietly
pointing at things that no longer existed. Not corruption, not a bug we
introduced: just years of accumulated reality that nothing had ever been able
to check.

The instinct here is to "fix" it — delete the broken rows, force the
constraint to validate, declare victory. We deliberately didn't do that as a
first move. A couple of those relationships were already designed, on
purpose, to tolerate a missing reference — the application never assumed the
other side would always exist. So rather than destroying real (if imperfect)
data to satisfy a constraint that was stricter than the system had ever
actually promised, we left those specific cases explicitly documented as
known, accepted exceptions, and made the tooling itself keep that record up to
date automatically rather than relying on someone to remember to update a doc
by hand. The alternative — silently deleting real rows because a new,
stricter check didn't like them — felt like solving the wrong problem.

## Problem 4: not everything in your database is actually yours

Partway through, we discovered a whole family of tables that belonged to a
third-party library we depend on for some background AI/agent tooling —
completely outside our own data model, managed entirely by that library at
runtime. Because of an inconsistent configuration setting between two
different services using that library, some of those tables were landing in
the wrong logical location relative to our own schema boundaries.

The tempting "fix" would have been to sweep them along with everything else
during the merge — rename them, move them, treat them like our own data. That
would have been wrong twice over: it's not our schema to change, and doing so
could have broken the library's own internal assumptions. The right move was
the boring one: explicitly detect and exclude anything belonging to that
subsystem, everywhere — in the data-movement logic, in the drift checks, in
the row-count validation — and leave it completely untouched, regardless of
which physical location it happened to land in.

## Problem 5: the unglamorous 90%

None of the above is the part that actually ate the most time. That honor goes
to: credential profiles not matching between tools that were supposed to
behave identically, an environment-loading order bug that let a stale
connection string quietly override an explicitly-set one, and a TLS/SSL
parameter that one client library understood and a different, lower-level
client tool flatly rejected. Every one of these was a real bug, confirmed
against a real failure, fixed with a small, specific change — and every one of
them cost more wall-clock time than the "interesting" architectural problems
above.

If there's a lesson in that, it's this: budget for the boring stuff. The hard
architectural problems are the ones you can plan for in a design doc. The
environment variable that gets silently clobbered three layers deep in your
own tooling is the one that actually eats your week.

## Problem 6: knowing when to stop being rigorous

Late in the project we built a whole parallel path to let developers rehearse
the exact same merge locally, against their own local database copies —
reusing all the same reconciliation logic as the real thing, for parity and
confidence.

Then we deleted it. Local development data isn't real. Nobody's local
database has years of accumulated drift worth reconciling — it's disposable,
seeded, throwaway data by design. Building a faithful local rehearsal of a
process meant to safely handle irreplaceable production history was solving a
problem nobody actually had. The right answer for local development was
almost insultingly simple by comparison: point at one database instead of
two, and rebuild it from scratch. Recognizing that the rigor appropriate for
real, irreplaceable data was actively wasted effort somewhere else was its own
small lesson.

## The actual takeaway

Splitting a database is a decision you can make in an afternoon, with a
diagram and good intentions. Undoing that decision means reconciling
everything that happened in the years between the diagram and today: every
shortcut anyone ever took under pressure, every relationship the system
quietly tolerated instead of enforcing, every piece of someone else's data
that ended up living inside your walls by accident.

None of that shows up in the "microservices vs. monolith" debate as it's
usually framed. But if you're ever the one holding the "let's put it back
together" ticket: budget real time for archaeology, not just architecture.
The database doesn't remember why it is the way it is. You have to go find
out.
