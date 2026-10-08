# Building a Question Bank That Is Actually Useful

Yesterday was about making AWS Quest work as an application. Today was mostly about giving it something worth studying.

I focused on the **AWS Certified Cloud Practitioner (CLF-C02)** content. Instead of dumping hundreds of unrelated questions into one file, I organized the material into exam domains, smaller concepts, short lessons, and questions attached to those concepts.

## What I worked on

The four domains are **Cloud Concepts**, **Security and Compliance**, **Cloud Technology and Services**, and **Billing, Pricing, and Support**. I expanded the concept lessons and question sets domain by domain, then updated the content inventory to track what was planned and what was actually present.

One commit completed Cloud Concepts at **18 concepts and 83 questions**. Later work filled out the other domains, and the inventory eventually reached **90 concepts, 90 lessons, and 415 active questions** across CLF-C02. Those numbers describe the seeded content, not proof that every question is pedagogically perfect.

The other important task was **question-bank validation**. A large JSON bank is surprisingly easy to damage: duplicated identifiers, missing concept links, invalid answer data, or content that silently fails to load can make the study experience unreliable. The validation work aims to catch content problems before they become confusing behavior inside the app.

I also learned to distinguish **content coverage** from **mock-exam weighting**. The domain with the most AWS services needs more concepts and practice material, but that does not mean it should automatically dominate a timed exam. The mock exam can select questions according to the certification blueprint while the content bank remains broad enough for focused study.

## Things that clicked

**Content is part of the product, not filler added at the end.** A working quiz screen means very little if the explanations are shallow, the distractors are misleading, or important concepts are missing.

**Tracking coverage helps avoid false progress.** A bank of 400 questions sounds substantial, but the number alone does not reveal whether entire topics are absent. The concept inventory makes that gap visible.

**Validation and review solve different problems.** Automated checks can catch structural mistakes, but they cannot decide whether an explanation is accurate, current, or genuinely helpful. Question quality still needs manual review against trustworthy AWS sources.

**Studying a certification also teaches content design.** Writing useful comparison and scenario questions forces me to distinguish similar AWS services and explain *why* one answer fits better than another.

## Still thinking about

The CLF-C02 bank has reached its planned size, but completion is not the same as quality assurance. I still want to review accuracy, wording, coverage, difficulty, and whether explanations help me correct my mental model. SAA-C03 content is a separate, larger task.

The main takeaway today: a learning engine can decide *when* to ask, but the question bank decides *what* someone learns.

---

**Project:** [aws-quest](https://github.com/muafa7/aws-quest)
