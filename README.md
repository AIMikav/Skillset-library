# Skillset Library

A collection of reusable Claude Code skills to streamline daily activities and development tasks.

## What are Claude Code Skills?

Skills are specialized tools in Claude Code that provide domain-specific capabilities. They can be invoked during conversations to help with specific tasks like productivity improvements, development workflows, and more.

## Repository Structure

```
skillset-library/
├── .claude/
│   └── .mcp.json                  # Kubernetes MCP server configuration
├── skills/
│   ├── productivity/              # Skills for task management, note-taking, etc.
│   │   └── meeting-notes-analyzer.md
│   ├── development/               # Skills for code setup, testing, documentation
│   │   └── github-setup.md
│   └── infrastructure/            # Skills for Kubernetes, OpenShift, policy management
│       ├── cluster-policy-analyzer/
│       │   ├── SKILL.md
│       │   └── README.md
│       └── k8s-best-practices-validator/
│           ├── SKILL.md
│           └── README.md
├── README.md
├── MCP_SETUP_GUIDE.md
├── TEAM_SHARING.md
└── .gitignore
```

## Installation

To use these skills with Claude Code:

1. Clone this repository:
   ```bash
   git clone https://github.com/YOUR_USERNAME/skillset-library.git
   ```

2. Copy the skill files to your Claude Code skills directory:
   ```bash
   cp -r skillset-library/skills/* ~/.claude/skills/
   ```

   Or create a symlink to keep them synced (recommended):
   ```bash
   ln -s /path/to/skillset-library/skills/* ~/.claude/skills/
   ```

3. The skills will be automatically available in your next Claude Code session

## Available Skills

### Development Skills

#### github-setup
**Purpose:** Automates GitHub repository setup by reading README instructions and setting up projects locally

**What it does:**
- Fetches and analyzes GitHub repository README files
- Extracts installation and setup steps automatically
- Clones the repository to your preferred location
- Installs dependencies (npm, pip, cargo, etc.)
- Sets up configuration files and environment variables
- Executes build commands
- Verifies the setup works

**Usage:**
```
Give Claude a GitHub URL and the skill will handle the entire setup process
Example: "Set up https://github.com/user/awesome-project"
```

**Key Features:**
- Supports multiple package managers (npm, yarn, pnpm, pip, cargo, etc.)
- Handles different project types (Node.js, Python, Rust, Go, etc.)
- Asks for confirmation before executing commands
- Provides clear setup summaries and next steps

---

### Productivity Skills

#### meeting-notes-analyzer
**Purpose:** Reads meeting notes from local files and extracts tasks assigned to you

**What it does:**
- Reads meeting notes from various formats (text, markdown, Gemini notes, etc.)
- Identifies tasks assigned to you by finding mentions of your name
- Detects action items in various formats (@name, "name will...", etc.)
- Creates structured todo lists with deadlines
- Groups tasks by priority
- Generates clean summaries of action items

**Usage:**
```
Point Claude to a meeting notes file to extract your action items
Example: "Analyze my meeting notes from ~/Documents/team-sync.txt"
```

**Key Features:**
- Recognizes multiple task assignment patterns
- Extracts deadlines and due dates
- Includes context so you remember what each task is about
- Creates actionable todo lists using Claude Code's TodoWrite tool
- Flags high-priority items with deadlines

---

### Infrastructure Skills

#### cluster-policy-analyzer
**Purpose:** Real-time discovery and analysis of Kubernetes/OpenShift clusters using the Kubernetes MCP server

**What it does:**
- Discovers all Custom Resource Definitions (CRDs) in any cluster
- Analyzes Gatekeeper/OPA policy configurations
- Generates Rego policies for resource validation
- Debugs policy violations
- Provides multi-cluster support (dev, staging, prod)

**Usage:**
```
Point Claude to analyze your cluster
Examples:
- "Analyze my cluster's policy configuration"
- "What CRDs are installed in my cluster?"
- "Generate a policy to require team labels on pods"
- "Why is my deployment being rejected?"
```

**Key Features:**
- Dynamic real-time cluster queries via kubectl
- Works with any Kubernetes/OpenShift cluster
- Context-aware (adapts to active kubectl context)
- Generates ConstraintTemplates with Rego code
- Pattern library for common validation policies
- Violation debugging and troubleshooting

**Requirements:**
- Kubernetes MCP server (configured in `.claude/.mcp.json`)
- kubectl with cluster access
- See [MCP_SETUP_GUIDE.md](MCP_SETUP_GUIDE.md) for setup

