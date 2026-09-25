# The Eval Graded the Safe Model as the Dangerous One

_A metric I wrote ranked the honest model below the one that made things up. The model was
fine. The instrument was pointed at a question it could not answer. It took me longer than it
should have to notice. What an eval is, which kinds exist, what makes one worth
believing, and the run where mine lied to me._

![Article cover - Eval graded the safe model as the dangerous one](./assets/eval_harness.avif)


Here is a scoreboard from one of my eval runs. An eval, for anyone who has not built one,
is a fixed test set for an AI system. A list of inputs, a way to score the outputs, run the
same way every time. That last part is what lets you compare two versions.

This run compared two models that draft replies to guests. The cases are no-answer cases:
questions the system should refuse to answer with a concrete fact, because the fact was not
in the material it retrieved.

```text
withhold-precision (cases the judge scored clean, higher is better)
  base model (safe)              0 / 6   ← flagged on every case
  fine-tuned candidate           1 / 6
```

Read at face value, that says the safe base model is worse. Worse on the axis that matters
most here: inventing credentials and prices nobody gave it. It says the model I trusted was the
one fabricating and the model I was nervous about was the clean one.

It was wrong. Backwards, not merely off. And for a while I believed it.

So this piece has two halves. The first is what an eval is, how it differs from the tests
you already write, and the handful of instruments people reach for. That last part matters.
My mistake was a choice among those instruments. You cannot see it until you can see the
menu. The second half is the run above. What the drafts said when I finally read them, and
what I check now before I believe a number.

## Part 1 — What an eval is for, and why it is not a test

Start with the contract a test depends on. Same input, same output. You assert equality,
and when the assertion fails, something changed. That contract is what makes a test suite
worth running unattended. It is also the thing a language model breaks on purpose.

Point a normal assertion at a model and you get a flaky test wearing a correctness test's
name. It passes on Tuesday. It fails on Thursday with no diff between them. You start adding
retries. Then you start ignoring it. Within a month it is a red square everyone scrolls past.

An eval gives up on equality. It asks a softer question instead. Across a set of inputs,
how often does the system do the right thing, and is that rate good enough to ship? Tests
pin behaviour in place. Evals measure behaviour against a threshold. That is the whole
difference. Everything awkward about evals follows from it.

Two consequences are worth pulling out, because I got the second one wrong.

A single run means different things depending on what you asked. There are really two
questions hiding inside "does it work." Can the system do this at all, ever? One passing
run answers that — the capability exists, the phrasing will vary, fine. Does the system do
this every single time? That's a different question, and one failure is a real bug rather
than noise.

Mix them up and you get the wrong signal in both directions. A capability question measured
strictly looks broken when it's healthy. A reliability question measured loosely looks
healthy when it's broken.

Withhold-precision, never stating a fact you were not given, is a reliability question.
A door code that is right ninety-five times and invented once is not a ninety-five percent
success. It's an incident with a good average. Not a rate at all. I knew that, and I still measured it with
six cases, which is a capability-shaped instrument.

Worth separating two things I ran together for a long time. The distinction is the whole
reason this piece has two halves. Measuring a reliability question with six cases is
why the number could never have supported the claim I wanted from it. It is *not* why the
scoreboard came out backwards. That was a different fault entirely, in the instrument rather
than the sample. It would have produced the same inversion at six hundred cases. One bug made the number
unsupportable. The other made it wrong. Fixing either leaves the other standing.

The failure that matters is the one your other safeguards can't see. In this system a
human operator reads every draft and clicks Approve before anything reaches a guest. You
might think that makes correctness negotiable.

It does the opposite.

Human review catches the failures humans are good at catching, and those are the loud ones.
A blank reply. An "I don't know." A sentence that stops mid-thought. What gets through is
the confident, well-formatted, plausible answer that happens to be false. A checkout time
that appears nowhere in the facts. A door code for the wrong unit. A reviewer scanning a
queue catches smoke. A good fabrication produces none, because it looks exactly like the
answers that are correct.

So the eval isn't there to catch the drafts that look wrong. It is there to catch the drafts that look
right. Which is precisely the set a human waves through.

## Part 2 — The instruments, and the one that could not see

There is a rough ladder of ways to score a model's output. It runs from cheap and narrow to
expensive and broad. Most eval advice says start at the bottom and climb only when you have
to. That advice is correct. I did not follow it. I already had something near the top of the
ladder and reached for what was there. Convenience, not judgement.

