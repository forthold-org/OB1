# Open Brain: A Twenty Minute Audio Tour

A spoken-word tour of the OB1 repository, written for text to speech apps like Speechify on iPhone. The four diagrams referenced throughout this script live next to it in the diagrams folder. Each diagram is a Mermaid file, so you can render it in GitHub, in any Markdown viewer that supports Mermaid, or by pasting it into the Mermaid live editor.

If you are following along visually, keep the diagrams folder open in a second window. If you are listening hands free, do not worry. Every diagram is also described in words as we go.

---

## Section One. What Open Brain Actually Is

Welcome. This is a tour of Open Brain, also called OB1, a repository created by Nate B. Jones and maintained by a small team alongside a growing community.

The pitch is one sentence long. Open Brain is the infrastructure layer for your thinking. One database. One A I gateway. One chat channel. Any A I tool you use can plug in. No middleware. No software as a service chain.

Most people who use A I today have a fragmentation problem. Their notes are in one app. Their email is in another. Their Claude history lives at Anthropic. Their ChatGPT history lives at OpenAI. Each tool has its own memory, and none of them share. The result is that every conversation starts cold, every helpful insight gets buried, and your most useful context is locked behind whichever vendor happens to own it that week.

Open Brain rejects that model. Instead of asking each vendor for a better memory, it gives you one shared memory and lets every vendor read and write to it. The shared memory is a database that you own. The protocol is open. The clients are interchangeable.

Concretely, Open Brain is built on three components. Supabase, a hosted Postgres database with a vector search extension called pgvector. OpenRouter, a single A I gateway that gives you access to many model providers at once. And M C P, which stands for Model Context Protocol, an open standard from Anthropic that lets A I clients talk to external tools and data. Put those three together, deploy a small server in front, and you have a personal knowledge system that any M C P aware client can hook into.

That is the elevator pitch. Now let us look at how the pieces fit together.

---

## Section Two. The System Architecture

Open the first diagram, named oh one system architecture, in the diagrams folder. It is a left to right flow chart with four labeled regions. Capture sources on the left, the Open Brain core in the middle, A I clients on the right, and frontend surfaces underneath.

Data enters Open Brain from capture sources. The most common ones are messages you send to a Slack or Discord channel you have wired up. You type a thought, the bot grabs it, and it is on its way to your database. Bulk imports are the other big category. The repo has importers for ChatGPT, Gmail, Obsidian, X, Instagram, Google Takeout, Perplexity, Grok, and Blogger. If you have years of writing trapped in some platform, there is probably a recipe in this repo to pull it out.

Once a piece of text arrives, it passes through a capture edge function. That is a tiny serverless function hosted on Supabase. It does light cleanup, calls OpenRouter to generate a vector embedding of the text, and writes the text, the embedding, and metadata into a single Postgres table called thoughts. That table is the beating heart of the system. Every piece of content you capture ends up there, regardless of which capture source it came from.

The thoughts table is simple on purpose. An i d, the raw content, an embedding column of one thousand five hundred thirty six dimensional vectors, a flexible metadata column stored as J S O N, and timestamps. There is a vector similarity index for fast semantic search, a general index on metadata for filtering, and a date index for time range queries. That is the entire schema.

On top of the database sits a remote M C P server, also a Supabase edge function. Any client that supports M C P can connect to it through a custom connector U R L. Claude Desktop, ChatGPT, Claude Code, Cursor, and a growing list of other clients all support that. You paste the U R L once, and from then on, that client can read your thoughts and add new ones.

The repo forbids one common shortcut. M C P servers in Open Brain must be remote edge functions. Not local Node servers. Not entries in a Claude Desktop config file. A remote server scales across devices and survives across machines. Anything local breaks the moment you switch contexts.

Frontends sit beside the database. Two dashboards in two flavors, SvelteKit and Next dot js. A daily digest recipe that summarizes recent thoughts by email or Slack. A Life Engine recipe that runs proactive briefings through Telegram or Discord. All of them read from the same thoughts table, so there is no syncing problem to solve. There is only one source of truth.

---

## Section Three. The Repo Topology

Open the second diagram, named oh two repo topology. It is a tree showing what lives at the root of the repository.

The repo has three major zones. Curated content, open content, and supporting material.

In the curated zone you will find two folders. The extensions folder is the learning path, six progressive builds that walk you from beginner to advanced. The primitives folder holds reusable concept guides that get referenced by multiple extensions. Curated means the maintainers approve new additions before they land. You cannot just open a pull request adding a new extension.

In the open zone you will find five folders. Recipes are standalone capability builds. Schemas are database table extensions. Dashboards are frontend templates that you can host on Vercel or Netlify. Integrations are M C P extensions, capture sources, and alternative deployment targets. Skills are reusable prompt packs for A I clients. All five accept community pull requests and run through an automated review.

In the supporting zone there are four folders. Docs contains the setup guide, the F A Q, companion prompts, and walkthroughs. Server contains the core M C P server source code, a single TypeScript file. Scripts holds maintenance tooling. Resources holds packaged exports and companion files. There is also a top level set of governance files including contributing, license, agents, code of conduct, and a security policy.

