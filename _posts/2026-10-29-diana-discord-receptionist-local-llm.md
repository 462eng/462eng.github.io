---
title: "Diana gets a voice and a desk: Discord messages, a status, and reception duty"
date: 2026-10-29 20:40:00 +0100
day: "2026-10-29"
short_title: "Diana on Discord, and at the reception desk"
description: "Diana, our assistant that reads monitoring incidents with a local LLM, now sends private Discord messages, shows the state of the fleet as her Discord status and can chat, all without the power to run anything. She also becomes the receptionist of a busy inbox: she notes facts and leaves the urgency to fixed rules, because if everything is urgent, nothing is. Plus two silent bugs found along the way."
tags: [llm, local-llm, monitoring, nagios, discord, email, triage, diana, mailtri]
---

**Short version:** Diana reads the incidents of our monitoring (Nagios)
with a language model running locally on the GPU box, and writes
summaries by e-mail. She now also has a **private Discord channel**
with me: short, informal messages, a status that reflects the fleet,
and a conversation. She still cannot run anything from there. She also takes a second
job: **receptionist** of a busy inbox, telling me what deserves my
attention, with rules rather than the model deciding what is urgent.

## Diana so far

(Who she is, and where her name comes from, is in
[a post of its own]({% post_url 2026-10-29-who-is-diana %}).)

Diana has been running since September. Every few minutes she reads
the state of Nagios, groups problems by machine, and asks a local
model (a 27B, so no API cost) for an assessment. A few principles have
not changed:

- **The model never writes a command.** It picks a name in a closed
  list of recipes, and anything else falls back to "a human looks".
- **The model never writes the facts.** It writes the opening line of
  a mail, at a high temperature so that two similar mails do not look
  alike; the plugin output is copied below it, word for word.
- **Severity has a mechanical floor.** The model can raise it, never
  lower it.

E-mail is the record: one incident, one thread, replies as commands.
What was missing was the tap on the shoulder.

## Private messages

Diana is now a private Discord bot. She sends me direct messages
through the REST API only, for a small set of events:

- **A machine is down**, and only that: never a service, and certainly
  never pending updates. One message per polling cycle, so a switch
  going down gives one message listing the machines behind it, and a
  "back after 12 minutes" when they return. Nothing for an acknowledged
  problem or a planned maintenance.
- **Updates that drag on**: a machine whose update check has not been
  green for 72 hours, with a reminder every 72 hours, between 09:00 and
  21:00. When it finally gets updated, she notices, and says so.
- **Lab benches to restart in the evening**: some test benches have to
  be restarted before 23:00 for the night. I restart them all at once,
  never one by one, so she stays quiet while one is still due to finish
  before the deadline, then reminds me every 20 minutes. These
  reminders are dry orders, by choice.

The tone of the other messages is deliberately the opposite of the
mails: a friend of long standing, as fanciful as the model likes, at a
temperature of 1.15. **Accuracy is not required**: the message only
says "look", and the details are in the mail. "RIP" for a machine that
fell is about as serious as it gets. A message that reads the same
every time stops being read, and that is the failure we design
against.

## A status that means something

Discord shows a status next to each member, and Diana's is computed
every minute from her state file:

- **online**: all is well;
- **do not disturb**: a machine is down;
- **away**: she has not polled Nagios for six minutes. The status is
  the heartbeat, so a dead Diana does not look like a healthy fleet.

The text under the name rotates too. In calm times it is drawn from 222
short sentences, and changes only twice a day, at 08:00 and 20:00.
When something is wrong, it switches immediately to alert sentences
(mourning, absence, investigation) that rotate every minute. When the
GPU box goes into game mode, Diana is stopped on purpose and shows
nap-themed statuses instead of an alarm. The status is visible to the
whole server, so it never names a machine or a person.

This service holds the bot's **only** gateway connection; everything
else goes through REST. Moving all messages through it is planned.

## Talking to her

In private messages, Diana now answers. Only my user ID gets a reply.
She has no memory of her own: the context is the last 30 messages of
the thread, plus a read-only view of the fleet state, and one reply is
written at a time. **She executes nothing.** If actions come to
Discord one day, they will work like the mail commands: a fixed
keyword and a checked identity, never the model's interpretation of a
sentence. Identifying a message and authorising it are two different
things.

## Diana at the reception desk

Diana is one character, but two programs. Besides the monitoring, the
mail sorter (mailtri) triages the contact addresses of a service we
run, and it writes as Diana too. It now gives her a second job:
**reading a busy inbox** where many people write to me, and telling me
what needs me. She reads that mailbox read-only and never sends
anything from it; she writes to me from her own address.

The design rule came from experience: **if everything is urgent,
nothing is urgent**. So the model does not decide the urgency. It
notes **facts**: what kind of message it is, what situation it
describes, which documented procedure covers it, whether the sender is
asking for something outside that procedure, and what is missing. The
level is then computed by rules:

- the urgency **written in the message is ignored by default**, with
  per-sender rules: some senders are to be followed to the letter,
  others to be checked;
- an equipment breakdown is **never urgent** by mail, because the
  telemetry and the monitoring know before anyone writes;
- the time allowed to restore a service is counted from the **first**
  mail of a thread; a reminder does not restart the clock;
- "immediate" is a **closed list** of situations, safety first.

We replayed a full year of that inbox (277 messages) and compared with
what I actually answered and when. Nothing would have reached
"immediate" outside one deliberately listed kind of document, which is
the point. One correction came out of it: commercial mails written to
me personally were filed as noise. Adding an exception to the
definition in the prompt did not help, the 27B ignored it; adding a
separate fact, "personal", that the model fills and the rules read,
did. The model is better at reporting a fact than at applying an
exception.

Her receptionist messages land in the same Discord thread as the
monitoring ones, in a very different voice: sarcastic, borderline
insulting, always on my side. Since she reads the last 30 messages of
that thread when we talk, she knows what both of her jobs said.
At most ten a day, between 08:00 and 21:00, and none older than 24
hours, so that a day in game mode does not end in a burst.

## Two silent bugs

Working on this, we found two bugs that had been there for weeks
without a single error message.

**Swapped state codes.** Nagios' JSON interface encodes service states
as bit flags, and we had 8 and 16 the wrong way round: CRITICAL read
as UNKNOWN, and back. Every security update (CRITICAL by convention on
our fleet) reached the model as "unknown", the morning report called
those machines unreachable, and its security section never found
anything. The tests had been written with the same mistake, so they
passed. Tests written from the same understanding as the code do not
check that understanding.

**A key bound to an old address.** Diana's SSH key, used to check
machines and run approved updates, was restricted on each server to
the GPU box's address. That address changed in September; the key was
refused everywhere from then on. The report showed those servers as
"not verified" and the updates were quietly skipped, which looked like
a calm fleet for five weeks. Fixed, and noted: if Diana's host changes,
that restriction has to change with it.

## What next

Diana is still in observation for anything that acts: she proposes,
and I approve. Next: route every message through the status service,
and teach her, as receptionist, to check whether a request is already done
(a report already sent, a bench already running) before raising it.

{% include coauthors.html %}
