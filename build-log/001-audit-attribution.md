# The column that was never wired

*Build log 001. An audit trail that could not name who wrote anything, for four months, and a first diagnosis that was wrong in a way worth writing down.*

---

Every write names its author. It is the least interesting principle in this system, because it sounds too obvious to need stating. It is also the one the whole governance model rests on: if a model proposes, a human decides, and a deterministic engine applies, then the record has to be able to tell you which of those three did what. A trail that cannot distinguish the assistant from the person is not an audit trail. It is a log.

It was not true for four months. Every audit row the current release line had produced was unattributed, including the per-field plan entries that had been a phase headline.

What follows is how I found that, the diagnosis I wrote down confidently before I had measured properly, and why being wrong in that particular direction changed what the repair had to be.

## The measurement that started it

The original report came out of a review of the write path. It said the authoriser was reaching the `actor` column but not `user_id`, and quantified it as zero of ninety-two rows since the seventh of September. That reads like a wire that broke recently.

So I measured the whole table rather than the recent slice.

| Month | Audit rows | Rows carrying `user_id` |
|---|---|---|
| March | 104 | 104 |
| April | 477 | 477 |
| May | 13 | 13 |
| June | 53 | 53 |
| July | 127 | 126 |
| August | 0 | 0 |
| September | 853 | 0 |

That is about as clean a break as a time series ever gives you. A hundred percent attribution for five months, then nothing. The last row carrying a `user_id` was the seventh of July. The production database was created three weeks later, on the twenty-seventh.

## What I concluded, in writing

That the column had been populated correctly, that the schema and the historic rows had come across during the migration to the cloud, and that the new write path on the other side had never been wired up. A gap introduced at the cutover. Not a mystery, I said. A bisect target.

I recorded it as an owner ruling, with the table above as the evidence, and moved on.

## Why that was wrong

`user_id` has never had a writer. Not since the cutover. Not before it. There is nothing to bisect and nothing to revert to, because there is no commit in the history of this system where that column was ever written by application code.

The proper measurement, done at the start of the next phase, showed three things.

All 773 rows carrying a `user_id` predate the eighth of July, and not one of them carries a backfill marker. They are exactly the legacy rows copied in by the audit table migration, whose backfill INSERT includes the column. They were never written by the running system. They were written by a migration script, once.

All 919 rows written since the end of July carry none.

And `user_id` is absent from the INSERT column list of both writers, at every commit in their history. The audit logger goes back to early May. The canonical write path never had it either. I did not need to bisect anything. I needed to read one INSERT statement.

## The artefact that made a wrong answer look obvious

The apparent boundary on the seventh of July is not a boundary. It is the near edge of a twenty-four day period in which the system produced no audit rows at all, because I was not using it. The production database happened to be created inside that silence.

So the date looked like a cutover because the system was idle across it. Absence of rows and a change in behaviour render identically in a monthly aggregate. The table above cannot distinguish "it stopped working here" from "nothing happened here", and I read the first because I already had the second explanation in mind: I had migrated to the cloud recently, so a cutover regression was the story I was primed for.

The column had been declared, indexed, given row-level security policies and backfilled with real history. Everything except the one thing that would have made it work. That is exactly the shape of defect that survives review, because every artefact around it looks correct.

## Why the difference mattered

If it were a regression, the repair is a revert. Find the commit, restore the wire, backfill the gap from whatever the old path recorded, done in an afternoon, no design decisions required.

Because it had never worked, the repair is a first implementation, and a first implementation has to answer a question a revert would never have raised: **what value does `user_id` take on an agent-authored write?**

That question is not incidental here. The release that was about to ship introduces a second class of writer. Until that point every write originated with a person, so the question could stay unasked. The moment an assistant can propose a change that a person approves, amends, or discards, the answer has to distinguish at least three cases: the person authored this directly, the assistant proposed it and the person approved it as proposed, the assistant proposed it and the person corrected it before approving. In the third case the authorship moved. The person who fixed the date is the author of that date, not the model that suggested the wrong one.

A revert would have shipped the wrong answer to that question silently, by restoring whatever behaviour existed before, and nothing existed before.

It also invalidated a constitutional argument. One clause justified skipping a change record for human-initiated writes on the grounds that the user is the authoriser and is therefore already recorded. Eight hundred and fifty-three rows say otherwise. The clause was reasoning from a property the data did not have.

## What I should have done

Two checks, neither of which takes longer than a minute.

Before concluding that something broke on a given date, confirm the series is continuous across that date. A gap in a monthly aggregate is invisible, and a gap adjacent to a real infrastructure event is close to irresistible as a story. Plot the rows by day, not by month, and the twenty-four days of silence are the first thing you see.

Before looking for the commit that removed a column from a write, check whether any commit ever added it. `git log -S user_id` against the two writer files would have returned nothing and ended the investigation in the first minute, rather than after a table, a ruling and a wrong paragraph.

The general version, which I keep relearning: a confident causal story built on aggregated data should be treated as a hypothesis about the raw data, and the cheapest test is usually to go and look at the raw data. I did not do that because the aggregate agreed with something I already believed.

## What it cost

Four months of audit rows that cannot tell you who made the change. They are not recoverable. Preserved audit history is preserved, which is the entire point of it, so a wrong record stays wrong and the honest repair is to write the correction next to it rather than to quietly clean it up. The completion report that claimed the gap had been closed keeps its original text with a dated note attached saying it was wrong.

And a rule that now sits in the test suite rather than in a document: a column that exists is not a column that is written. Schema presence, index presence and policy presence are all satisfied by a column nobody ever inserts into. If a field is load-bearing for governance, there is a test that writes through the real path and asserts the field arrives, and it fails loudly when it does not.

---

*Part of the [Tawkshop](../README.md) build log. The implementation is private; the reasoning is not.*