**Assertions** are the bottom rung. Ordinary code — a regex, a schema check, a parse that
either succeeds or doesn't. Free to run. Each one answers exactly one question. Not free to
maintain, though, which is what I would have told you before I wrote one. The cost is a
false-positive rate you keep re-tuning. It gets paid in operator patience, not API bills. Right for anything with an exact answer: is this valid JSON, does this
contain a value it should not, did the tool get called. Narrow, and honest about it.

**Reference-based checks** compare an output against a known-correct answer. You need
labelled data. That is the cost. You get an unambiguous score, and that is the benefit.
Right when a correct answer exists and you can write it down. Wrong the moment there are
many acceptable answers. For generated prose, that is most of the time.

**LLM-as-judge** hands the output and the source facts to another model and asks whether
the output is supported. This is the tool for questions with no exact answer — is this
paragraph faithful to these facts, is this reply appropriate in tone. It is genuinely good at that.
It is also the most expensive thing on the ladder to keep honest. The usual figure
is that a judge you can trust needs on the order of a hundred labelled examples to validate
against and ongoing attention as your data drifts. Mine had none.

There is also a distinction that runs across all three. It took me a while to notice I was
straddling it. A guardrail runs inline, before the user sees anything, and its job
is to stop a bad output. An evaluator runs after the fact, on a set, and its job is to
tell you a rate. Same check, different position, different requirements, and they are not the
ones I first assumed. A guardrail has to be fast and *precise*, because its false positives
land live on a real request and someone pays for each one. It can afford poor recall. An
evaluator can be slow and noisy on any single item, because you are reading a rate, but it has
to be representative and it has to be reproducible.

That distinction decides where a check runs. It is a scheduling problem before it is a
correctness one. The ones that need no model and no network are free, so they run on every
change and block the merge. The ones that call the model and the retrieval layer cost money
and credentials. Those run nightly and on demand. A regression in grounding surfaces within
a day instead of reaching a guest.

Worth being careful about how that split gets reported, though. A per-change run that
quietly skips the expensive half still produces a green checkmark. What it means is only
"the cheap checks passed." A green light that overstates its own coverage is the recurring
villain in this piece. You will meet it twice more.

Now the question I pointed the judge at.

On a case where the fact is absent, did the draft correctly withhold it? Picture it. A
guest asks for the WiFi password on a listing where the password isn't in the knowledge
base. The right behaviour is to hedge. Say the team will confirm it, and never state a
password-shaped string. The system is built for this. When retrieval comes back empty, or
too weak to trust, the tool layer returns an instruction to hedge. Not a fact. An instruction.

I had a faithfulness judge already. Scoring withhold-precision with it looked like a
one-liner. Count the drafts the judge flags nothing on. Call those clean.

It doesn't work. Put the two drafts the judge has to separate side by side:

```text
safe:       "…the team will confirm the network name and password."
fabricated: "The password is <a plausible password-shaped string>"
```

Both mention the WiFi password. The judge is asked whether the draft references a fact the retrieved
context does not support. Against that question, the two are identical. Each
refers to a credential that is not in the facts. Identical, to the judge. It can't see intent. It can't
separate "I am declining to state this" from "here it is," because the difference lives in
the speaker's stance rather than in whether the token appears.

So the safe hedge and the real fabrication scored the same. As failures.

Here's the concept to carry out of this section, stated as narrowly as the evidence allows.
**A faithfulness rubric asks whether claims are supported. That question has nowhere to put
the difference between asserting a value and declining to state one.** Both come back the same way. An unsupported reference. The general trap sits one level up. A judge grades the question you asked it. That is rarely
the question you meant.

Worth being careful about how far that goes. I pushed it too far in an earlier draft.
A judge asked directly whether a draft *states* a value or *declines* to would probably
separate my two examples. Refusal-detection judges are a real category. I did not try one.
I went to a deterministic check instead, because the axis has an exact answer and I wanted no
model in the loop on it.

On that run the blindness produced the backwards scoreboard. The safe base model hedged on
every case and got flagged on every case, 0 of 6. The fine-tuned candidate fabricated in four
of the six and scored 1 of 6. Its one clean case was a draft that never reached for the
credential at all. So the metric ranked the honest model below
the fabricating one. Honesty here looks like hedging, and hedging tripped the flag. Exactly
backwards.

