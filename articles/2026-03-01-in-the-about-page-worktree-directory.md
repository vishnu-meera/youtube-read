## Unleash Parallel AI Development: Mastering Git Worktrees with Claude Code

Imagine this: you're knee-deep in a project, and two urgent features land on your plate. Traditionally, you'd finish one, merge it, then start the next. But what if you could tackle them **simultaneously**? With the rise of AI coding agents like Claude Code, this scenario becomes even more compelling. However, running multiple agents on the same codebase at once is a recipe for conflict and chaos.

That's where **Git Worktrees** come in, and Claude Code's recent built-in support for them is a game-changer. It allows you to transform your AI coding workflow from a sequential bottleneck into a parallel powerhouse, minimizing conflicts and maximizing productivity.

### The Conflict Cauldron: Why Parallel Agents Fail on a Single Codebase

Let's illustrate the problem. Suppose you have a Laravel project and want to add two new pages: an "About Us" page and a "Contact Us" page. Both require new routes and corresponding menu items in the sidebar.

If you were to launch two separate Claude Code instances (or even work manually) directly on your main branch, you'd quickly hit a wall. Both processes would attempt to modify crucial shared files like `routes/web.php` (for defining new routes) and `resources/views/layouts/app/sidebar.blade.php` (for adding menu links).

```php
// routes/web.php (hypothetical conflict point)
Route::middleware(['auth', 'verified'])->group(function () {
    Route::view('dashboard', 'dashboard')->name('dashboard');
    // Agent 1 adds 'about' route here
    // Agent 2 adds 'contact' route here
});
```

These simultaneous changes would lead to inevitable **merge conflicts**, forcing you to stop, manually resolve them, and interrupt your flow. This is precisely the kind of friction Git Worktrees are designed to eliminate in the context of parallel AI-assisted development.

### Your Isolated Sandbox: Understanding Git Worktrees

A **Git Worktree** is essentially a separate working directory linked to the same underlying Git repository. Think of it like a lightweight, temporary clone of your project, allowing you to have multiple branches or commits checked out *simultaneously* in different directories. Crucially, each worktree operates in its own isolated environment, preventing direct interference with other worktrees or your main project.

It's similar to working on different Git branches, but with a key distinction:
*   **Branches** switch your entire working directory to a new state within the same folder.
*   **Worktrees** create entirely new, independent working directories, each potentially on a different branch or commit. This physical separation is what provides true isolation.    

While Git Worktrees are not a new Git concept, Claude Code has now integrated built-in support, making it incredibly convenient for orchestrating multiple AI agents.

### Setting Up Your AI Agent Workspaces

To leverage worktrees with Claude Code, you simply use the `--worktree` flag when launching your agent.

**1. Create the "About Us" Worktree:**

First, let's create a worktree for the "About Us" page. This command will spin up a Claude Code agent in a dedicated worktree named `about-page`.

```bash
claude --dangerously-skip-permissions --worktree about-page
```

_Note: `--dangerously-skip-permissions` is used here for demonstration speed; omit in production unless you understand the security implications._

Upon execution, Claude Code will create a new directory: `~/Herd/worktrees/.claude/worktrees/about-page` (the exact path might vary based on your project structure). This new directory is a full, isolated copy of your entire Laravel project. Any changes made by this Claude agent will only affect files within this `about-page` worktree, not your main project directory.

**2. Create the "Contact Us" Worktree:**

Now, in a separate terminal tab, launch another Claude Code agent for the "Contact Us" page, creating its own isolated worktree.

```bash
claude --dangerously-skip-permissions --worktree contact-page
```

This creates another isolated project copy at `~/Herd/worktrees/.claude/worktrees/contact-page`. You now have two AI agents working in parallel, each in their own clean, isolated environment.

### Witnessing Parallelism (and Avoiding Immediate Conflicts)

With our two agents humming along in their respective worktrees, let's give them their tasks:

**Agent 1 (About Us worktree):**
Prompt: `Add "About us" page to the sidebar menu with dummy text in Blade view, using Route::view`

**Agent 2 (Contact Us worktree):**
Prompt: `Add "Contact us" page to the sidebar menu with dummy text in Blade view, using Route::view`

As each agent works, they will modify files such as `routes/web.php` to add their respective routes and `resources/views/layouts/app/sidebar.blade.php` to add their menu items. Because they are operating in entirely separate working directories (the worktrees), their changes **do not immediately conflict**. Each agent sees only its own project copy, blissfully unaware of the other's modifications. This isolation is the core benefit, allowing parallel development without constant interruption.

After both agents complete their tasks and pass their tests within their respective worktrees, you'll have two sets of changes ready to be integrated.

### The Merge Gauntlet: When Worlds Collide (Gracefully)