The naming convention across all five open categories is consistent. Each contribution gets its own folder. Inside that folder there is a readme and a metadata file. Everything else is up to the contributor. The repo does not impose a code structure on you. It only imposes structure on documentation and metadata.

---

## Section Four. The Extension Learning Path

Open the third diagram, named oh three extension learning path. It shows the six extensions in order, with the primitives each one introduces flowing in from the side.

Extension one is the Household Knowledge Base. Beginner. You teach your agent the facts about your home, the kind of trivia that is annoying to look up but easy to lose. Air filter sizes, wifi passwords, appliance model numbers. The build introduces deploying an edge function and connecting a remote M C P server to your A I client. Both become primitives you reuse.

Extension two is the Home Maintenance Tracker. Still beginner. Now you extend the system with a sense of time. When did you last change the air filter. When is the next service. By the end you have a working maintenance log that your A I can both read and update.

Extension three is the Family Calendar. Intermediate. Multi person coordination enters the picture. Who is at home this week. Who has a school event on Thursday. Who is traveling. The agent stops being a personal assistant for one person and starts being a coordination layer for a household.

Extension four is Meal Planning. Intermediate. The most important extension structurally, because it introduces two new primitives in one go. Row Level Security, which is a Postgres feature that scopes data access by user. And Shared M C P Server, the pattern that lets you give your spouse or your kids scoped access to specific parts of your brain without giving them everything. By the end your meal planner knows what is in the fridge, who is home, what people like, and can negotiate a grocery list across multiple humans.

Extension five is the Professional C R M, where C R M stands for Customer Relationship Management. Intermediate. The contact list. The interaction log. The reminders to follow up. The signal that someone has gone quiet for too long. This is where your professional life enters the brain, and it shares context with everything you have already captured.

Extension six is the Job Hunt Pipeline. Advanced. Applications, interviews, status changes, and follow ups. This is the capstone that ties everything together. Your C R M contacts become your network. Your network surfaces during job hunt research. Your calendar coordinates interviews. By the end of extension six, you have a system that genuinely operates across the major surfaces of an adult life.

The big idea of the learning path is compounding. Each extension teaches you something new, but it also wires into the previous extensions. Your C R M knows about thoughts you have captured. Your meal planner checks who is home. Your job hunt contacts automatically become professional network contacts. That cross referencing is what makes Open Brain feel different from a normal app.

---

## Section Five. Primitives, the Pieces That Repeat

Primitives are short concept guides. There are five of them today. Deploy an Edge Function teaches the general pattern that every extension uses. Remote M C P Connection teaches the custom connector pattern across clients. Common Troubleshooting collects the most frequent issues. Row Level Security is used by extensions four through six. Shared M C P Server is used by extension four.

The rule for adding a new primitive is strict. It has to be referenced by at least two extensions to justify extraction. This keeps the folder small and high signal. You also do not need to read primitives in advance. Each extension tells you when to read which primitive.

---

## Section Six. Community Recipes and Skills

This is where the repo gets big and varied. As of this recording, the recipes folder has more than forty entries.

On the data import side, you can pull in your ChatGPT history, your Perplexity searches, your Obsidian vault, your X export, your Instagram, your full Google Takeout including search, Maps, YouTube, and Chrome history, your Grok conversations, your Blogger archives, and your Gmail. Almost any platform that gives you an export, somebody in the community has written a recipe for. The pattern is always the same. Parse, dedupe, embed, and write into the thoughts table with rich metadata.

On the tools and workflows side, there are recipes that operate on the data you already captured. Auto Capture stores session summaries at the close of an A I session. Panning for Gold mines brain dumps and voice transcripts for actionable ideas. Aiception, formerly Claudeception, creates new skills from work sessions, skills that go on to create other skills. Schema Aware Routing distributes unstructured text across multiple tables. Daily Digest sends summaries by email or Slack. Life Engine is a self improving personal assistant that delivers proactive briefings through Telegram or Discord.

Skills are the lighter cousin of recipes. A skill is a plain text prompt pack you drop into Claude Code, Codex, Cursor, or another client. There are skills for competitive analysis, financial model review, deal memo drafting, research synthesis, meeting synthesis, panning for gold, and several more. If a behavior is reusable, its canonical home is the skills folder, and recipes are expected to depend on it through a metadata field called requires skills. That keeps the same prompt from getting copy pasted into a dozen places.

Schemas are simpler. Each schema adds tables and indexes alongside the core thoughts table. There are schemas for Agent Memory, Enhanced Thoughts, Entity Extraction, Provenance Chains, Typed Reasoning Edges, and Workflow Status. None of them modify the core thoughts table. That rule is hard. Adding columns is fine. Altering or dropping columns is forbidden by the automated review.

Integrations are about new connections. Discord Capture mirrors the Slack pattern for Discord. Kubernetes Deployment lets you self host the whole stack on your own cluster without using Supabase at all. The Agent Memory A P I exposes a runtime neutral memory layer. Open Brain R E S T exposes the same data over a standard H T T P interface.