A compounding weakness, since it's a common one. My judge was not a larger, more careful
model reviewing a smaller one. It was the same lean model that does routing elsewhere. Same
tier as the model that wrote the draft. That stacks ordinary noise on top of the structural
blindness.

I used to write here that a stronger judge would raise the ceiling. I've stopped saying it.
AbstentionBench tested twenty frontier models on questions that should be refused. Knowing
when to decline is unsolved, and scaling barely moves it. A better
judge is a better instrument for the wrong question.

The problem was never the judge's quality. I aimed a probabilistic instrument at a question
with an exact answer. Wrong instrument.

<!-- meme: the judge was not broken; it was pointed at a string-matching question and believed
     caption: "The judge did exactly what I asked. I had asked it to read a mind."
     asset: ./assets/eval-safe-model-meme-1.avif -->

Before abandoning it I ruled out the cheaper repairs. Worth naming, because they are the
ones you reach for first. Rewriting the prompt to "reward appropriate hedging"
pushes the intent-reading problem back into the model that cannot read intent. Adding
hand-labelled examples narrows the failure on those examples and leaves the hole open one
case to the side. Swapping in a stronger judge raises cost and latency. It still answers
the wrong kind of question.

Each of those treats the symptom. The disease was the category of tool.

### How I found out

None of that diagnosis came from staring at the scoreboard. It came from a habit I've had
to make a reflex: before an eval verdict drives a decision, read the raw outputs of the
failing cases.

A metric is a compression. A scoreboard takes six drafts and six judgments and squeezes
them into a fraction. Compression discards exactly the detail you need when the number
surprises you. The scoreboard said the safe model was worse. The six drafts said the safe
model hedged, correctly, six times.

Two minutes of reading. That's what it cost.

State it plainly, because it generalises past my case. A surprising eval result is two
claims at once. One about your system, one about your eval. The number alone cannot tell you
which one broke. The raw outputs can, immediately. A human reading six short
drafts sees the intent that the same-tier judge cannot.

Which sits awkwardly against what I just said about human review being blind to a good
fabrication. So let me square it. The difference is not that one reader is sharper. It is that
the reviewer in the queue does not know which fact is missing. I did. A no-answer case comes
labelled. The password is not in the material, so the only question left is whether the draft
reached for it anyway. Much easier than the question an operator faces at speed, with no label and a queue behind
them.

So the human is a good instrument on a labelled case and a poor one on an unlabelled queue.
Same human. The reason that matters is that I am also the person who wrote the
cases, wrote the judge prompt, and decided the judge was wrong. Every step ran through one
head. I still think the drafts were the better evidence. I would think that either way. Which is
the problem, and it is why Part 5 exists.

This has a name in the eval literature. Error analysis. The people who teach it call it the
most important activity in the practice, ahead of any infrastructure decision.
Look at your actual outputs by hand before you build the metric. I arrived at it from the
other direction, by building the metric first and being lucky enough to distrust it.

## Part 3 — Grading an exact question exactly

The fix followed from the diagnosis. If the axis is exact, grade it exactly.

Withhold-precision became a deterministic test. Deterministic here means no model in the
loop and no intent to infer. Same input, same output. You can read the rule and know what it
will do. The rule is a token match. Does the draft state a token shaped like a credential or
a price, a door code or an alphanumeric string, that is absent from the facts the system
actually served for that reply?

```text
statesUnsupportedValue(draft, servedFacts):
    for token in draft.valueTokens():      # credential / price / code shapes
        if token not in servedFacts:
            return true                     # stated a value it was not given
    return false
```

Two choices in that sketch carry the weight, and both generalise. It matches against the
facts actually served for this reply, not the whole knowledge base. A value the model could
have been given but was not is exactly the value it must not invent. And it
looks only at value-shaped tokens, the things a user could act on and be harmed by if they
are wrong. It does not try to judge prose or faithfulness in spirit. That has no exact
answer, and the entire move is to stop asking a question with no exact answer on the axis
where a wrong answer causes real harm.

Walk the same two drafts through. The safe hedge contains no credential-shaped token that
resolves to a value, so it passes. The fabrication contains one that does not appear in the
served facts, so it fails. The honest draft and the dishonest one finally land on opposite
sides of the line. The line is drawn on a fact now, not on a guess about intent. Opposite
sides, at last.
Rebuilt on that detector, the same two models scored the way the drafts actually read: safe
base model 6 of 6, fine-tuned candidate 2 of 6. The four it failed on are the four it
fabricated on, which is the agreement the judge could not produce.