While worktrees prevent *live* conflicts, the changes still need to be merged back into your main branch. This is where conflicts might arise, but now you handle them in a controlled, deliberate manner after the AI agents have done their individual work.

**1. Commit Worktree Changes:**

First, commit the changes within each worktree. Each worktree effectively acts as its own branch (e.g., `worktree-about-page` and `worktree-contact-page`).

For the `about-page` worktree, you'd commit its changes:

```bash
# In the about-page worktree directory
git add .
git commit -m "feat: Add about page"
```

Similarly, for the `contact-page` worktree:

```bash
# In the contact-page worktree directory
git add .
git commit -m "feat: Add contact page"
```

**2. Merging into Main:**

Now, navigate back to your main project directory (the parent of the `.claude/worktrees` folder) and switch to your `main` branch.

Let's merge the "About Us" worktree first:

```bash
# In the main project directory
git merge worktree-about-page
```

If these changes are unique and don't overlap with existing `main` content, Git will perform a "fast-forward" merge with no conflicts.

Next, attempt to merge the "Contact Us" worktree:

```bash
# In the main project directory
git merge worktree-contact-page
```

This is where you'll likely encounter conflicts. Git will report conflicts in files both worktrees modified, such as `package-lock.json`, `resources/views/layouts/app/sidebar.blade.php`, and `routes/web.php`.

**3. Resolving Conflicts Manually:**

Git will mark the conflicting files. You can open these files in your code editor (like VS Code) which provides tools to visualize and resolve conflicts.

For `sidebar.blade.php`, you'll see both the "About Us" and "Contact Us" menu items. You simply accept both incoming changes, carefully placing them in the desired order.

```html
<!-- resources/views/layouts/app/sidebar.blade.php -->
<x-flux::sidebar.item icon="home" href="{{ route('dashboard') }}" :current="request()->routeIs('dashboard')">
    {{ __('Dashboard') }}
</x-flux::sidebar.item>
<x-flux::sidebar.item icon="information-circle" href="{{ route('about') }}" :current="request()->routeIs('about')">
    {{ __('About Us') }}
</x-flux::sidebar.item> <!-- Added from about-page worktree -->
<x-flux::sidebar.item icon="envelope" href="{{ route('contact') }}" :current="request()->routeIs('contact')">
    {{ __('Contact Us') }}
</x-flux::sidebar.item> <!-- Added from contact-page worktree -->
```

Similarly, for `routes/web.php`, you'll combine the new routes:

```php
// routes/web.php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::view('dashboard', 'dashboard')->name('dashboard');
    Route::view('about', 'about')->name('about');     // Added from about-page worktree
    Route::view('contact', 'contact')->name('contact'); // Added from contact-page worktree
});
```

For `package-lock.json`, running `npm install` (if it's a JS/TS project) usually resolves the changes correctly by updating dependencies.

After resolving all conflicts:

```bash
# In the main project directory, after manual resolution
git add routes/web.php resources/views/layouts/app/sidebar.blade.php package-lock.json
git commit -m "Merge about and contact pages"
```

Your main branch now contains the combined, conflict-resolved changes from both worktrees.

**4. Cleaning Up Worktrees:**

Once merged, you can delete the temporary worktrees. You can do this via your IDE (VS Code has a "Delete Worktree" option in the Source Control view) or using Git CLI:

```bash
git worktree remove about-page
git worktree remove contact-page
```

This removes the isolated directories, keeping your main project directory clean.

### Beyond the Basics: Considerations and Advanced Use Cases

*   **Tool Compatibility:** Be aware that some development tools (e.g., formatters like PHP Pint, dependency managers) might not be configured to run correctly within a worktree's nested directory structure. You might need to adjust their paths or commands to ensure they work as expected.
*   **Subagents:** For complex tasks, Claude Code supports subagents that can *also* leverage worktree isolation. This is incredibly powerful for large-scale batched changes or code migrations, where a single master agent orchestrates many subagents, each working in its own isolated environment.
*   **Non-Git Source Control:** Worktrees are inherently a Git feature. While the video touches on hooks for other SCMs, the primary benefit of built-in worktree support in Claude Code currently shines brightest within a Git workflow.

### The Power of Isolated Parallelism

Git Worktrees, especially when combined with AI coding agents like Claude Code, offer a powerful way to manage complex development workflows. They allow multiple agents (or developers) to work on different features in parallel, each in their own clean, isolated sandbox. This dramatically reduces the headache of constant context switching and immediate conflicts, streamlining the development process. The conflicts, when they do arise, are handled at a later, more controlled stage, making integration smoother and more predictable.

This paradigm shift enables developers to leverage AI agents more effectively, pushing the boundaries of what's possible in parallel code generation and large-scale refactoring. 