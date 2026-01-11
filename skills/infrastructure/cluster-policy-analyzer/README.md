# Cluster Policy Analyzer

A Claude Code skill for analyzing Kubernetes/OpenShift cluster configurations, CRDs, and OPA/Gatekeeper policies using the Kubernetes MCP server.

## Overview

This skill analyzes your Kubernetes or OpenShift cluster to provide:
- **CRD Discovery** - Complete inventory of Custom Resource Definitions
- **Policy Analysis** - Gatekeeper ConstraintTemplates and active Constraints
- **Violation Debugging** - Understanding policy violations and their causes
- **Cluster Summarization** - Clear summaries of cluster configuration

## Prerequisites

1. **kubectl** or **oc** CLI installed and configured
2. **Active cluster context** pointing to your cluster
3. **Kubernetes MCP Server** configured for Claude

## Setup

### Step 1: Install Kubernetes MCP Server

Add the Kubernetes MCP server to Claude using:

```bash
claude mcp add kubernetes --scope user -- npx -y kubernetes-mcp-server@latest
```

This configures the MCP server for your user profile, making it available to all Claude Code sessions.

**Verify Installation:**
```bash
# Check MCP configuration
cat ~/.claude/mcp.json

# You should see:
{
  "mcpServers": {
    "kubernetes": {
      "command": "npx",
      "args": ["-y", "kubernetes-mcp-server@latest"]
    }
  }
}
```

### Step 2: Configure kubectl Access

Ensure you have access to a Kubernetes/OpenShift cluster:

```bash
# Check current context
kubectl config current-context

# Verify cluster access
kubectl cluster-info

# List available contexts
kubectl config get-contexts
```

### Step 3: Start Claude Code

The skill is automatically available when you start Claude Code:

```bash
claude
```

## Usage

### Example 1: Analyze Cluster CRDs

```
You: What CRDs are installed in my cluster?
```

**Claude will:**
- List all Custom Resource Definitions
- Group them by API group (gatekeeper.sh, openshift.io, etc.)
- Identify policy-related CRDs
- Provide a summary of cluster capabilities

### Example 2: Analyze Gatekeeper Policies

```
You: Analyze my cluster's policy configuration
```

**Claude will:**
- Discover ConstraintTemplates
- List active Constraints
- Show total violations
- Summarize enforcement actions

### Example 3: Debug Policy Violations

```
You: I have 8 violations for the teamlabel policy. Help me understand why.
```

**Claude will:**
- Retrieve the ConstraintTemplate and Constraint
- Analyze the Rego policy logic
- Show specific violations with resource names
- Explain why resources were rejected
- Suggest remediation steps

### Example 4: OpenShift Cluster Analysis

```
You: Analyze my OpenShift cluster
```

**Claude will:**
- Identify OpenShift-specific CRDs (Routes, Builds, ImageStreams)
- Analyze Security Context Constraints (SCCs)
- Summarize OpenShift resources
- Provide OpenShift-specific recommendations

## Switching Clusters

To analyze different clusters, switch your kubectl context:

```bash
# List available contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# Verify switch
kubectl config current-context

# Analyze the new cluster
```

Then ask Claude to analyze the cluster.

## How It Works

The skill uses the Kubernetes MCP server to execute these steps:

### 1. Identify Cluster Context
```
mcp__kubernetes__configuration_view(minified=true)
```

### 2. Discover CRDs
```
mcp__kubernetes__resources_list(
  apiVersion="apiextensions.k8s.io/v1",
  kind="CustomResourceDefinition"
)
```

### 3. List Namespaces
```
mcp__kubernetes__namespaces_list()
```

### 4. Analyze Gatekeeper (if installed)

**ConstraintTemplates:**
```
mcp__kubernetes__resources_list(
  apiVersion="templates.gatekeeper.sh/v1",
  kind="ConstraintTemplate"
)
```

**Constraints:**
```
mcp__kubernetes__resources_list(
  apiVersion="constraints.gatekeeper.sh/v1beta1",
  kind="K8sRequireNonRoot"
)
```

### 5. Summarize Findings

Claude organizes the information into a clear summary including:
- Overview (CRD count, namespaces, Gatekeeper status)
- CRD summary by API group
- Gatekeeper policy configuration
- Violations summary
- Key findings and recommendations

## Output Format

The skill produces structured analysis reports:

```markdown
## Cluster Analysis: minikube

**Current Context:** minikube
**Cluster Type:** Kubernetes

### Overview
- Total CRDs: 19
- Total Namespaces: 8
- Gatekeeper Installed: Yes

### CRD Summary by API Group

**gatekeeper.sh** (16 CRDs)
- ConstraintTemplates, Constraints, Mutations

**networking.k8s.io** (2 CRDs)
- NetworkPolicies, Ingress

### Gatekeeper Policy Configuration

**ConstraintTemplates** (3 total)
1. k8snocpulimits - Prevents CPU limits
2. k8srequirenetworkpolicy - Requires NetworkPolicies
3. teamlabel - Requires team label

**Active Constraints** (2 total)
1. no-cpu-limits (K8sNoCPULimits)
   - Enforcement: deny
   - Violations: 1

2. teampods (TeamLabel)
   - Enforcement: deny
   - Violations: 8

### Violations Summary
Total: 9 violations

By Constraint:
1. teampods - 8 violations
   - gatekeeper-system/Pod/gatekeeper-audit-xxx: Missing team label
   - opa/Pod/opa-xxx: Missing team label
   ...

### Key Findings
- 8 pods missing required team labels
- 1 pod has CPU limits (best practice: avoid CPU limits)

### Recommendations
- Add team labels to system pods or exclude system namespaces
- Review CPU limits configuration
```

## Supported Platforms

- **Kubernetes** (vanilla, minikube, kind, k3s)
- **OpenShift** 3.x and 4.x
- **Cloud Platforms**: GKE, EKS, AKS
- **Gatekeeper/OPA** 3.x

## Troubleshooting

### MCP Server Not Working

**Symptom:** Claude can't execute Kubernetes commands

**Solutions:**
1. Verify MCP installation:
   ```bash
   cat ~/.claude/mcp.json
   ```
2. Restart Claude Code
3. Check kubectl is in PATH:
   ```bash
   which kubectl
   ```
4. Test manually:
   ```bash
   npx -y kubernetes-mcp-server@latest
   ```


## Version History

- **v2.0.0** (2026-01-11)
  - Removed policy generation capabilities
  - Focus on cluster analysis and summarization

- **v1.0.0** (2025-10-02)
  - Initial release

---

**License:** Internal Use
**Last Updated:** 2026-01-11
