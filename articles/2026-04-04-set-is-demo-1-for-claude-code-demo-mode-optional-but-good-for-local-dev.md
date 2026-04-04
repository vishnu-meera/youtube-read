It's one thing to hear about Large Language Models (LLMs) writing code, but it's another to see a full-fledged web application materialize from thin air – or rather, from a carefully crafted prompt and a local LLM running right on your machine. This isn't just about generating snippets; it's about an autonomous coding agent building a functional .NET 8 MVC application, complete with controllers, models, and views, all without touching a single cloud API or incurring any external costs.

This guide will walk you through setting up a local LLM environment using **Ollama** and the **Claude Code** agent to achieve precisely that. We'll leverage a powerful **Qwen 3.5 9B** model, showcasing how consumer-grade hardware can become your personal, free code factory.

## The Local LLM Frontier: Power in Your Hands

Most LLM tutorials focus on cloud-based APIs, which are convenient but come with usage costs and privacy implications. What's often overlooked is the burgeoning ecosystem of local LLMs. Running an LLM locally offers several compelling advantages: **zero API costs**, enhanced **privacy**, and the ability to **fine-tune** and **experiment** without constraints.

Our experiment involves generating a simple, yet realistic, issue-tracking system using a .NET 8 MVC framework. The impressive part? It's entirely driven by a local LLM.

## Setting Up Your Autonomous Coding Environment

To get started, we need two core components:

1.  **Ollama**: A fantastic tool that makes it incredibly easy to download and run large language models locally.
2.  **Claude Code**: An autonomous coding agent that orchestrates the code generation process based on your instructions.

The hardware setup used for this demonstration is a PC equipped with an **Nvidia RTX 3060 Ti** graphics card, featuring **16GB of VRAM**. This is crucial because, for larger models and longer context windows, the GPU accelerates processing significantly.

### Launching Claude Code with a Local Qwen Model

First, ensure you have Ollama installed and the `qwen3.5-9b` model downloaded. If not, you can get it with `ollama pull qwen:3.5-9b`.

To launch Claude Code and instruct it to use our local Qwen 3.5 model, we'll use a specific command. This command also sets a generous context length and activates a "demo mode" for Claude Code, which prevents it from asking for email or similar details.

```bash
# Set IS_DEMO=1 for Claude Code demo mode (optional, but good for local dev)
SET IS_DEMO=1

# Launch Claude Code using Ollama with the Qwen 3.5 9B model
# and a context length of 128,000 tokens.
ollama launch claude --model qwen3.5-120b:9b -c 128k
```

This command kicks off Claude Code, telling it to use the locally available `qwen3.5-120b:9b` model (a variant of Qwen 3.5 with 9 billion parameters) and allocating a substantial **128k context window**. This larger context is vital for the LLM to comprehend and manage the entire application's design specification effectively.

When you run this, Claude Code will launch, confirming the model and context it's using:

```
Launching Claude Code with qwen3.5-120b:9b...
...
T Opus now defaults to 1M context - be more room, some pricing
```
*(Note: The "Opus" line in Claude Code's output is likely a default message from the Claude Code tool itself, indicating its fallback to a larger context if an external model wasn't specified. In our case, `--model qwen3.5-120b:9b` explicitly overrides it.)*

You can verify the current model within the Claude Code interface by typing `/model`.

## The Blueprint: Detailed Design Specifications

The key to successful code generation with LLMs lies in providing clear, comprehensive instructions. For this project, two files were created in the working directory: `claude.md` for the detailed design specification and `prompt.txt` for the initial high-level instruction. Claude Code automatically detects these files.

### `claude.md`: The Application's DNA

This Markdown file serves as the core design specification for the issue tracker. It outlines everything from functional requirements to architectural constraints, guiding the LLM in its code generation process.

