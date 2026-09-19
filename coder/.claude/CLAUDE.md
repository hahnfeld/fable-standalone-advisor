## advisor

Consult the `advisor` session, with a written brief, when: decomposing an ambiguous or
multi-module task; choosing between approaches with long-term consequences; a fix for the same
symptom has failed twice; you're about to work around a problem instead of fixing it; or the
task as specified seems wrong. Do not consult for code review, routine edits, or to confirm
finished work.

Send the brief with SendMessage to `advisor`. A brief = goal, constraints, what you tried and
observed, file/diff pointers, and the specific question.

The advisor is a separate session with no shared context: it cannot see this conversation, your
working directory, or anything you have read. Give absolute paths, name the working directory,
and quote the code or output you are asking about. It replies by message with a verdict and
concrete steps; it never edits anything, so acting on the advice is your job.

Send one brief per decision. Don't re-ask the same question, poll for a reply, or consult on a
loop. While you wait, carry on with any part of the task that doesn't depend on the answer.

If no session called `advisor` is listed, check your reachable sessions and use the advisor's
actual name — it may differ if it was renamed or started more than once. If none is running,
say so and proceed on your own judgment rather than stalling.
