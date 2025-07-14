---
layout: post
title: "Weekend Hack: Combining ChatGPT, Claude & More to Build TaleShade"
description: ""
category: blog
blog: true
tags: ["ai", "claude", "chaptgpt"]
lang: en
---

Decided to try new AI tools to see how much they'd help take a project from idea to deployment. Hacked together a project in a couple days: [TaleShade](https://taleshade.com) - every day a new short story is created in genres like Horror, Mystery, Sci-Fi, Thriller etc, and at midnight the story disappears. Like a bedtime story for reading/listening.

## 1. The Idea.
Used ChatGPT, which has custom GPTs for everything. Took one of the startup checkers and threw my idea at it. It sketched out the basic functionality. I asked it to create prompts for each page of the site.

## 2. Design.
With the ChatGPT-generated prompts, I went to design creation tools.
Tried:
- Stitch by Google
- heroui.chat
- 21st.dev
The first two mostly generated garbage that vaguely resembled what I needed, but 21st.dev produced something decent + generates code (React) right away if you need it. So AI-powered design and markup creation definitely has potential.

## 3. Code.
For code generation I used Claude Code. Fed it the same prompts from ChatGPT to create the foundation, then went through each page and dove into the functionality. Ran claude with dangerous mode: claude --dangerously-skip-permissions so it wouldn't ask permission for actions, worked out well, didn't delete anything :). Only made a couple manual edits.
About story generation specifically. Of course AI quality isn't perfect, but it can do something, seems like you can achieve acceptable quality with a good model and prompt. So, gpt-4o generates the text itself, tts-1-hd does the voiceover, and at one stage images for the story were generated using dall-e-3.

## 4. Deployment.
Got a $15 server for this. Been wanting to set up Kubernetes on a single machine for a while. For experiments and hosting pet projects.
Made an Ansible playbook. Does basic server setup, installs tailscale, k3s, cloudflare tunnel.
The project deployment itself is done with Helm. The app builds and deploys via Github Actions.
Claude Code helped with all this too, but naturally everything had to be reviewed and edited manually. Deployment actually took about the same time as development, but now I can easily deploy other stuff to the cluster.

Final stack of services and technologies:
- ChatGPT (gpt-4o, tts-1-hd, dall-e-3)
- 21st.dev
- Claude (opus and sonnet)
- Rails
- Hotwire
- TailwindCSS
- S3
- Kubernetes (k3s)
- Ansible
- Helm
- Tailscale
- Cloudflare
- Github Actions

By the way, didn't use MCP for Claude (though it's connected), regular CLI commands were enough. MCP is a bit overhyped.

For now, this approach of using different tools and combining their strengths shows better results than using just one tool that tries to do everything.

A year and a half ago we started seriously using ChatGPT, now a small project from idea to deployment can be done in a couple days. What's coming in another year?