---

#### k8s-best-practices-validator
**Purpose:** Validates Kubernetes/OpenShift clusters against Red Hat best practices and generates OPA Gatekeeper policies

**What it does:**
- Analyzes cluster resources against 30+ curated Red Hat best practices
- Identifies violations grouped by category (Security, Resource Management, Networking, Storage, Container Images, Pod Configuration, Workload Best Practices)
- Generates ConstraintTemplates and Constraints for Gatekeeper
- Saves policies as deployable YAML files organized by category
- Provides detailed compliance reports and remediation guidance

**Usage:**
```
Point Claude to validate your cluster
Examples:
- "Validate my cluster against Red Hat best practices"
- "Check for compliance violations in app-prod namespace"
- "Generate Gatekeeper policies for security best practices"
- "Analyze resource management violations"
```

**Key Features:**
- 30+ curated Red Hat best practices
- 7 categories: Security, Resource Management, Networking, Storage, Container Images, Pod Configuration, Workload Best Practices
- Uses Kubernetes MCP server for cluster analysis
- Automated Rego policy generation
- Dryrun-first approach for safe policy deployment
- Category-based file organization
- Complements cluster-policy-analyzer skill

**Requirements:**
- Kubernetes MCP server (configured in `.claude/.mcp.json`)
- kubectl with cluster access
- See skill README: [k8s-best-practices-validator/README.md](skills/infrastructure/k8s-best-practices-validator/README.md)

---

## Creating Your Own Skills

Each skill is a markdown file with the following structure:

```markdown
---
name: skill-name
description: Brief description of what the skill does
---

# Skill Name

You are a [description] assistant that helps users [purpose].

## Capabilities
[List of what the skill can do]

## When Invoked
**IMPORTANT: First, explicitly announce to the user: "Using the [Skill Name] skill..."**

1. [Step-by-step instructions for what to do]
2. [Use appropriate tools]
3. [Provide output in specific format]

## Best Practices
[Guidelines and recommendations]

## Example Workflow
[Sample usage scenario]
```

Place your skill file in the appropriate category folder under `skills/`.

## Usage in Claude Code

There are two ways to use these skills:

1. **Implicit invocation:** Simply describe what you want to do in natural language
   - "I want to set up this GitHub repo: https://github.com/user/project"
   - "Extract my tasks from ~/Documents/meeting-notes.txt"

2. **Explicit invocation:** Use the Skill tool
   ```
   Use the github-setup skill for: [repository URL]
   ```

The skill will announce itself when it's being used, so you always know which skill is active.

## Contributing

Feel free to add your own skills to this library! Create a pull request with:
- Your skill file in the appropriate category folder
- Updated README with skill description in the "Available Skills" section
- Clear documentation of what the skill does
- Example usage scenarios

### Skill Development Guidelines

When creating new skills:
- **Always announce the skill name** when invoked so users know which skill is running
- Use clear, actionable step-by-step instructions
- Leverage Claude Code tools (Read, Write, Bash, TodoWrite, etc.)
- Ask for user confirmation before executing potentially destructive commands
- Provide helpful error messages and fallback options
- Include example workflows and best practices

## License

MIT License - feel free to use and modify these skills for your needs.

## MCP Server Integration

This repository includes **Kubernetes MCP Server** configuration for real-time cluster analysis:

**What is MCP?**
Model Context Protocol enables Claude to interact with external systems (like Kubernetes clusters) through standardized tool interfaces.

**Setup:**
1. The `.claude/.mcp.json` file is version-controlled in this repo
2. When team members clone the repo, MCP server auto-downloads via npx
3. No manual installation required - just Node.js 16+ and kubectl

**See:** [MCP_SETUP_GUIDE.md](MCP_SETUP_GUIDE.md) and [TEAM_SHARING.md](TEAM_SHARING.md) for complete setup instructions

## Supported Platforms

- **Kubernetes** (all distributions: vanilla, minikube, kind, k3s)
- **OpenShift** 3.x and 4.x
- **Cloud Platforms**: GKE, EKS, AKS, DOKS
- **Gatekeeper/OPA** for policy enforcement

## Roadmap

Future skills under consideration:
- ~~Kubernetes cluster policy analyzer~~ ✅ Added!
- ~~Kubernetes best practices validator~~ ✅ Added!
- Terraform infrastructure analyzer
- Helm chart generator
- Prometheus metrics analyzer
- Service mesh policy generator
- Code reviewer with security checks
- Test generator
- Daily standup report creator

Suggestions and contributions welcome!
