# bosons.github.io
---
layout: post
title: "I Built the Recruiter’s Version of career-ops"
subtitle: "Finding developers by what they build and write, not what their résumé says"
author: Subhendra Basu
date: 2026-10-29
tags: [talent-ops, career-ops, claude-code, recruiting, open-source]
description: "talent-ops is an open-source tool that finds developers by what they build and write, not what their résumé says."
---

*Finding developers by what they build and write, not what their résumé says*

**By Subhendra Basu**
GitHub: [github.com/bosons14/talent-ops](https://github.com/bosons14/talent-ops)

## The best engineers aren’t on job boards

The developers I most want to hire are rarely looking for a job. They are shipping a library that a few thousand people depend on. They are writing a long post explaining why their KV-cache trick works. They are fixing someone else’s bug at 2 a.m. in a repo with 18,000 stars.

I have spent 24 years in EDA, GPU systems programming and applied AI. These days I also sit on the other side of the table, conducting AI and Gen AI interviews. From there one thing is obvious: a résumé tells you what someone claims, but their public work shows you what they can do.

Recruiters know this too. But searching GitHub, Medium, Substack, DEV, Hashnode and Hacker News by hand, then working out which accounts belong to the same person, takes days per role.

Then I came across [career-ops](https://github.com/santifer/career-ops), an open-source job-search system that runs inside Claude Code. It scans portals, scores job descriptions and tailors CVs for the candidate. It made me ask the obvious question: where is the version for the people doing the hiring?

So I built one. It’s called **talent-ops**, and it’s open source.

## What talent-ops does

You give talent-ops a role and a few technical topics, such as “LLM agents”, “RAG” or “LLM inference”. It returns a ranked shortlist of developers who both build and write about those topics, with links to the evidence behind every name.

It searches two kinds of places.

**Writing:**

- Medium (tag feeds and authors you follow)
- Substack (newsletters you list)
- DEV / dev.to (top posts per tag)
- Hashnode (popular posts per tag)
- Hacker News, to discover independent blogs
- Any RSS or Atom feed

**Code:**

- GitHub (repos by topic, sorted by stars, plus the top contributors of popular org repos)
- GitLab (gitlab.com or self-hosted)
- Codeberg and other Gitea/Forgejo hosts
- Bitbucket (workspaces you name)

The workflow mirrors career-ops. Each step is a command inside Claude Code:

| Command | What it does |
|---|---|
| `setup` | Paste a job description and Claude turns it into a search brief and config. |
| `scout` | Collect public evidence from every enabled source. |
| `rank` | Merge, score and filter, for example by location, into a Markdown shortlist and a CSV. |
| `profile` | Build an evidence pack on one person; Claude then reads their actual code and posts and writes an assessment. |
| `outreach` | Draft a short, specific first message that refers to something the person actually built. It is a draft only; you send it. |
| `tracker` | Keep a simple pipeline (shortlisted, contacted, replied and so on). |

The scripts do the tedious part: searching, deduplicating, scoring. The AI does the part that needs judgment: reading the work and deciding whether it fits the role.

## Three problems worth solving properly

### 1. One person, six accounts

The same engineer might be `asharaman` on GitHub, `@asha_builds` on X, the author of a popular DEV post, and the owner of a personal blog that hit the Hacker News front page. Treating those as four candidates is useless. Merging them on name alone is dangerous: there are a lot of people called Rohan Kapoor.

talent-ops gives every piece of evidence one or more identity keys, such as `gh:asharaman`, `devto:asharaman`, `twitter:asha_builds` or `domain:asharaman.dev`. Keys are joined with a union-find structure, but only on explicit links:

- a DEV profile that lists its GitHub username
- a GitHub profile whose website or X field points to the blog
- a Hashnode profile with social links
- the same URL appearing in two sources

Two accounts that share a display name are flagged as a possible match for a human to check. They are never merged automatically.

### 2. Scores you can argue with

A black-box “talent score” is worse than none. Each talent-ops score is a weighted mix of five components, and the report shows every one of them along with the reasons:

- **code:** stars on on-topic repos the person owns, plus commits to popular repos, weighted toward recent work
- **writing:** on-topic posts, weighted by engagement and recency
- **influence:** followers
- **topic coverage:** how many of the search topics the person shows evidence for
- **builds and writes:** a bonus for people who do both

Each component is scaled against the 90th percentile of the pool, so one superstar doesn’t flatten everyone else. The weights live in a config file. Hiring a developer advocate? Raise writing. Hiring a kernel engineer? Raise code.

Popular repos are often owned by an organisation rather than a person, so talent-ops also expands the biggest org repos into their top human contributors, skipping bots. That is how the quiet maintainer with 340 commits to a project with 18,000 stars shows up.

### 3. Being honest about what platforms allow

Every platform has limits, and the tool says so rather than hiding them:

- **Bitbucket** has no stars and no public search, so talent-ops can only check workspaces you name, ranked by watchers.
- **Medium** RSS feeds carry no clap counts and return only the newest ~10 posts per tag.
- **Substack** has no official search API. talent-ops reads the newsletters you list; discovery through an undocumented endpoint is off by default.
- **GitHub** allows only 60 requests an hour without a token. A free token raises that to 5,000.

The whole thing is plain Node.js with zero dependencies: built-in `fetch`, a small RSS/Atom parser, a polite HTTP client with per-host throttling, retries and an on-disk cache. Clone it and run it.

## Sourcing without being creepy

A tool that finds people can easily be used badly. So the rules are written into the project’s agent instructions, which Claude follows in every step:

- **Public, professional data only.** No scraping LinkedIn, no collecting emails from commit history, no trying to unmask pseudonymous accounts.
- **No ranking on personal characteristics.** The tool never infers gender, ethnicity, age or anything similar from a name or a photo. Location is used only when the role needs it, and only as the person states it on their own profile.
- **Draft, never send.** Claude writes outreach; a human reads it and decides whether to send it.
- **Opt-outs are permanent.** Mark someone do-not-contact and they never appear in a shortlist again.
- **Data protection.** Candidate data is personal data under GDPR, India’s DPDP Act 2023 and similar laws. Keep only what the search needs and delete on request.
- **Evidence, not verdicts.** Every report opens by saying the score ranks visibility of public work, not ability.

That last point matters most. Stars and followers favour people who work in the open, in English, and who have been visible for years. Many excellent engineers have no public footprint at all. talent-ops is a way to find some great people faster, not a way to judge everyone.

## Try it, break it, improve it

You need Node.js 20 or newer. There is nothing to install.

```bash
git clone https://github.com/bosons14/talent-ops
cd talent-ops
npm run demo           # offline demo with made-up people
export GITHUB_TOKEN=…  # free token, no permissions needed
claude                 # open Claude Code, then:
/talent-ops setup      # paste a job description
/talent-ops scout
/talent-ops rank
```

The scripts also run on their own, without Claude: `node scout.mjs`, then `node rank.mjs --location "India,Remote"`.

This is version 0.1. The core logic is covered by tests run against sample data. The live connectors are written to each platform’s published API and need real-world mileage. Here is where I would love help:

- testing each connector against the live platforms and reporting what breaks
- new sources: Kaggle, Hugging Face, Stack Overflow, arXiv, conference talk listings
- a small web dashboard for the shortlist, like the one career-ops has
- better relevance matching using embeddings instead of keywords
- export to common applicant-tracking systems

If you recruit for deep-tech roles, I’d like to hear what you’d want from a tool like this. If you are a developer, [star the repo](https://github.com/bosons14/talent-ops), open an issue, or send a pull request.

The best engineers are already showing their work in public. talent-ops makes it easier to notice them.

---

*Thanks to [santifer](https://github.com/santifer) for [career-ops](https://github.com/santifer/career-ops), which inspired this project.*