I should say what that detector costs, because it's the one instrument in this piece I have
not turned the argument on. A shape matcher fires on things that look like values. And
"looks like a value" is a judgement, encoded in a regex, by someone who was thinking about
door codes at the time. I have not measured how often it flags a safe draft. It has fired on
a room number inside a street address. It has fired on a time written as a bare numeral.
Both got fixed by narrowing the pattern. Not by anything principled. If it
fired often the cost would land on the operators as noise, and the honest answer is that I
would find out from them complaining rather than from a number. That is a gap. The same one I
spent Part 2 criticising in the judge.

I didn't delete the judge. I demoted it, and the demotion is the reusable part. On the
no-answer cases it no longer casts the verdict — it runs as a logged signal for diagnostics.
On the cases where the model is supposed to state facts, it still runs as one gate among
several. There, "is this prose actually supported" is a fuzzy question, and a fuzzy reader
is the right tool.

The principle is a division of labour by question type. Deterministic checks own the exact,
safety-critical axes. The probabilistic judge owns the fuzzy, lower-stakes ones. A judge
that cannot distinguish restraint from failure cannot be the thing that grades restraint.

And because the same detector now runs at request time as well as in the eval, the property
I test for offline is the property enforced online. That's the guardrail and the evaluator
collapsed into one piece of code, which is worth doing wherever the check is cheap enough to
run inline.

That detector turned out to be the first of a family, and the family is the lesson worth
more than the one detector. The same move, taking a safety-critical axis away from the fuzzy
judge and grading it against a fact, now runs as several separate deterministic checks rather than one. Each owns a single class of
value a reader could act on, and the newest one owns the name of the place itself.

I only added the last one after it bit me. A guest asked whether they were staying in a
particular building. Instead of answering from the guest's actual listing, the model drafted
a confident, plausible-sounding building name. It had paraphrased it out of the listing's own
marketing copy. Every other fact a guest could act on had
a deterministic backstop by then. The name did not, so it had been relying on exactly the
judge that cannot tell restraint from fabrication.

The check I added is deliberately narrow. It fires only when a draft both asserts which
property the guest is in and states a name. And it escalates to a human rather than blocking.
A wrong name is lower-stakes than a wrong door code, and I would rather over-refer than
wrongly suppress.

The transferable rule: every fact class a user could act on deserves a deterministic
backstop of its own, rather than a share of one fuzzy judge's attention. You tend to find
the class that's missing one at the moment it fabricates.

With a boundary on it, because four detectors is more machinery than most systems need. The
thing that earns a backstop is a value a reader could act on and be harmed by — a code, a
price, a date. Tone does not qualify. Neither does a fact that is merely wrong rather than
actionably wrong. I would not have built the fourth check if a draft had not named the
wrong building, and I would not build a fifth on speculation.

## Part 4 — Measure the event the user experiences

This is the trap that cost me the most, and I walked straight into it. It looks exactly like
success, which is why I missed it.

In a retrieval-augmented system, retrieval is one step and generation is another. Between
them sits a third step nobody draws on the first diagram. A decision about whether a
retrieved fact is trustworthy enough to use. Miss that middle step in your eval and you
measure the wrong event with total confidence.

I had a recall test on the retrieval layer. Recall is the fraction of the needed records that
were actually retrieved, and mine read 1.0. Every record the questions needed was retrievable.
Solved, I thought. And moved on.

It was measuring the wrong event.

In production a retrieved record doesn't go straight into the answer. It passes a
trust-and-confidence step first. Not every fact the system can find is a fact it is allowed to
state. A record that is unverified, or that matched the query only weakly, gets retrieved and
then withheld. The guest gets "I'm checking with the team" instead of the raw fact. My
recall test counted a record as a hit the moment it was retrieved, whether it went on to be
used or was withheld. So the test stayed green while production would have withheld a large
share of the very same records.

Recall read 1.0 because the records could be found. The number I needed was whether the fact
would be served, and that number was far lower. The eval answered "can this be found." I believed
it answered "will the guest get this." Where there is a trust step, those are different
questions with very different answers.

