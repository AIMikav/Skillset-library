# Team Sharing Guide

## How to Share These Skills with Your Team

This document explains how the skills and MCP server configuration work when shared with team members.

## What Gets Shared (Version-Controlled)

When you commit this repository to Git, team members get:

```
openshift-policy-skills/
├── .claude/
│   ├── .mcp.json                          #  MCP server config (shared)
│   └── skills/
│       ├── cluster-policy-analyzer/       #  Dynamic skill (shared)
│       │   ├── SKILL.md
│       │   └── README.md
│       └── gatekeeper-policy-generation/  #  Static skill (shared)
│           └── SKILL.md
├── .gitignore                             #  Git ignore rules (shared)
├── README.md                              #  Main documentation (shared)
├── MCP_SETUP_GUIDE.md                     #  Setup guide (shared)
└── TEAM_SHARING.md                        #  This file (shared)
```

## What Team Members Need to Install

### Required (One-Time Setup)

1. **Node.js 16+**
   ```bash
   # macOS
   brew install node

   # Linux (Ubuntu/Debian)
   curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
   sudo apt-get install -y nodejs

   # Verify
   node --version  # Should be 16.0.0+
   ```

2. **kubectl or oc CLI**
   ```bash
   # macOS
   brew install kubectl

   # Linux
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl
   sudo mv kubectl /usr/local/bin/

   # Verify
   kubectl version --client
   ```

3. **Claude Code**
   - Install from https://claude.ai/code

### NOT Required (Auto-Installed)

❌ **kubernetes-mcp-server** - Downloaded automatically by npx
❌ **Manual configuration** - MCP config is in the repo

## Team Member Onboarding (3 Steps)

### Step 1: Clone the Repository

```bash
git clone https://github.com/your-org/openshift-policy-skills.git
cd openshift-policy-skills
```

**What they get**:
- `.claude/.mcp.json` - MCP server configuration
- Skills files - Cluster analysis instructions
- Documentation - Usage guides

### Step 2: Configure Cluster Access

```bash
# Verify they have cluster access
kubectl cluster-info

# If not, they need to configure kubectl:

# For minikube:
minikube start

# For cloud clusters:
# GKE:
gcloud container clusters get-credentials <cluster-name>

# EKS:
aws eks update-kubeconfig --name <cluster-name>

# AKS:
az aks get-credentials --resource-group <rg> --name <cluster-name>

# For OpenShift:
oc login <api-url>
```

### Step 3: Start Claude Code

```bash
# In the openshift-policy-skills directory
claude
```

**What happens automatically**:
1. Claude reads `.claude/.mcp.json`
2. Runs `npx -y kubernetes-mcp-server@latest`
3. npx downloads the MCP server (first time only, ~2-5 seconds)
4. MCP server starts and connects to kubectl
5. Skills become active

**First-time output**:
```
Starting Claude Code...
Loading MCP servers...
  ✓ kubernetes-mcp-server (downloading... done)
  ✓ kubernetes-mcp-server ready

Skills loaded:
  - cluster-policy-analyzer
  - gatekeeper-policy-generation

Ready! Try: "Analyze my cluster"
```

**Subsequent times**:
```
Starting Claude Code...
Loading MCP servers...
  ✓ kubernetes-mcp-server (using cached)
  ✓ kubernetes-mcp-server ready

Skills loaded: 2
Ready!
```

## How the MCP Server Works

### The Magic of npx

The `.claude/.mcp.json` uses `npx`:

```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "npx",           // ← Package runner
      "args": [
        "-y",                      // ← Auto-accept downloads
        "kubernetes-mcp-server@latest"  // ← Package to run
      ]
    }
  }
}
```

**npx behavior**:
1. Checks if `kubernetes-mcp-server` is in local cache
2. If not found, downloads from npm
3. Caches it in `~/.npm/_npx/`
4. Runs it
5. Next time: uses cached version (instant start)

### What Team Members DON'T Need

