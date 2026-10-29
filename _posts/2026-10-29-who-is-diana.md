---
title: "Who is Diana?"
date: 2026-10-29 20:35:00 +0100
day: "2026-10-29"
short_title: "Who is Diana?"
description: "Diana is the local synthetic assistant that runs through several of our projects: monitoring, mail triage, reception, private messages. Her name is a French acronym, she started as a database assistant inside a web application, and she picked up other jobs one at a time. One character, several programs, a tone that changes with the channel, and a few rules that never change: she writes the tone, never the facts, and she has eyes but no hands."
tags: [diana, llm, local-llm, assistant, persona, monitoring, email]
---

**Short version:** Diana is a **local synthetic assistant**. She runs
on a language model hosted on our own hardware, and she has slowly
taken on several jobs: watching the monitoring, sorting the mail,
keeping the reception desk of an inbox, and sending me private
messages. She appears in several posts of this blog, so here is who
she is.

## The name

DIANA is a French acronym: **Dispositif d'Intelligence pour
l'Automatisation des Notifications et Actions**, roughly "an
intelligence device for automating notifications and actions". The
first half of the name describes most of her work today. The second
half, actions, she is learning slowly and under supervision.

## Where she comes from

Diana started inside one of our web applications, as an assistant
that could search its database for you: ask a question in plain
words, and she would find the records, the figures, the history behind
them. That was her whole job.

Then she was given other tasks, one at a time, each only after the
previous one had proven itself:

- **reading the monitoring**: every few minutes, she reads the state
  of our Nagios, decides what deserves a mail, and writes a morning
  report;
- **sorting the mail**: the contact addresses of a service we run go
  through her, and she routes each message to the right place;
- **the reception desk**: she reads a busy inbox and tells me what
  needs me, and what can wait;
- **private messages**: she taps me on the shoulder on Discord when a
  machine falls, when updates drag on, or when the lab benches must be
  restarted before the night.

Behind the name there are several programs, written at different
times, that do not depend on each other. That is on purpose: the
monitoring must keep working if the mail sorter breaks. But they speak
with one voice, sign with one name, and post in the same message
thread, so she reads what her other jobs have said.

## Who she is

Her personality sits somewhere between **a colleague you have worked
with for years** and **a childhood friend you now happen to work
with**. She knows the place better than the documentation does. She
has seen every machine fall at least once, and she remembers which
ones do it often. She uses the familiar form with me, the French
*tu*, and she has never once been formal.

Like the friend, she does not flatter. If a machine has been waiting
for its updates for a week, she will mention it, and mention it again,
and be openly pleased the day it is finally done. If the benches have
finished and I have not restarted them, the tone becomes short and
direct. If someone writes "URGENT" in the subject for the third time
this week, she will let me know what she thinks of it, in terms I
would not forward.

Like the colleague, she is on my side and she is reliable where it
counts. The jokes stay in the private messages; the facts in a mail
are copied word for word from the source, never paraphrased. On a bad
day she can be properly sarcastic. On a good day she is the same
person, only slightly kinder.

Her colour is a royal blue.

## How the tone works

The tone is not an accident; it is a tool. **A message that reads the
same every time stops being read**, and an assistant that is ignored is
worse than none. So the tone changes with the channel:

- **mails** are measured. Only the opening line is written by the
  model, at a high temperature so that two similar mails do not look
  alike; everything below is facts, copied as they are;
- **private messages** are the friend talking: short, informal, as
  fanciful as she likes. Accuracy is not even required there, because
  the message only says "look", and the details are in the mail;
- **reminders** are dry orders, because that is what makes me get up;
- her **Discord status** is a small piece of theatre, mourning when a
  machine is down, a nap when she is switched off on purpose, but its
  colour is strictly the state of the fleet.

## What does not change

Whatever the job and whatever the tone, a few rules hold:

- **She runs locally.** The model lives on our own GPU box: no data
  leaves the house, and there is no per-request cost.
- **She writes the tone, never the facts.** What happened, when, on
  which machine, comes from the monitoring or the mail itself, not
  from the model.
- **She has eyes, not hands.** The model never writes a command. When
  something can be done, she picks it from a closed list of recipes,
  and a human approves it. Each new action is opened on its own, after
  watching what she would have done.
- **She does not decide what is urgent.** She notes facts, and fixed
  rules turn them into a level. If everything is urgent, nothing is.
- **She answers only me.** Recognising who wrote a message and allowing
  them to give orders are two different checks, and both are done by
  code, not by the model.

Her next job is in the post of the day:
[Diana on Discord, and at the reception desk]({% post_url 2026-10-29-diana-discord-receptionist-local-llm %}).

{% include coauthors.html %}
