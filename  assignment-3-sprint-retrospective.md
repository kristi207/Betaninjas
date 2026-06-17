Assignment 3 — Our Last Sprint: Honest Look
Sprint: Bubbles

When did QA get involved?
QA was involved at every step of the sprint. The team shared the plan when new features were being added and asked for feedback. Once designs were made, QA reviewed those too. This was not a team where QA only showed up at the end — involvement started at planning and continued through development.

Real Examples
QA (preventing a problem):
QA was present during planning and design reviews, giving feedback before development started. This gave the team a chance to catch misalignments early before any code was written.

QC (finding a problem):
During the timer feature build, QA tested it in staging and it appeared to work correctly. However, when it reached production, pro users were getting 2 weeks of access they were not supposed to have. This was a defect found after the feature was built — a real QC moment.

Testing (running a test):
Testing the timer feature in the staging environment to verify it functioned as expected before release.

Agile Ceremonies
QA was part of every Agile ceremony — planning, standup, review, and retro. Full involvement throughout the sprint.

One Thing to Change in the Next Sprint
The timer bug worked fine in staging but broke in production specifically for pro users. Staging was not testing the right conditions — pro user access was not being properly simulated.

The change: Before any feature that touches user tiers or permissions is marked ready for release, QA must run tests using a real pro account in a staging environment that mirrors production. This becomes an explicit exit criterion — not just "does it work" but "does it work correctly for each user type."