Dashboards close out the open contributions. A SvelteKit dashboard, a Next dot js dashboard, and a pro version of the Next dot js dashboard with additional features like smart ingest and quality auditing. You host whichever you prefer on Vercel or Netlify and point it at your Supabase project.

---

## Section Seven. Agent Memory and the OpenClaw Direction

A newer thread inside the repo is the work around Agent Memory, also branded as OB1 Agent Memory. The guardrails are spelled out in the agents file at the root.

Keep the Agent Memory layer runtime neutral. OpenClaw is the flagship launch runtime, not the product boundary. Treat inferred or generated memory as evidence by default. Instruction grade memory, the kind that tells an agent how to behave, requires either human confirmation or trusted import. Avoid storing raw transcripts, model reasoning traces, secrets, and large code blocks by default. Prefer diagram first documentation, in the order of diagram, short explanation, copy paste setup, then deeper reference.

This shows up in the recent merged contributions. The Agent Memory schema sidecar adds tables for provenance, review, use policy, source reference, relation, recall trace, and audit. The Agent Memory A P I integration is the runtime neutral surface. The OpenClaw Agent Memory integration packages that A P I as a publishable OpenClaw plugin. If you are evaluating Open Brain as something more than a personal knowledge base, this is the thread to watch. The agent memory work is what turns Open Brain from a personal compounding system into something a team of agents can share.

---

## Section Eight. How to Contribute

Open the fourth diagram, named oh four contribution flow. It shows the path from idea to merged pull request, including the path for people who do not want to write code themselves.

The flow starts with you having an idea. The first decision point is whether you want to code it yourself or not.

If you do not want to code it, the repo has a non technical contribution path. You open an issue using the non technical contribution template. You describe the workflow in plain language. A community mentor picks it up, works with you to shape it, writes the code, and co-authors the pull request with you. You get full credit. Your name goes into the author field of the metadata file and into the contributors file. This is a first class path, not a workaround. Some of the best contributions come from people who use Open Brain daily but do not code.

If you do want to code it, the steps are these. Build it against your own Open Brain instance first. Do not submit something you have not run yourself. Pick the right category folder. Add a readme and a metadata file. No secrets. No binaries over one megabyte. No destructive S Q L like drop table or unqualified delete from. No modifications to the core thoughts table structure. Open a pull request titled in brackets with the category name, followed by a short description.

When the pull request opens, an automated review agent runs. It checks folder structure, validates the metadata schema, scans for secrets, checks S Q L safety, confirms required readme sections, validates internal links, and verifies that the remote M C P pattern is followed. If the agent finds problems, you address them and push fixes. If everything passes, a human admin reviews for quality and clarity. Expect two to five business days. When it merges, your name lands in the contributors file.

There is also a contributor ladder. You start as a community member just by showing up. You become a contributor with your first merged pull request, code or non code. You become a regular at three or more merged contributions. You become a maintainer by invitation, based on sustained quality involvement.

---

## Section Nine. Closing Thoughts

That is the tour. In one breath. Open Brain is a personal knowledge database with vector search, exposed through an open protocol, that any A I client can read from and write to. The repository contains the curated learning path that teaches you how to build it, the primitives that explain the patterns that repeat, and a growing collection of community recipes, schemas, dashboards, integrations, and skills.

The license is F S L one point one M I T, a functional source license that flips to plain M I T after two years. No commercial derivative works in the meantime. The code is open. The data is yours. The vendors are interchangeable. That is the entire philosophy.

If you want to get started, the four landing places are the setup guide, the A I assisted setup guide for people who would rather point Cursor or Claude Code at the repo, the companion prompts document with five prompts to help you build the capture habit, and the F A Q for the questions that come up most often. All four live in the docs folder.

If you hit a wall, there is a Discord for real time help and three dedicated A I assistants, a Claude skill, a custom GPT, and a Gemini gem, each one trained on the contents of this repository. Use whichever assistant matches the A I tool you already use.

You now have enough context to open the repository and know what you are looking at. Pick an extension if you want a guided build. Pick a recipe if you want a specific capability. Pick a skill if you just want to install a smart behavior into an A I client you already use. Pick a dashboard if you want a nice surface on top of your data.

Welcome to Open Brain. Happy building.

---

## Listening Notes

This script runs about three thousand two hundred words. At Speechify's default reading speed on iPhone, around one hundred sixty five words per minute, it is roughly twenty minutes of audio. At one point two five times speed, around sixteen minutes. At one point five times, around thirteen minutes.

The four diagrams live in the diagrams folder next to this file. They are written in Mermaid syntax with the file extension dot M M D. To view them, open them in GitHub or any Markdown viewer with Mermaid support, or paste their contents into the Mermaid live editor.

Diagram one. System architecture. The end to end flow from capture sources to A I clients.

Diagram two. Repo topology. What lives at the root of the repository.

Diagram three. Extension learning path. The six extensions in order with the primitives each introduces.

Diagram four. Contribution flow. From an idea to a merged pull request.