```markdown
# Simple Issue Tracking System - Design Specification

## Purpose
Create a small but realistic .NET 8 MVC web application using Razor Views, C#, and plain JavaScript. The app should allow users to manage issues.

## Functional Requirements

### Issue Fields
Each issue must contain:
- **ID** (string)
- **Title** (string)
- **Description** (string)
- **Priority** (Low, Medium, High)
- **Status** (Open, Closed)
- **CreatedAt** (DateTime)

### Features
- List all issues
- Create a new issue
- Edit an existing issue
- Close or reopen an issue (toggle status)
- Filter by status and priority
- Optional: small dashboard summary (open vs closed count)

## Architecture Requirements

### Project Type
- .NET 8
- MVC (Controllers + Views)
- Razor Views for UI
- No SPA frameworks (no React, Vue, Angular)
- Plain JavaScript allowed
- Optional JS libraries allowed (Chart.js, Sortable.js, etc.)

### Controllers
Create a controller named `IssuesController` with actions:
- `Index()`: GET - list issues
- `Create()`: GET - show form
- `Create(Issue)`: POST - save new issue
- `Edit(ID)`: GET - show form
- `Edit(Issue)`: POST - update issue
- `ToggleStatus(ID)`: POST - open/close issue

### Models
Create a model class `Issue` with the fields listed above.

### Views
Create Razor Views:
- `Views/Issues/Index.cshtml`
- `Views/Issues/Create.cshtml`
- `Views/Issues/Edit.cshtml`
- `Views/Issues/Detail.cshtml` (for individual issue view, perhaps linked from index)

Views must:
- Use Bootstrap for layout (CDN is fine)
- Use partials where appropriate
- Use layout page `_Layout.cshtml`

### Persistence Requirements

#### Storage Method
- Use a JSON file stored in: `/data/issues.json`

#### Behavior
- The controller reads the JSON file on each request.
- The controller writes back to the JSON file after modifications.
- No database is used.
- No special permissions required (Visual Studio dev environment)

#### JSON Format
The file contains an array of `Issue` objects.

### JavaScript Requirements
- Place JS in: `/wwwroot/js/issues.js`
```

### `prompt.txt`: The Initial Command

The `prompt.txt` file provides the overarching instruction for the coding agent. It tells Claude Code what its primary objective is:

```
You are an autonomous coding agent. Your task is to generate a complete .NET 8 MVC web application from this design specification.

Follow these instructions exactly:
1. Read the entire design specification.
2. Create a full Visual Studio solution named 'IssueTracker'.
3. Generate all folders and files exactly as described.
4. For each file, output a code block containing the full file contents.
5. Do not skip any files.
6. Do not combine partials or client-side code blocks.
7. Do not combine unrelated files into a single code block.
8. Do not add commentary outside code blocks.
9. Ensure the project compiles without modification.
10. Ensure JSON persistence works without modification.
11. Ensure all controllers, models, views, and JS files are included.
12. Ensure the Layout page and Bootstrap are correctly referenced.
13. Ensure the solution uses Razor Views, not Blazor.
14. Ensure the solution uses plain JavaScript, not a SPA framework.

When outputting the solution:
- Start with the '.sln' file.
- Then output each project file in a logical order.
- Follow heading like:
  - `## File: Controllers/IssuesController.cs`
- Each file must be in its own fenced code block.

Your final output must be a complete, ready-to-open Visual Studio solution.

Begin now unless you have additional questions.
```

## The AI in Action: Building the Application

With the environment set up and the detailed specifications in place, the Claude Code agent takes over. The speaker notes that while the advanced cloud models like Opus might generate a perfect build on the first try, a local model like Qwen 3.5 9B might require a few iterations to get everything just right. This highlights the iterative nature of working with even advanced AI tools.

However, after a couple of refinements, the local LLM successfully generated a complete, functional .NET 8 MVC application.

### The Generated Application

The output is a full Visual Studio solution named `IssueTracker`. The solution explorer view reveals the familiar structure of a .NET MVC application:

```
IssueTracker (Solution)
└── IssueTracker (Project)
    ├── Controllers
    │   └── IssuesController.cs
    ├── Data
    │   └── issues.json (for persistence)
    ├── Models
    │   └── Issue.cs
    ├── Views
    │   ├── Issues
    │   │   ├── Create.cshtml
    │   │   ├── Edit.cshtml
    │   │   ├── Index.cshtml
    │   │   └── Detail.cshtml
    │   └── Shared
    │       └── _Layout.cshtml
    ├── wwwroot
    │   └── js
    │       └── issues.js
    └── ... other standard .NET MVC files
```

The application, while not "fancy," is fully functional. It manages issues, allowing users to create, edit, list, and toggle the status of issues. All data is persisted to a local `issues.json` file as specified, demonstrating adherence to the architectural requirements.

## Why This Matters: The Power of Local LLMs

This demonstration is a powerful testament to the capabilities of local LLMs. It shows that you don't always need expensive cloud APIs to leverage advanced AI for code generation.

*   **Cost-Effective:** By running the model on your local workstation, you eliminate API billing costs, making AI-powered development accessible to everyone.
*   **Privacy-Focused:** Your code and data never leave your machine, addressing concerns about intellectual property and data security.
*   **Rapid Prototyping:** The ability to quickly generate significant portions of an application from a design specification accelerates the development workflow.
*   **Iterative Development:** While it may require a few iterations, the local LLM provides a strong starting point and can be guided to refine its output, making it a valuable co-pilot for developers.

This isn't just a glimpse into the future; it's a practical, present-day capability. Developers can set up their own autonomous coding agents, provide detailed instructions, and watch as full applications are built right on their desk, all powered by free, locally hosted models. The era of personalized, private, and powerful AI development is here.