---
layout: post
title:  "Zed, Claude, and a Docker sandbox to put it in"
date:   2026-10-19 10:00:00
categories: zed claude agents docker sandboxes
---

I have two ways of running a coding agent on my laptop and I don't like either of them. I can approve every action, which means spending the afternoon clicking "allow" on `ls` and, somewhere around the fortieth prompt, no longer reading what I'm allowing. Or I can turn the prompts off and give a fast, keen, occasionally confused process the run of my home directory and whatever credentials happen to be lying around in it.

What I actually want is to turn the prompts off somewhere it doesn't matter. For me that's a Docker sandbox. The place I want to talk to the agent from is Zed. Joining the two took one settings block, and I think the block is worth walking through, because there's more going on in that one line of shell than it looks.

## The three pieces

Zed has an agent panel, and it's my favourite place to work with a coding agent. Its edits show up as diffs in the editor I'm already using, and I can click straight through to the files it mentions.

The panel talks to agents over ACP, the Agent Client Protocol. Zed starts the agent as a subprocess and they exchange JSON-RPC over stdin and stdout. Claude Code doesn't speak ACP itself, so there's an adapter, `@agentclientprotocol/claude-agent-acp`, that sits in front of it and translates.

`sbx` is Docker's CLI for sandboxes. Each one is a small VM with your project directory mounted in and outbound traffic going through a proxy that enforces a network policy. Nothing else from your machine is visible unless you put it there.

The thing that makes this work is that ACP only cares about stdin and stdout. It has no idea where the process is running. If the command Zed launches is "run the adapter inside a sandbox and pipe its stdio back out", Zed can't tell the difference, and doesn't need to.

## The whole config

This goes under `agent_servers` in Zed's `settings.json`:

```json
"agent_servers": {
  "Claude (sandbox)": {
    "default_config_options": {
      "effort": "default",
      "fast": false,
    },
    "type": "custom",
    "command": "/bin/zsh",
    "args": [
      "-lc",
      "exec sbx exec -i \"claude-$(basename \"$PWD\")\" -- npx -y @agentclientprotocol/claude-agent-acp",
    ],
    "env": {},
  },
},
```

It turns up in the agent panel as its own entry, alongside the built-in ones.

<!-- TODO: screenshot of Zed's agent panel with "Claude (sandbox)" selected -->
<!-- ![Zed's agent panel with the "Claude (sandbox)" agent selected](/assets/img/zed-claude-sandbox.png) -->

The sandbox has to exist before Zed can use it. You create it once per repository, from the repository root:

```bash
sbx create claude .
```

By default a sandbox is named after the agent and the directory, so in a repo called `jekyll-klise` you get `claude-jekyll-klise`. Remember that, because it's the name the config goes looking for.

## Reading the command

`/bin/zsh -lc` runs the rest in a login shell. Zed starts agent servers as plain subprocesses, and depending on how Zed itself was launched, that subprocess might not have the `PATH` my terminal has. A login shell reads my profile, so `sbx` gets found either way.

`exec` replaces the shell with `sbx` instead of running it as a child. When Zed stops the agent, the signal goes to `sbx` directly rather than to a shell that may or may not pass it on.

`claude-$(basename "$PWD")` is the sandbox name. Zed runs the command from the project root, so `basename "$PWD"` is the repo's folder name, and sticking `claude-` on the front gets you back to the default name from earlier. One sandbox per repo, found by convention, and nothing per-project in Zed's settings. Open a different repo and the same entry talks to a different sandbox. If that sandbox is stopped, `sbx exec` starts it first.

`-i` keeps stdin open. ACP is a conversation over stdin and stdout, and without `-i` the adapter would see its input close straight away and exit. There's deliberately no `-t`: a TTY in the middle would start mangling the JSON-RPC.

`npx -y @agentclientprotocol/claude-agent-acp` is everything after the `--`, and it runs inside the sandbox. It's the adapter, and it starts Claude Code in there, where it can do what it likes.

## Why the paths line up

I fully expected this part to break.

ACP messages are full of absolute paths. Zed tells the agent which files are open, the agent tells Zed which files it changed, and Zed draws diffs and links against those paths. If the sandbox mounted the project at `/workspace` while Zed thought it lived at `/Users/me/repos/thing`, every one of those paths would point at nothing.

It doesn't break, because `sbx` mounts the workspace at the same absolute path inside the sandbox as on the host. My Mac's `/Users/...` path exists, character for character, inside a Linux VM. Paths Zed sends resolve in the sandbox, paths the agent sends back resolve on the Mac, and nobody has to translate anything. Diffs end up in the right buffer.

## What the sandbox does and doesn't protect

The agent can see the mounted repository and whatever the network policy lets it reach. It can't see the rest of my home directory, my host credentials or my SSH keys. It can still make a complete mess of the repository, since that's the one thing I deliberately handed over. Git is the undo button, so I commit before I let it loose.

## Where it's rough

A few things about this setup annoy me.

`npx -y` checks for the adapter every time the agent starts, and downloads it the first time. That adds a bit of startup time, and it means the sandbox's network policy has to allow the npm registry.

If you forget `sbx create`, the agent panel just fails, and the error doesn't tell you the sandbox is missing.

Naming by folder is only a convention. Two repos that happen to share a folder name will end up in the same sandbox.

And `"type": "custom"` means you're on your own. Zed's built-in Claude entry looks after the adapter for you. This one is a command line you own, and if the adapter or `sbx` changes underneath it, fixing it is your job.

None of this is specific to Claude, either. Any agent that speaks ACP can go in the same sandbox if you swap out what comes after the `--`.

Zed gets the conversation, the sandbox gets the consequences, and one command line joins them.
