
AI Email Triage Pipeline
I built a system that reads incoming customer email, figures out what each message actually needs, and writes a reply draft for only the ones that should get an automatic answer. The whole thing runs in n8n calling the Claude API.

It hit a 93% routing accuracy rate on a 40 email test set. It started at 78%. The interesting part of this writeup is how I got it from one to the other, and the decisions behind it.

<img width="1988" height="931" alt="n8n_workflow_canvas" src="https://github.com/user-attachments/assets/fa907c2e-1ffb-4953-b8c0-874de0479524" />
<img width="1572" height="1341" alt="n8n_workflow_score" src="https://github.com/user-attachments/assets/297096e2-ad8d-45e1-9483-050d3ca272d4" />

The problem
A small business inbox is a pile of disorganized “stuff” wearing matching outfits. A broken product complaint wants a fast, friendly reply. A double charge needs a real person. An allergy complaint is a legal exposure time bomb. No one wants to look at phishing emails. Sorting all that by hand is slow, arduous, and easy to get wrong.

Aiming an LLM at the whole inbox and letting it auto reply is worse, because it will happily answer a billing dispute or a scam with total enthusiasm. So the goal was easy to say but harder to build: handle the easy stuff automatically, push anything sensitive or uncertain to a human, and never lose a message along the way.
How it works
Seed Emails -> Classify (Haiku 4.5) -> Validate & Flag -> Switch -> draft  (Sonnet 4.6 writes a reply)

                                                                 -> review (held for a human)

                                                                 -> ignore (cold shouldered into junk, no action)

                                                                 -> fallback (safety net)

Each step has one job:

Classify sends each email to Claude (Haiku 4.5) and gets back structured JSON: category, urgency, a one line summary, a suggested action, a safety flag, and a confidence score.
Validate & Flag strips the markdown fences off the model's output, parses it inside a try/catch so a bad response can never crash the run, checks that the fields are valid, then applies the business rules that decide where each email goes.
Switch sends each email down one of four lanes based on that decision.
draft is where Claude (Sonnet 4.6) writes an actual reply, but only for safe, confident, non sensitive mail. Drafts are never sent automatically.
The decisions that actually matter
The architecture was the easy part. These are the choices that make it work, and every one was a tradeoff I made on purpose.

Drafts only, never auto send. Even on the automatic lane, the system writes a draft, not an outgoing email. Worst case is a human deletes a bad draft. No one wants the worst case: an angry customer getting “Thank you for your input! Have a wonderful day” as a reply.

Three lanes, not two. The obvious build is "auto reply or human." But a real inbox has a third outcome: junk that should be shoveled to the ignore box, with no reply and no human time wasted on it. So mail routes to draft, review, or ignore. That keeps obvious spam off a person's desk completely.

Validation that cannot fail quietly. The model's output gets parsed inside a try/catch. Anything malformed gets flagged for a human with the raw text kept, instead of crashing or silently vanishing. In a support inbox, "never lose a message" matters more than raw accuracy, and this is where that promise lives.

Business rules beat model confidence. Billing and personal email go to a human even when the model is 95% sure of itself, because the cost of an automated reply about someone's money or private life is high. The model classifies. The rules decide. A confident model does not get to overrule the policy.

Safety detection by understanding, not keywords. Health complaints, legal threats, and exposed personal data go straight to a human no matter the category or confidence. I detect this by asking the model to set a safety flag, which is a comprehension job, rather than scanning for keywords. A keyword list would miss something like "my throat closed up." This check runs first, ahead of even the spam filter, because a legal threat buried in a spammy looking email still needs eyes on it. It’s far safer to let the LLM work its comprehension than to parse keywords against a million different scenarios.

Cheap model to sort, better model to write. Classification runs on Haiku 4.5, which is fast, cheap, and fires once per email. Reply drafting runs on Sonnet 4.6, which is stronger and writes the thing a customer actually reads. Matching the model to the difficulty of the job keeps the cost at bay without making the replies worse.
Testing it
How I measured it. I built a 40 email test set modeled on a real small business inbox, and tagged each email with its correct category, urgency, and expected handling. That tagging is the answer key. On top of the normal emails I planted 9 deliberately tricky ones: a scam dressed up as a big order, a billing problem disguised as a website bug, a personal note that smuggles in a business request, a phishing email faking urgency, a health complaint that looks like routine support. The point was to test the system's judgment, not just whether it can handle the easy cases. A scoring step compared each email's actual route against the answer key and gave me a number.

How it went from 78 to 93. The score moved through four versions, and every change came from reading the specific emails it got wrong:

Version
What changed
Accuracy
Baseline
Classify, validate, three way routing, sensitive category rule
78% (31/40)
Fix 1
Stopped letting the model's suggested action drive routing. It was sending personal mail to ignore.
88% (35/40)
Safety flag added
Added the safety override, but with an over eager "when in doubt, flag it" instruction
83% (33/40)
Fix 2
Tightened the safety definition with explicit exclusions to cut the false alarms
93% (37/40)


The drop from 88 to 83 speaks the most. Adding a correct safety rule made the score go down, because the model started over flagging and pulling product complaints and spam into the review queue. This is a precision tradeoff in real life. For safety stuff, leaning towards over flagging is the right instinct, since a missed allergy complaint costs way more than an extra human review. But it was too aggressive and clogging the ignore lane with junk. Telling the model exactly what not to flag, with a clear list, cut the false alarms while keeping both real safety cases protected.
The three it still gets "wrong"
I quit tuning at 93% on purpose. Every miss that's left is a judgment call, not a bug:

The overpayment scam went to review instead of ignore. The model wasn't sure about a weird $6,800 "I'll overpay by check" order, so it sent it to a human. That's a miss against the answer key, but a person catching a check scam is honestly the safer outcome anyway.
The influencer collab went to draft instead of review. Whether an unsolicited collab pitch should get a human or an auto reply is a real gray area that different businesses would call differently.
A loud all caps order question went to review instead of draft. The model read the panic and a coupon mention as billing related, which tripped the sensitive category rule. Too cautious, but caused by a rule I'd defend since LLMs typically can’t answer that anyway.

I won't chase 100% because forcing those three to pass would mean bending the rules around one specific test set. That's overfitting, and it would make the system worse on actual mail it has never seen. A 93% I can explain beats a 100% I cheated my way to.
Limits and what's next
The safety flag leans on the model to catch things. Because I handed detection to the model, a miss there means a real safety issue could slip through. It sits on top of the category rules as a backup, not as the only safeguard, but a real deployment would want a second check.
The input is a fixed dataset right now. The next step is a live Gmail trigger so it runs on real incoming mail, plus a Google Sheet so flagged items land in an actual review queue.
No feedback loop yet. A production version would log the corrections a human makes on the review lane and feed them back to sharpen the prompt over time.
What it cost
The entire project, including every test run while I was tuning it, came to 44 cents in API spend. The cheap classification model runs on every email, and only the handful of safe ones per run reach the pricier drafting model. That split is the whole reason it stays this cheap.
Stack
n8n (self hosted with Docker), Claude API (Haiku 4.5 for classification, Sonnet 4.6 for drafting), and JavaScript for the validation, routing, and scoring.

