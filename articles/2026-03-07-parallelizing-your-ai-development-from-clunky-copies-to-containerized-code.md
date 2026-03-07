# Parallelizing Your AI Development: From Clunky Copies to Containerized Code

Imagine a world where you could simultaneously explore three distinct redesigns for your website – a cyberpunk fantasy, a vibrant Barbie aesthetic, and a minimalist black-and-white theme – all while your AI coding agents work in perfect harmony, never stepping on each other's toes. This isn't a sci-fi dream; it's the reality of **parallel agents** in generative coding, and it's set to revolutionize how we build software.

The challenge in modern development, especially with the rise of AI-powered coding assistants, isn't just about generating code; it's about managing multiple, concurrent development efforts within a single codebase. How do you let several agents (or developers) experiment with different features or design iterations without creating a chaotic merge nightmare or bloating your local environment?

Let's explore three evolving workflows, each offering increasing levels of sophistication and efficiency, to tackle this parallel development problem. While our examples will use Claude Code, these principles apply to virtually any coding agent or collaborative environment.

## The Brute Force: Copying Folders Like It's 1999

The simplest, albeit most primitive, way to achieve parallel development is to **literally copy your project folder**. Need to try a new feature? Copy the whole directory. Want to experiment with a different design? Copy it again.

Here's how that looks:

```bash
# Our original project
ls
# galea.dev-react

# Create two copies for our AI agents
cp -r galea.dev-react agent-a
cp -r galea.dev-react agent-b

ls
# agent-a agent-b galea.dev-react
```

You now have three identical, independent projects. You can navigate into `agent-a` and have your AI agent work on one feature, and in `agent-b` for another.

### The Immediate Headaches

Sounds easy enough, right? Not so fast. The moment you try to run your copied project, you'll likely hit a snag. If your project has dependencies (like `node_modules` in a JavaScript project), copying the folder often breaks the internal links or paths required for execution.

```bash
cd agent-a
npm run dev
# Error: ERR_MODULE_NOT_FOUND Cannot find module 'vite/dist/node/cli.js'
```

To fix this, you're looking at manually reinstalling dependencies for each copied folder.

```bash
# Remove the broken node_modules and reinstall
rm -r node_modules && npm install
npm run dev
# VITE v5.4.10 ready in 319 ms
# Local: http://localhost:8080/
```

Now imagine managing two AI agents, Agent A and Agent B, each creating a different button animation effect. After they've done their work, you have two modified `HeroSection.tsx` files. If Agent A adds a subtle glow and Agent B implements a full cyberpunk glitch effect, you're left with a tedious manual merge. This approach quickly becomes unmanageable, wastes disk space, and introduces significant friction.

## A Better Way: Leveraging Git Worktrees

The "copy folders" method is clunky because it duplicates *everything*. What if you could have separate working directories that shared the core Git repository? That's exactly what **Git Worktrees** offer.

A Git Worktree allows you to maintain multiple working directories (and branches) for a single repository. It's like having several checkouts of your project, each on a different branch, but without the overhead of entirely cloning the repository each time.

```bash
# Start in our main project directory
cd galea.dev-react

# Create two worktrees, each on a new branch for our agents
git worktree add -b links-section-update/agent-a ../agent-a
git worktree add -b links-section-update/agent-b ../agent-b

ls ../ # Observe the new directories
# agent-a agent-b galea.dev-react

git branch # See the new branches alongside main
# + links-section-update/agent-a
# + links-section-update/agent-b
# * main
```

Instantly, two new directories (`agent-a`, `agent-b`) appear. Each points to a distinct Git branch, but they are lightweight and managed efficiently by Git.

### Lingering Friction

While `git worktree` is a massive improvement, it's not perfect. Like the "copy folders" method, each new worktree still needs its dependencies installed:

```bash
cd ../agent-a
npm install # Still needed to set up node_modules
```

Our AI agents (or human developers) can now work in their respective worktrees, each focused on a different feature, without directly interfering. When they're done, we can review the changes.

Let's say Agent A focused on improving Click-Through Rate (CTR) for the links section, and Agent B prioritized the aesthetic appeal. After they complete their tasks, you might have two distinct versions of your link section UI.

```bash
# Agent A's work in agent-a
# Agent B's work in agent-b

# After reviewing Agent A's changes, we might want to merge them to main:
cd galea.dev-react # Switch back to the main project
git merge links-section-update/agent-a
# Fast-forward merge successful!
```

This is cleaner than manual copying, but the process of managing environments and dependencies still involves manual steps. What if we could abstract away the environment setup entirely?

## The Future: Containerized Parallel Agents with Dagger's `container-use`

This is where things get truly exciting. Imagine environments that spin up instantly, are perfectly isolated, and come pre-configured with all their dependencies. No more `npm install` for every branch. This is the promise of **Dagger's `container-use`**.