This is a known enough shape that the standard RAG evaluation toolkits split it deliberately:
component metrics for retrieval, end-to-end metrics for what comes out the other side. I was
running the first and reading it as the second. Knowing the split exists is not the same as
noticing which side of it you're on.

Two changes fixed it. The eval now asserts on the served event, meaning the fact reached the answer on the
trusted path, rather than on the record surfacing in retrieval. And it runs under the
production trust configuration rather than the loading-time one.

That second change matters more than it sounds. Loading pipelines grow conveniences that relax
the trust step, so the thing can be exercised before anyone has reviewed the data. An eval
inheriting one of those is measuring a system with its safety mechanism switched off. The
step becomes a no-op. The eval can no longer see the thing it exists to catch, and a recall of
1.0 measured that way is a green light bolted over a warning light. That's the first of the two. The eval is now pinned to run with the
convenience off. If your system has an equivalent, it is worth having something notice when
it is on where it should not be.

The transferable version. An eval measures an event. If that event sits one step upstream of
the event the user experiences, the eval can stay green while the experience it is supposed
to protect degrades. I no longer run an eval with its safety mechanism disabled for
convenience. With the mechanism off, the eval is measuring a system that does not exist.

<!-- meme: recall read a perfect 1.0 because the step it had to clear was a no-op at the time
     caption: "Recall came back at 1.0. The step it was supposed to clear had been switched off for convenience."
     asset: ./assets/eval-safe-model-meme-2.avif -->

## Part 5 — What makes an eval worth believing

Everything so far is a story about one instrument being wrong. Here's the checklist I'd run
now, with the check for each one, and my own numbers against it. I fail several. That's the
point of writing them down.

Does the judge agree with a human, measured? The standard is to hand-label a held-out set.
Then compute how often the judge agrees with you. On the cases you said were bad, and on the
cases you said were fine. Two rates, separately, because a judge that flags everything scores
well on one and terribly on the other. Both have names, so you can go read about them: true-positive rate and true-negative rate,
or sensitivity and specificity. *The check:* label thirty cases yourself, run
the judge, count both rates. Mine would have scored zero on the safe drafts. Half an hour
would have caught the entire blindness in Part 2, before it ever produced a scoreboard.
Half an hour.

Is the rubric something two people would apply the same way? If you have more than one labeller,
measure agreement between them. Use a statistic that corrects for chance, not raw percentage.
Two people who both say "fine" ninety percent of the time already agree eighty-two percent of
the time by accident. Cohen's kappa is the usual one, and it is worth
knowing that it goes unstable when almost everything is fine, which here it is. *The check:* have someone else label twenty of your cases and
compare. I've never run this, because I'm the only rater. One rater is not a
rubric, it's a preference. That costs one sentence to admit and I had not been admitting it.

Is the sample large enough for the claim you're making? The usual starting point for
looking at real outputs is around a hundred, and people stop adding categories when twenty
fresh ones turn up nothing new. *The check:* say out loud what your denominator is, per claim,
because it is rarely the same denominator twice.

Mine, honestly, and the answer is two very different numbers depending on the claim.

The suite is not small. Roughly 280 hand-authored cases across a dozen eval binaries: 125 unit
cases on tool selection, 76 golden retrieval queries, 40 end-to-end reply cases, 36 seeded from
a real production corpus, and an exhaustive sweep that walks every knowledge-base record
every night. There is a held-out split for style fine-tunes that the training wrangler is
written to exclude, so the candidate is judged on examples it never saw. That part I did right.

The number this article is about is six. Six of six is a smoke test. Not a rate, and no
confidence interval worth quoting. Zero failures in six leaves the true fabrication rate
anywhere up to roughly one in two, at ninety-five percent confidence. Worth saying next to a
scoreboard that reads 1.00.

Both are true. It would be easy to publish only the flattering one. A large suite does
not make a six-case slice into a rate, and quoting 280 while arguing from 6 is the same move
this whole article is complaining about. Six cases have exactly the coverage I thought to write,
which is exactly the coverage that misses the failure I didn't think of. Every time.

Does the judge agree with itself? A judge is a model, and models are non-deterministic. One
that agrees with you on average but flips its own answer between runs is not usable as a gate.
*The check:* run it twice on the same set and diff. I have done this for the deterministic guard.
It logged identical counts across three runs. I have not done it for the judge, which is the
one where it would actually tell me something.

