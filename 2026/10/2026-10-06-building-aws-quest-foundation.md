# Building the Foundation of AWS Quest

Today I worked on the first version of **AWS Quest**, a personal AWS certification study app with a retro RPG feel. I wanted something more useful to me than a collection of flashcards: a place to read a short explanation, answer a question, notice what I got wrong, and come back to it later.

The first challenge was deciding what the application actually needed. I kept V1 deliberately local and small: **Next.js, React, TypeScript, Prisma, and SQLite**, without a separate API server, authentication, or runtime AI calls. The idea is to get the learning loop right before adding infrastructure.

## What I worked on

I set up the application structure, database models, and the main learning flows. The app separates learning a concept, practicing questions, taking mock exams, and reviewing progress. That matters because each screen answers a different question: *What does this mean? Can I remember it? Can I apply it under exam conditions? What should I review next?*

I also worked on the adaptive practice logic. Not every correct answer should count equally, and not every wrong answer means the same thing. Being **confident and wrong** can reveal a misconception; being **correct but unsure** means I probably need another review. The engine uses these signals alongside recent question history, accuracy, and review due dates when deciding what to show next.

Spaced review uses increasing intervals — **1, 3, 7, 14, 30, and 60 days** — so a concept should earn its way toward mastery through repeated recall rather than a single lucky answer. I added tests around this learning behavior and documented how the app and its content should work.

## Things that clicked

**A learning app is not just a question bank.** The interesting part is what happens *after* someone answers. The feedback, confidence signal, review schedule, and choice of the next question are what make it a learning system.

**A smaller architecture can be a feature.** For one person's local study tool, SQLite and a full-stack Next.js app make more sense than deploying several services. I can always revisit that trade-off if the product changes.

**Data structures shape learning behavior.** Concepts, lessons, questions, attempts, and mastery need to remain distinguishable. Otherwise, it becomes difficult to explain why the app thinks a topic needs more practice.

## Still thinking about

How do I know the adaptive rules actually help learning, instead of just feeling clever? Unit tests can show whether the ranking logic behaves as intended, but they cannot prove it improves memory. That needs real use over time.

For now, the milestone is a coherent foundation. Tomorrow's problem is making sure it has enough *good content* to be useful.

---

**Project:** [aws-quest](https://github.com/muafa7/aws-quest)
