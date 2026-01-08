# Cluster Policy Analyzer Skill

A reusable Claude Code skill for real-time discovery and analysis of Kubernetes/OpenShift clusters, CRDs, and OPA/Gatekeeper policies.

## Overview

This skill enables Claude to dynamically analyze any Kubernetes or OpenShift cluster you have access to, providing:

- **Real-time cluster discovery** - CRDs, namespaces, resources
- **Policy analysis** - Gatekeeper ConstraintTemplates and Constraints
- **Rego generation** - Custom policy creation based on your requirements
- **Violation debugging** - Understanding and fixing policy violations
- **Multi-cluster support** - Works with any kubectl context

## Quick Start

### Prerequisites

1. **kubectl** or **oc** CLI installed and configured
2. **Active cluster context** - pointing to your cluster
3. **Kubernetes MCP Server** - configured in your project

### Setup for Team Members

#### Step 1: Clone the Repository

```bash
git clone https://github.com/AIMikav/Skillset-library.git
cd Skillset-library
```

#### Step 2: Configure MCP Server

The MCP server configuration is already set up in `.claude/.mcp.json` at the repository root:

```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "npx",
      "args": [
        "-y",
        "kubernetes-mcp-server@latest"
      ]
    }
  }
}
```

This configuration is **version-controlled** and works for all team members.

#### Step 3: Verify kubectl Access

Ensure you have access to a Kubernetes/OpenShift cluster:

```bash
# Check current context
kubectl config current-context

# Verify cluster access
kubectl cluster-info

# List contexts (if you have multiple clusters)
kubectl config get-contexts
```

#### Step 4: Start Claude Code

The skill will be automatically available when you start Claude Code in this directory:

```bash
# Navigate to repository directory
cd Skillset-library

# Start Claude Code
claude
```

### Switching Clusters

To analyze different clusters, simply switch your kubectl context:

```bash
# List available contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# Verify switch
kubectl config current-context

# Now ask Claude to analyze the cluster
# "Analyze my cluster's policy configuration"
```

## Usage Examples

### Example 1: Discover Cluster CRDs

```
You: What CRDs are installed in my cluster?

Claude: [Uses skill to run kubectl get crds and provides analysis]
```

### Example 2: Analyze Gatekeeper Policies

```
You: Show me all Gatekeeper policies in my cluster

Claude: [Discovers ConstraintTemplates, active Constraints, and violations]
```

### Example 3: Generate a New Policy

```
You: Create a Gatekeeper policy to require memory limits on all containers

Claude: [Analyzes existing patterns, generates ConstraintTemplate with Rego,
         creates Constraint YAML, provides testing instructions]
```

### Example 4: Debug Violations

```
You: I have 8 violations for teamlabel policy. Help me understand why.

Claude: [Retrieves policy, analyzes violations, explains cause, suggests fixes]
```

### Example 5: Compare Environments

```
You: Compare the policies between dev and prod clusters

# First, switch to dev
kubectl config use-context dev-cluster

You: Analyze dev cluster policies

# Then switch to prod
kubectl config use-context prod-cluster

You: Analyze prod cluster policies and compare with dev
```

## How It Works

The skill provides Claude with:

1. **Discovery Workflow** - Step-by-step commands to analyze clusters
2. **Rego Pattern Library** - Common policy patterns to use as templates
3. **Best Practices** - Guidelines for policy creation and enforcement
4. **Debugging Techniques** - How to troubleshoot policy issues

Claude executes kubectl commands in real-time to:
- List CRDs and understand cluster capabilities
- Retrieve ConstraintTemplate YAML and extract Rego code
- Find active Constraints and their violations
- Analyze resource instances for context

## Skill Capabilities

### Cluster Discovery
- List all CRDs with grouping by API group
- Identify cluster type (Kubernetes, OpenShift, managed cloud)
- Discover namespaces and their organization
- Find Gatekeeper/OPA installation

### Policy Analysis
- List all ConstraintTemplates
- Extract and explain Rego policies
- Show active Constraints and their configuration
- Display violations with context

### Policy Generation
- Create ConstraintTemplates with custom Rego
- Generate Constraint instances with proper scoping
- Follow cluster-specific patterns
- Include parameter schemas

### Troubleshooting
- Debug policy compilation errors
- Explain violation messages
- Suggest fixes for rejected resources
- Analyze Gatekeeper logs

## Supported Cluster Types