Dagger, built by the creators of Docker, provides a powerful tool for defining your development, CI/CD, and deployment pipelines as code. Its `container-use` command specifically allows AI agents to work in isolated, ephemeral containers, offering an unparalleled developer experience for parallel efforts.

### Getting Started with `container-use`

First, you need to install `container-use`.

```bash
# For macOS with Homebrew:
brew install dagger/tap/container-use

# For other platforms (using curl):
curl -L https://raw.githubusercontent.com/dagger/container-use/main/install.sh | bash

# Verify installation
container-use version
# container-use version 0.8.4.2 ...
```

Next, integrate `container-use` with your AI agent (like Claude Code) by configuring it as an **MCP (Multi-Codebase Project) server**. This tells your agent how to interact with these containerized environments.

```bash
# In your project's root directory, configure Claude Code to use container-use
claude mcp add container-use -- container-use stdio
```

You can also define agent rules to restrict what tools your AI agent can use, enhancing security and predictability.

```bash
# Fetch and append Dagger's agent rules to your CLAUDE.md file
curl https://raw.githubusercontent.com/dagger/container-use/main/rules/agent.md >> CLAUDE.md
```

### Instant, Isolated Environments

Now, the magic begins. Instead of copying folders or creating worktrees, you create new *environments* for each parallel task.

```bash
# Create an environment for a Fantasy RPG theme redesign
container-use environment create \
  --title "Fantasy RPG Website Redesign" \
  --explanation "Redesign this website to have a fantasy RPG style theme" \
  --env-id safe-rhino

# Create an environment for a Barbie-themed website
container-use environment create \
  --title "Barbie-themed Website Redesign" \
  --explanation "Redesign this website to have a Barbie theme" \
  --env-id great-mudfish

# Create an environment for a super minimal design
container-use environment create \
  --title "Minimal Website Redesign" \
  --explanation "Redesign this website to have a super minimal feel" \
  --env-id quick-anteater
```

Each of these commands instantly creates a new, isolated environment in a Docker container (Docker needs to be running in the background). No manual `npm install` needed – the environment is ready to go. The changes made within each environment are tracked as distinct Git commits associated with that environment ID.

You can list your active environments:
```bash
container-use list
# ID             TITLE                        CREATED     UPDATED
# safe-rhino     Fantasy RPG Website Redesign 14 minutes ago 13 minutes ago
# quick-anteater Minimal Website Redesign     13 minutes ago 7 minutes ago
# great-mudfish  Barbie-themed Website Design 12 minutes ago 6 minutes ago
```

### Switching and Merging with Ease

Want to inspect a specific redesign? Simply check it out.

```bash
# Check out the Barbie theme locally
container-use checkout great-mudfish
```
This command instantly updates your local working directory to reflect the state of the `great-mudfish` environment. You can run `npm run dev` and immediately see the Barbie-themed website. All changes, including new assets and CSS, are brought in seamlessly.

Once you're happy with a version (say, the Barbie theme!), you can merge its changes into your main branch.

```bash
# In your main project directory
container-use merge great-mudfish
# Merge made by the 'ort' strategy.
# ... files changed, insertions(+), deletions(-)
# Environment 'great-mudfish' merged successfully.
```
This command handles the Git merge, bringing the containerized environment's changes directly into your main branch's history. It's clean, efficient, and tracks all changes with proper commit messages automatically generated by the agent.

Finally, you can clean up unused environments:
```bash
container-use delete quick-anteater
```
This removes the container and its associated worktree, keeping your local workspace tidy.

## The Power of True Parallelism

With `container-use`, each AI agent (or developer) can work in their own completely isolated, reproducible environment. They have full access to a consistent, pre-configured toolkit and codebase, without needing to worry about conflicting dependencies or polluting the main branch.

This workflow shines for several reasons:

*   **Isolation:** Each environment is a self-contained container, preventing conflicts between different features or experiments.
*   **Reproducibility:** Environments can be easily recreated, ensuring consistent results across different machines or time.
*   **Efficiency:** Spin up and tear down environments instantly, saving time on setup and dependency management.
*   **Streamlined Git Operations:** Dagger integrates directly with Git, automatically tracking changes and facilitating merges with rich commit histories.
*   **Scalability:** Run multiple AI agents or human developers in parallel without local environment constraints.

## The Road Ahead

The landscape of generative coding is evolving rapidly, and the ability to manage parallel development efforts effectively is becoming paramount. While manual copying and `git worktree` offer incremental improvements, Dagger's `container-use` represents a significant leap forward. By providing truly isolated, reproducible, and seamlessly managed environments, it empowers developers and AI agents alike to experiment, iterate, and innovate at unprecedented speeds.

This approach isn't just about faster code generation; it's about enabling a more fluid, creative, and less friction-filled development cycle. As AI takes on more complex coding tasks, the tools that facilitate parallel, robust development will be the ones that truly define the future of software engineering.