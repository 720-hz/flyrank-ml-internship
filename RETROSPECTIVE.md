# Retrospective — Content Review Prioritization Capstone

*Written for the version of me who started this in Week 1. Draft — swap in
your own voice and specifics wherever this doesn't sound like you; the
technical facts are real, the "how it felt" parts are yours to correct.*

---

Eight weeks ago I picked a problem I could actually explain to someone
outside this field: FlyRank has more content pages than anyone has time to
review, and no good way to decide which ones to look at first. That was the
whole pitch. I didn't set out to build a "machine learning model" — I set
out to answer one question: if a reviewer can only check 20 pages a week,
which 20 should they be?

What I didn't expect was how much the *framing* of that question would
change everything downstream. I went in thinking this was a classification
problem — declining or not. It turned out to be a ranking problem, because
the real bottleneck was never detection, it was reviewer time. 54.2% of
pages in this dataset are trending down, and at 20 reviews a week that's a
658-week backlog just for today's declining pages. Once I saw the actual
constraint, the whole project reoriented around it.

The single biggest thing that changed my approach happened in Week 6, and
it's the part of this project I'm proudest of, not because the number went
up, but because it went down and I let it. An early random train/test split
made the model look excellent — precision@20 of 0.95. It was wrong. The
same clients were leaking across train and test, so the model was partly
just memorizing which clients tend to decline, not learning anything that
would generalize to a client it had never seen. Once I switched to a
client-grouped split — every client entirely in training or entirely in
test, never both — the honest number dropped to 0.70. That's a 25-point
drop I chose to keep, because it was true and the other number wasn't.
That's the moment this stopped feeling like a class exercise and started
feeling like real engineering.

If I kept building this, the next thing I'd add is a time-based holdout on
top of the client-grouped one — right now the label is backward-looking
(it says a page *was* declining as of this snapshot, not that it *will*
decline next month), and the honest next step is validating against future
data, not just unseen clients. I'd also want real revenue data instead of
the CPC-based proxy I used for "value at stake," and I'd want to try
folding the five-action recommendation queue into something closer to an
actual reviewer workflow tool, instead of a notebook that produces a table.

Three things from this that I'll carry into whatever I build next:

1. **Validation design beats model choice, every time.** The gap between my
   random-split number and my client-grouped number was bigger than the
   gap between my baseline and my Random Forest. I used to think picking
   the right model was the hard part. It isn't.
2. **Test each signal before you trust it, not after.** I tried staleness
   as a decline signal before CTR-vs-position and it came back mixed. Not
   every plausible-sounding feature earns a place in the pipeline —
   checking that early saved me from building on a shaky foundation.
3. **Using Claude well means arguing with it, not accepting the first
   answer.** The useful pattern wasn't "ask AI, get code." It was:
   research the options myself, bring multiple candidate approaches to the
   conversation, and only settle once I'd actually compared them — then
   verify the output myself afterward instead of trusting it by default.
   That's the difference between using AI as a shortcut and using it as a
   real collaborator.

To Week-1 me: you picked a boring-sounding problem over a flashy one, and
that was the right call. The interesting part was never the algorithm. It
was the discipline of not lying to yourself about your own numbers.