- **Minikube** - Local development clusters
-  **OpenShift** - Red Hat OpenShift (3.x, 4.x)
-  **GKE** - Google Kubernetes Engine
-  **EKS** - Amazon Elastic Kubernetes Service
-  **AKS** - Azure Kubernetes Service
-  **Rancher** - Rancher-managed clusters
-  **Kind** - Kubernetes in Docker
-  **K3s** - Lightweight Kubernetes

## File Structure

```
skills/infrastructure/cluster-policy-analyzer/
├── SKILL.md              # Main skill file (Claude's instructions)
└── README.md             # This file (team documentation)
```

## Advanced Usage

### Custom Rego Patterns

The skill includes patterns for:
- Required labels enforcement
- Resource limits validation
- Image registry restrictions
- Replica count requirements
- Security context enforcement
- ConfigMap/Secret validation
- Namespace labeling

### OpenShift-Specific Features

When working with OpenShift, the skill can:
- Analyze Security Context Constraints (SCCs)
- Discover Routes and their configurations
- Work with DeploymentConfigs
- Understand ImageStreams
- Handle Projects (OpenShift namespaces)

### Testing Policies Locally

The skill guides Claude to:
1. Extract Rego from ConstraintTemplates
2. Create test input JSON
3. Use OPA CLI for local validation
4. Iterate before deploying to cluster

## Troubleshooting

### MCP Server Not Working

**Symptom**: Claude can't execute kubectl commands

**Solutions**:
1. Verify `.claude/.mcp.json` exists in the project root
2. Restart Claude Code session
3. Check kubectl is in your PATH: `which kubectl`
4. Test MCP manually: `npx -y kubernetes-mcp-server@latest --help`

### Cluster Access Issues

**Symptom**: kubectl commands fail with connection errors

**Solutions**:
1. Verify cluster is running: `kubectl cluster-info`
2. Check context is correct: `kubectl config current-context`
3. Validate credentials haven't expired
4. For OpenShift: `oc login` if needed

### Skill Not Activating

**Symptom**: Claude doesn't use the skill

**Solutions**:
1. Mention keywords: "analyze cluster", "CRDs", "Gatekeeper", "policy"
2. Explicitly ask: "Use the cluster-policy-analyzer skill"
3. Verify skill file exists: `ls skills/infrastructure/cluster-policy-analyzer/SKILL.md`

## Sharing with Team

### Via Git

This skill is version-controlled and automatically shared when team members clone the repository:

```bash
git clone https://github.com/AIMikav/Skillset-library.git
cd Skillset-library
claude  # Skill is immediately available
```

### Customization

Teams can customize the skill by editing `SKILL.md`:
- Add company-specific policy patterns
- Include environment-specific conventions
- Document custom CRDs
- Add team-specific workflows

**After customization**, commit and push:

```bash
git add skills/infrastructure/cluster-policy-analyzer/SKILL.md
git commit -m "Add custom policy patterns"
git push
```

## Best Practices for Teams

### 1. Maintain One Skill Version
Keep the skill in a central repository and have team members pull updates.

### 2. Document Custom Patterns
As you create policies, add successful patterns to the skill for reuse.

### 3. Version Control Everything
Store ConstraintTemplates and Constraints in Git alongside the skill.

### 4. Use Consistent Naming
Establish naming conventions for policies (e.g., `k8s-require-*`, `company-*`).

### 5. Test in Dev First
Use dryrun mode in development before enforcing in production.

### 6. Review Together
Have policy changes reviewed like code changes.

## Contributing

To improve this skill:

1. **Fork or branch** the repository
2. **Edit** `skills/infrastructure/cluster-policy-analyzer/SKILL.md`
3. **Test** your changes with Claude Code
4. **Submit** a pull request or merge to main
5. **Document** what you changed

## Security Considerations

- **Credentials**: The skill uses your local kubectl credentials
- **Permissions**: Only accesses what your kubectl user can access
- **Read-Mostly**: Primarily read operations; writes are explicit and shown
- **Audit Trail**: All kubectl commands are visible in Claude's output

## Support

For issues or questions:
- **Check logs**: Claude shows all kubectl commands it executes
- **Review docs**: See [MCP_SETUP_GUIDE.md](../../../MCP_SETUP_GUIDE.md) and [TEAM_SHARING.md](../../../TEAM_SHARING.md)
- **Team channel**: Post in your team's support channel
- **File issue**: Create an issue in the [repository](https://github.com/AIMikav/Skillset-library/issues)

## Version History

- **v1.0.0** (2026-01-08) - Initial release
  - Real-time cluster discovery
  - Gatekeeper policy analysis
  - Rego generation patterns
  - Multi-cluster support

---

**Maintained By**: Platform Team
**License**: Internal Use
**Last Updated**: 2026-01-08
