---
title: "How I use AI in 2026"
date: 2026-04-25T12:14:10+02:00
categories: ["ai", "development"]
description: How I use AI in my daily work as a maintainer and developer, from coding to triaging PRs and CI failures
---

# It's funny

I had a draft post sitting in my local repo for a while, where I was about to scream about how AI is overestimated. Well, that post aged pretty badly. I never published it, and looking back at the notes I'm glad I didn't. So what I'm going to write today will only be about my current workflow and how I actually use AI in my daily work — no hype, no predictions, just what I've found useful.

## My setup

I run Claude Code with `--dangerously-skip-permissions` inside a libvirt VM. Running it in a VM adds a layer of isolation I'm comfortable with when giving an agent broad permissions to run commands. My configuration and scripts for setting this up live at [clauderunner](https://github.com/fedepaol/clauderunner).

I work with tmux and keep at most 3 sessions running in parallel, each working on a different task. Beyond that, it becomes hard to keep up — I want to review what each agent produces before moving forward, and three is about the limit where I can do that without losing track. I found Mitchell's Hashimoto suggestion to always have an agent running interesting, and trying to build my own variation of it. It's also true that sometimes I need to stop and gather all the open threads I left hanging, so I don't want to have too many of them.

I also use [caveman](https://github.com/JuliusBrussee/caveman/) to cut down Claude's verbosity and reduce the number of tokens a bit. By default it narrates everything it's doing in great detail, which I find more distracting than helpful. I don't need the narration, I just need the results.

## Adding new features to the projects I am working on

This is the most obvious use case and probably where I get the most value. I work primarily on [MetalLB](https://github.com/metallb/metallb) and [OpenPerouter](https://github.com/openperouter/openperouter), both of which are non-trivial Go projects with real users, so the bar for quality is high.

I use [speckit](https://github.com/github/spec-kit) intensively. Unsurprisingly, the more time I spend upfront drafting a precise spec and carefully reviewing each intermediate artifact — the plan, the task breakdown — the less I need to iterate on the generated code. Vague instructions produce vague code. A well-structured spec acts as a forcing function that keeps the agent on track and reduces the number of correction cycles significantly. Also, if I have a specific structure or architecture in mind and I describe it carefully, the quality of the output is much better.

For larger features I enable `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` and spin up a team: typically 3 coding agents working in parallel, 1 reviewer, and 1 QE agent writing tests alongside the implementation. For smaller, well-scoped changes a `context.md` file with hand-written instructions is sufficient — no need to over-engineer the scaffolding.

Once the code is generated I use [diffity](https://github.com/nilbuild/diffity) to review it. It gives me a convenient way to annotate the diff with comments and then ask the agents to iterate on them. It's a tighter feedback loop than editing files by hand.

So why I am not pushing out one thing after the other? The initial outcome, even after I've reviewed it, is never the finished product. It still has to pass CI (and we know it's painful!), and it still has to survive the GitHub review process. Real reviewers (or other agents) catch things that neither I nor the agent noticed. When a comment is straightforward to address, I just tell the agent to read the review and fix it. For anything more subtle, I stay involved.

## On pushing code without reviewing it

I feel it's unfair to push code I haven't read. Not reviewing your own PR means deferring the work you were supposed to do in the first place to the reviewer — that's disrespectful of their time, and it offloads the cognitive burden of understanding the change onto someone else.

I'm often on the other side, and I can immediately tell when a PR was generated and pushed without any care. The signals are familiar: inconsistent naming, code that can be moved to functions, tests that don't really test anything. Low-quality, unreviewed AI output is one of the more frustrating things landing in open source review queues right now.

## Triaging CI failures

All my projects use [Ginkgo](https://github.com/onsi/ginkgo) for e2e tests. I care a lot about CI reliability — a flaky test suite is worse than no test suite, because it's not reliable. One rerun here, on rerun there and suddently you don't know anymore what works and what doesn't.

To make failures actionable, whenever a test fails the system dumps the current state of the cluster plus the last 10 minutes of logs from all relevant components. That's a lot of data, and manually correlating it across multiple files is tedious and error-prone.

I built [artifactsdownloader](https://github.com/fedepaol/artifactsdownloader) to fetch those artifacts from a failed PR run, and wrote a skill at [analyze-test-failures](https://github.com/fedepaol/skills/tree/main/analyze-test-failures) that ties the whole thing together: given a PR, it downloads the artifacts and asks Claude to identify the most probable root cause and whether there's a quick fix available.

Even when the suggested root cause turns out to be wrong, it's still genuinely useful. Claude is very good at correlating events across multiple log files and reconstructing the timeline of what happened when. Even when I don't buy the proposed explanation, I already have an instance with all the right context loaded — I can just ask follow-up questions and explore the failure interactively. That's significantly faster than doing it by hand, taking manual notes, and building a mental model of the failure from scratch each time.

## Triaging PRs

I took inspiration from Steve Yegge's [Vibe Maintainer](https://steve-yegge.medium.com/vibe-maintainer-a2273a841040) post and started using CI to automatically classify incoming PRs — understanding at a glance whether a PR is mergeable, needs work, or has structural issues that require a conversation.

The classification saves time on the triage step. Rather than reading every PR cold, I have a starting point: a summary of what the PR does, whether it's consistent with the project's conventions, and any obvious issues. I can then focus my attention on the PRs that actually need it.

On a few occasions I've also taken the approach Yegge describes: picking up a contributor's PR, improving it, and merging it directly rather than waiting on multiple review cycles. It's not the right call for every PR — sometimes the back-and-forth is the right process — but for stalled contributions it can cut the round-trip time considerably and be a better experience for the contributor too.

The skill I use for this is at [triage-prs](https://github.com/fedepaol/skills/tree/main/triage-prs).

## Improving the workflow

A side effect of generating more code is that more PRs end up in review, both from myself and from contributors. I started noticing that a significant chunk of my review comments were the same things over and over — the same patterns, the same nitpicks, the same structural issues.

So I asked Claude to go through my past review comments and identify the 10 most common ones. Then I asked which of those could realistically be caught automatically with a linter or a script. Several of them could. I implemented those checks and wired them into the CI pipeline, so they fail fast rather than showing up in a human review.

For the remaining patterns — the ones that are too contextual for a static check — I updated my Gemini reviewer configuration to flag them. The goal is to move as much of the mechanical feedback as possible out of the human review loop, so that by the time I'm reading a PR, the low-level stuff is already handled and I can focus on the things that actually require judgment.

## Learning

I use Gemini to get up to speed on things like new protocols and technologies. Recently that meant things like IS-IS routing internals or the OpenShift installer architecture — topics where the official documentation is dense, or assumes a lot of background knowledge I didn't have.

Being able to ask questions interactively, get explanations pitched at the right level, and then immediately drill into the parts I didn't follow is a much faster ramp-up than reading specs or documentation linearly. It's not a replacement for eventually reading the primary sources, but it's a much better way to build the initial mental model before you do.

## Building stuff locally

Tools like [containerlab](https://containerlab.dev/) and [kind](https://kind.sigs.k8s.io/) are great for spinning up local network environments, but configuring them from scratch can be fiddly. Claude makes this much more practical — I describe the topology I want at a high level and let it handle the configuration details.

One thing that works particularly well here is providing examples to take inspiration from. The output is noticeably better when Claude has a concrete reference rather than building from scratch.

A good example of this in practice: I wanted to build a lab mixing EVPN and SRv6 — a topology that requires getting several moving parts to talk to each other correctly. Rather than working through it manually, I wrote some reference configs and asked Claude to assemble the topology I had in mind. The result is at [srv6lab](https://github.com/fedepaol/srv6lab/tree/srv6evpn). What I found particularly satisfying was watching Claude triage and fix misconfigurations on its own — identifying why two nodes weren't establishing a session, adjusting the config, and verifying the fix. That kind of iterative debugging loop on network configuration is exactly the sort of thing that used to eat up a lot of time.

## Conclusion

AI is saving me a real amount of time, although I'd find quite hard to say "how much" (but this is material for another post!).

That said, I still validate the output. Both for correctness and for maintainability. Code that works but is impossible to reason about creates problems down the line (and I am pretty vocal on caring about [keeping the cognitive load low](https://archive.fosdem.org/2023/schedule/event/goreducecognitive/)), and those problems land on maintainers — often the same people who already put time into the original review. Skipping that validation step is a short-term gain with a long tail of costs.

I was deeply skeptical when all of this started, so I'm not going to make bold predictions about where it's headed. My skepticism was wrong before and I don't trust my own forecasting here. What I can say is where things stand today, for me, in my specific workflow.

The way I work has changed a lot. I spend more time in maintainer mode — writing detailed specs, reviewing generated code, thinking about structure and architecture — and less time writing code directly. I'm at a point in my career where that trade-off suits me fine. Writing less code and focusing more on the shape of a system is something I find more interesting anyway.

This is still a new field and I'm learning as I go. A lot of what shaped my current workflow came from sources that incidentally landed in my timeline rather than from any deliberate research: [Steve Yegge](https://steve-yegge.medium.com/) and [Addy Osmani](https://addyosmani.com/blog/) wrote things that changed how I think about this. The [Pragmatic Engineer podcast](https://www.pragmaticengineer.com/podcast/) has also been a good signal in a space full of noise.

If you have better approaches, know reputable sources I should be following, or think something I'm doing could be improved, I'd genuinely like to hear it. You can reach me at fedepaol@gmail.com or leave a comment below.



