---
prompts:
  - prompt: Add a Slack bot to this Next.js app that answers mentions and DMs from the asking user's own data and keeps the thread for follow-up questions.
    stack: Postgres
  - prompt: Have the bot post a weekly report to a channel, and draft emails that a person approves in Slack before they are sent.
    stack: Postgres
  - prompt: Our Slack bot sometimes answers twice and loses the thread. Find out why.
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/slack-ai-bot) on timerise.ai.
