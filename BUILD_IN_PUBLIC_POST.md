# Build-in-public post (draft)

*Draft for LinkedIn / wherever you post this. Trim, add a screenshot of the
results table or the ranked queue, and adjust the voice to match how you
actually talk.*

---

**I built a model to tell a content team which pages to review first — and
the most important decision I made had nothing to do with the model.**

The problem: FlyRank has way more content pages than review time. 54% of
pages in the dataset are trending down, and at a realistic pace of 20
reviews a week, that's a 650+ week backlog just for today's declining
pages. So the real question isn't "is this page declining" — it's "which
pages should a reviewer look at first." That's a ranking problem, not a
classification one, and framing it that way changed everything downstream.

**The decision I'm most proud of:** early on, a random train/test split
made my model look great — precision@20 of 0.95 on the top-ranked pages.
It was a lie. The same clients showed up in both training and test, so the
model was partly just memorizing *which clients* tend to decline, not
learning anything that would hold up on a client it had never seen. I
switched to a client-grouped split — every client entirely in train or
entirely in test — and the honest number dropped to 0.70. I kept that
number instead of the flattering one, because it's the one that actually
means something.

**The limitation I want to be upfront about:** the label this model
predicts is backward-looking. It tells you a page *was already* declining
as of this data snapshot — it doesn't forecast what happens next month.
This is decision support for today's backlog, not a crystal ball, and I've
built the whole project around not overselling that distinction.

With an honest, client-grouped evaluation, the model still beats a
transparent baseline rule — ROC-AUC goes from 0.596 to 0.708, average
precision from 0.476 to 0.550 — but the real story here isn't the numbers.
It's that the validation method mattered more than which model I picked.

Full writeup, the eval results, and the limitations list (nothing hidden):
[CAPSTONE_README.md](https://github.com/720-hz/flyrank-ml-internship/blob/main/CAPSTONE_README.md)
Live paper: https://720-hz.github.io/flyrank-ml-internship/
Demo video: https://youtu.be/PRKaZWDSsMA