They do **NOT** need to:
- ❌ Install the MCP server manually
- ❌ Run `npm install`
- ❌ Configure `.claude/.mcp.json` (it's in the repo!)
- ❌ Set up anything special

It **just works** when they start Claude Code.

## Publishing to Git

### Initialize and Commit (First Time)

```bash
# Initialize git (if not already done)
git init

# Add files
git add .
git add .claude/  # Ensure .claude/ is included!

# Commit
git commit -m "Initial commit: Add cluster policy skills"

# Add remote
git remote add origin https://github.com/your-org/openshift-policy-skills.git

# Push
git push -u origin main
```

### Verify .claude/ is Tracked

```bash
# Check what's committed
git ls-files | grep .claude

# Should show:
# .claude/.mcp.json
# .claude/skills/cluster-policy-analyzer/SKILL.md
# .claude/skills/cluster-policy-analyzer/README.md
# .claude/skills/gatekeeper-policy-generation/SKILL.md
```

## Team Member Workflow

### Developer Alice (Dev Cluster)

```bash
# Clone repo
git clone https://github.com/your-org/openshift-policy-skills.git
cd openshift-policy-skills

# Configure kubectl for dev cluster
kubectl config use-context dev-cluster

# Start Claude
claude
```

```
Alice: Analyze my cluster's Gatekeeper policies

Claude: [Connects to dev-cluster via kubectl]
        [Discovers CRDs, policies, violations]
        [Provides analysis]
```

### Platform Engineer Bob (Prod Cluster)

```bash
# Clone same repo
git clone https://github.com/your-org/openshift-policy-skills.git
cd openshift-policy-skills

# Configure kubectl for prod cluster
kubectl config use-context prod-cluster

# Start Claude
claude
```

```
Bob: Generate a policy to require 2 replicas in production

Claude: [Connects to prod-cluster via kubectl]
        [Generates ConstraintTemplate]
        [Creates Constraint with enforcementAction: dryrun]
        [Provides testing commands]
```

### Same Skill, Different Clusters ✨

The **same skill file** works for both Alice and Bob because:
- The skill uses `kubectl` commands
- kubectl respects the active context
- No cluster-specific configuration in the skill

## Updating Skills

### Adding a New Pattern

When you discover a useful policy pattern:

```bash
# Edit the skill
vim .claude/skills/cluster-policy-analyzer/SKILL.md

# Add your pattern to the "Common Rego Patterns Library" section
# For example, add a new "Pod Security" pattern

# Commit and push
git add .claude/skills/cluster-policy-analyzer/SKILL.md
git commit -m "Add pod security Rego pattern"
git push
```

### Team Members Get Updates

```bash
# Team members pull updates
git pull

# Restart Claude (to reload skills)
exit  # Exit current Claude session
claude  # Start new session

# New pattern is now available!
```

## Multi-Cluster Scenarios

### Working with Multiple Clusters

Team members can work with any cluster they have access to:

```bash
# List available clusters
kubectl config get-contexts

# Output:
# CURRENT   NAME              CLUSTER
# *         dev-cluster       dev-cluster
#           staging-cluster   staging-cluster
#           prod-cluster      prod-cluster
```

### Switching Contexts in Claude Session

```bash
# Inside Claude Code session
You: Use dev cluster
```

```bash
# Claude can help switch contexts
kubectl config use-context dev-cluster
```

```
You: Now analyze this cluster
```

The skill adapts to whichever cluster is active.

## Version Pinning (Optional)

If you want all team members on the same MCP server version:

**Edit `.claude/.mcp.json`**:
```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "npx",
      "args": [
        "-y",
        "kubernetes-mcp-server@1.2.3"  // ← Pin specific version
      ]
    }
  }
}
```

**Commit and push**:
```bash
git add .claude/.mcp.json
git commit -m "Pin MCP server to v1.2.3"
git push
```

**Team members pull**:
```bash
git pull
# Restart Claude to use new version
```

## Troubleshooting for Team Members

### "MCP server not found"

**Solution**: Install Node.js
```bash
node --version  # Should be 16+
npm --version   # Should exist
```

### "Cannot connect to cluster"

**Solution**: Configure kubectl
```bash
kubectl cluster-info  # Should show cluster info
kubectl config current-context  # Should show context name
```

### "Skills not working"

**Solution**: Verify files exist
```bash
ls .claude/.mcp.json  # Should exist
ls .claude/skills/*/SKILL.md  # Should list skill files

# Restart Claude
exit
claude
```

## Security Considerations

### What's Shared

 **Safe to share**:
- Skill instructions (generic, no secrets)
- MCP server configuration (just commands)
- Documentation

❌ **NOT shared** (and shouldn't be):
- `~/.kube/config` (cluster credentials)
- Cloud provider credentials
- API tokens

### Access Control

Each team member:
- Uses their own kubectl credentials
- Has their own cluster access permissions
- Can only query clusters they're authorized for

The skill **doesn't grant access** to clusters - it only helps analyze clusters you already have access to.

## Summary

### What You Share (Git)

```bash
git push  # Shares:
          # - .claude/.mcp.json (MCP config)
          # - .claude/skills/ (skill files)
          # - Documentation
```

### What Team Members Install

1. Node.js (for npx)
2. kubectl (for cluster access)
3. Claude Code

### What Happens Automatically

1. npx downloads MCP server
2. Claude loads skills
3. Skills work with their clusters

### Why It Works

- **MCP config is version-controlled** → Everyone gets it
- **npx downloads on-demand** → No manual installation
- **Skills use kubectl** → Works with any cluster
- **Context-aware** → Adapts to active cluster

---

**Result**: Team members clone, configure their cluster access, and immediately have working cluster analysis skills! ✨

**Last Updated**: 2026-01-08
**Version**: 1.0.0