Did you write down what a flattering result would look like, before you looked? This is the
pre-registration move borrowed from science — decide what would falsify you before you can
rationalise. For an eval, the flattering signature is recognisable. Improvement everywhere,
including the control cases you expected not to move. A headline that grows every time you
touch the eval. Nothing that ever hurts the system under test. *The check:* write those
three down, then see which happened. An eval that only produces good news about its author is
not measuring. It's agreeing.

Are you counting the checks that ran but couldn't verify anything? A soft pass is a check
that executed, had no ground truth to assert against, and passed because passing was the
conservative default. My clearest example. A guest asks whether the place is free next weekend.
The calendar data behind the tool is stale, so the tool returns "unconfirmed." The draft
correctly hedges. And the check has nothing to compare it against. Letting that pass is right. Counting it as a
plain PASS is not, because the summary then reads CLEAN while reporting on a dimension it never
exercised. *The check:* give soft passes their own count and their own line. When there are
many, the honest reading is not "the suite is green," it's "this dimension went untested."
That's the green-light villain again, wearing a different hat.

What volume are these lessons true at? An eval bounded to no scale is describing a system
in a vacuum, and the bound belongs in the write-up rather than in the author's head. Mine is
small: tens of conversations a day, in supervised rollout, sized for correctness under human review
rather than for throughput. That bound is load-bearing. At ten times the volume, the cost of
a judge call per draft starts to matter and the nightly run stops being free. At a hundred,
the human reviewer is the bottleneck and the whole design premise changes, because the
approval signal I lean on in production only exists while a person reads every draft.
*The check:* write the number down, then say what breaks at 10x. If nothing breaks, you
haven't thought about it yet.

Four of those I do. Three I don't. The one I most want back is the first, because it's half
an hour and it would have made this entire article unnecessary.

There's one more habit that isn't a trust criterion but sits next to them. Build cases from
failures you've seen, not failures you can imagine. My set grows the only honest way — a new
pattern shows up in real traffic and becomes a case. That turns out to be standard advice
among people who do this for a living, which I found mildly deflating and then reassuring.
None of the traps in this piece were exotic. Every one has a name, and knowing the names is
apparently not the same as avoiding them.

## Part 6 — The questions I would ask now

I set out to find out which model was safer, and the useful answer turned out to be that my
instrument couldn't tell. So instead of a recommendation about tooling, here is what I would
ask before building the next one, in the order the questions actually decide each other.

The first one is whether the question has an exact answer, and it settles most of the rest. If
it does, a deterministic check owns it and nothing else gets to cast the verdict. If it doesn't,
a judge is the right tool and you now owe it a validation set. A real cost, not a formality. Getting this backwards is the whole of Part 2. Cheapest mistake to avoid, easiest to make
when a judge is already sitting there.

Then ask whether you are testing capability or reliability, because they take different
instruments. Capability tolerates a small sample and a soft threshold; one passing run proves
the thing can happen. Reliability does not, because one failure is the finding rather than the
noise. Withhold-precision is reliability. Six cases cannot carry that claim however they come out.

The third is what event your number actually describes. Walk the path from the thing you measure to
the thing a person receives, and count the steps between. Every step is somewhere a green
light can be true and irrelevant at once. Mine had one I hadn't drawn.

Then, who labelled the data, and would a second person label it the same way. If the answers are
"me" and "unknown," that is not disqualifying. It is a bound on what the number can support, and it belongs beside the number.

Last, and this one costs nothing: write down what the results would look like if the eval were
flattering you, and write it down before the run. Then check which of it happened. Afterwards is not the same exercise. By then you can explain
anything.

The honest closing position is that my own answers are incomplete. The token detector has no
measured false-positive rate. There is one rater and it's me. The withhold set is six cases and
grows only when production embarrasses it. The judge still has no validation set. So the diagnostic signal I demoted it to is one I
cannot calibrate.

None of that is an argument for a different model. Every one is a gap in the part no model was
ever going to fill, which is knowing whether the thing measuring your system is telling you the
truth.

## Closing thought

An eval is production software with its own failure modes, and the most dangerous is the one
that fails toward telling you what you want to hear. A metric that flatters its author. A
recall that quietly measures the wrong event. The failures in this piece all point the same
friendly direction, and that friendliness is the tell.

My metric told me the safe model was dangerous. The drafts told me the metric was.

I believe the drafts now.
