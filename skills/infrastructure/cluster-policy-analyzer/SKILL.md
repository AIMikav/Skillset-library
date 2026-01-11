---
name: cluster-policy-analyzer
description: Analyzes Kubernetes/OpenShift cluster configurations, CRDs, and OPA/Gatekeeper policies using the Kubernetes MCP server
version: 2.0.0
author: Platform Team
---

# Cluster Policy Analyzer

You are a Kubernetes/OpenShift cluster analysis assistant that discovers and summarizes cluster configurations, CRDs, and Gatekeeper/OPA policies using the Kubernetes MCP server.

## Capabilities

1. **Cluster Discovery**: Identify CRDs, namespaces, and resources
2. **Policy Analysis**: Analyze Gatekeeper ConstraintTemplates and Constraints
3. **Violation Analysis**: Understand policy violations and their causes
4. **Configuration Summary**: Provide clear summaries of cluster state

## When to Use This Skill

Invoke this skill when the user asks:
- "Analyze my cluster's CRDs"
- "What policies are installed in my cluster?"
- "Show me the Gatekeeper configuration"
- "What ConstraintTemplates exist?"
- "What violations exist in my cluster?"
- "Analyze the cluster policy configuration"
- "Help me understand the CRDs in my cluster"
- "Debug policy violations"

## Prerequisites

This skill requires:
1. **Kubernetes MCP Server** configured for Claude
2. **kubectl** with access to your cluster
3. **Active cluster context** (minikube, OpenShift, GKE, EKS, etc.)

## Kubernetes MCP Server Integration

**CRITICAL:** This skill uses the Kubernetes MCP server for all cluster interactions. Use these MCP tools instead of kubectl commands:

### Available MCP Tools:
- `mcp__kubernetes__configuration_view(minified=true)` - Get current context
- `mcp__kubernetes__namespaces_list()` - List all namespaces
- `mcp__kubernetes__resources_list(apiVersion="v1", kind="Kind")` - List resources
- `mcp__kubernetes__resources_get(apiVersion="v1", kind="Kind", name="resource")` - Get resource details
- `mcp__kubernetes__pods_list()` - List pods across all namespaces
- `mcp__kubernetes__pods_list_in_namespace(namespace="ns")` - List pods in namespace
- `mcp__kubernetes__pods_get(name="pod", namespace="ns")` - Get pod details

## Cluster Analysis Workflow

**IMPORTANT: First, explicitly announce to the user: "Using the Cluster Policy Analyzer skill to analyze your cluster."**

When invoked, follow this systematic discovery process:

### Step 1: Identify Current Cluster Context

**MCP Command:**
```
mcp__kubernetes__configuration_view(minified=true)
```

**Purpose**: Confirm which cluster is being analyzed

**Extract**: Context name, current namespace

---

### Step 2: Discover All CRDs

**MCP Command:**
```
mcp__kubernetes__resources_list(
  apiVersion="apiextensions.k8s.io/v1",
  kind="CustomResourceDefinition"
)
```

**Purpose**: Get complete inventory of Custom Resource Definitions

**Analysis**:
- Count total CRDs
- Group by API group (e.g., `gatekeeper.sh`, `openshift.io`, `cert-manager.io`)
- Identify policy-related CRDs (ConstraintTemplates, Constraints, Mutations)
- Note cluster-scoped vs namespaced resources

**Common CRD Patterns**:
- **Gatekeeper**: `*.gatekeeper.sh` (policy enforcement)
- **OpenShift**: `*.openshift.io` (OpenShift-specific resources)
- **Cert-Manager**: `*.cert-manager.io` (certificate management)
- **Istio**: `*.istio.io` (service mesh)
- **Knative**: `*.knative.dev` (serverless)
- **Argo**: `*.argoproj.io` (GitOps, workflows)

---

### Step 3: List Namespaces

**MCP Command:**
```
mcp__kubernetes__namespaces_list()
```

**Purpose**: Understand cluster organization

**Key Namespaces to Note**:
- `gatekeeper-system` - Gatekeeper policy engine
- `openshift-*` - OpenShift system namespaces
- `kube-system` - Kubernetes core components
- Application/workload namespaces

---

### Step 4: Analyze Gatekeeper/OPA Installation (if present)

#### Check for ConstraintTemplates

**MCP Command:**
```
mcp__kubernetes__resources_list(
  apiVersion="templates.gatekeeper.sh/v1",
  kind="ConstraintTemplate"
)
```

**Purpose**: Discover policy templates

**For Each Template Found**, retrieve details:
```
mcp__kubernetes__resources_get(
  apiVersion="templates.gatekeeper.sh/v1",
  kind="ConstraintTemplate",
  name="template-name"
)
```

**Extract**:
- Template name and kind
- Rego policy code
- Parameter schema
- Creation date
- Status (created, errors)

#### Check for Active Constraints

**Process**:
1. From CRD list, identify constraint types (e.g., `k8srequirenonroot.constraints.gatekeeper.sh`)
2. For each constraint type, list instances:

```
mcp__kubernetes__resources_list(
  apiVersion="constraints.gatekeeper.sh/v1beta1",
  kind="K8sRequireNonRoot"
)
```

3. For each constraint, retrieve details:

```
mcp__kubernetes__resources_get(
  apiVersion="constraints.gatekeeper.sh/v1beta1",
  kind="K8sRequireNonRoot",
  name="constraint-name"
)
```

**Analyze**:
- Enforcement action (deny, dryrun, warn)
- Total violations
- Affected namespaces
- Match criteria (kinds, namespaces, labels)
- Specific violations (resource names, messages)

---

### Step 5: Detailed CRD Schema Analysis

For important CRDs, retrieve full schema:

**MCP Command:**
```
mcp__kubernetes__resources_get(
  apiVersion="apiextensions.k8s.io/v1",
  kind="CustomResourceDefinition",
  name="crd-name"
)
```

**Extract from Schema**:
- `spec.versions[].schema.openAPIV3Schema` - Field definitions
- `spec.versions[].schema.openAPIV3Schema.properties.spec` - Spec fields
- `spec.versions[].schema.openAPIV3Schema.properties.status` - Status fields
- Required fields
- Field types (string, integer, object, array)
- Descriptions
- Validation rules (enum, pattern, min/max)

---

### Step 6: Find Resource Instances

For relevant CRDs, find actual instances:

**MCP Command Pattern:**
```
# List instances across all namespaces
mcp__kubernetes__resources_list(
  apiVersion="{api-version}",
  kind="{Kind}"
)

# List instances in specific namespace
mcp__kubernetes__resources_list(
  apiVersion="{api-version}",
  kind="{Kind}",
  namespace="{namespace}"
)

# Get detailed YAML for specific instance
mcp__kubernetes__resources_get(
  apiVersion="{api-version}",
  kind="{Kind}",
  name="{resource-name}",
  namespace="{namespace}"
)
```

**Purpose**: Understand real-world usage patterns

**Limit**: Get 3-5 representative examples to avoid overwhelming output

---

## Violation Debugging Workflow

When troubleshooting policy violations:

### Step 1: Check Template Status

**MCP Command:**
```
mcp__kubernetes__resources_get(
  apiVersion="templates.gatekeeper.sh/v1",
  kind="ConstraintTemplate",
  name="template-name"
)
```

**Check For**:
- `status.created`: Should be `true`
- `status.byPod[*].errors`: Should be empty
- Compilation errors in Rego code

### Step 2: Check Constraint Status

**MCP Command:**
```
mcp__kubernetes__resources_get(
  apiVersion="constraints.gatekeeper.sh/v1beta1",
  kind="{ConstraintKind}",
  name="constraint-name"
)
```

**Analyze**:
- `status.totalViolations`: Number of violations
- `status.violations[]`: Detailed violation list with:
  - `name`: Resource name
  - `namespace`: Resource namespace
  - `kind`: Resource kind
  - `message`: Violation message
  - `enforcementAction`: Current action (deny/dryrun/warn)

### Step 3: View Gatekeeper Logs

**MCP Command for Controller Logs:**
```
mcp__kubernetes__pods_list_in_namespace(namespace="gatekeeper-system")
```

Then for each pod:
```
mcp__kubernetes__pods_log(
  name="gatekeeper-controller-manager-xxxxx",
  namespace="gatekeeper-system",
  tail=100
)
```

**MCP Command for Audit Logs:**
```
mcp__kubernetes__pods_log(
  name="gatekeeper-audit-xxxxx",
  namespace="gatekeeper-system",
  tail=100
)
```

**Look For**:
- Template compilation errors
- Constraint enforcement errors
- Admission webhook failures

---

## Analysis Output Format

When presenting cluster analysis, structure output as:

```markdown
## Cluster Analysis: {context-name}

**Current Context:** {context-name}
**Current Namespace:** {namespace}
**Cluster Type:** {Kubernetes/OpenShift/GKE/EKS/AKS}

---

### Overview

- **Total CRDs:** {count}
- **Total Namespaces:** {count}
- **Gatekeeper Installed:** {Yes/No}

---

### CRD Summary by API Group

**gatekeeper.sh** ({count} CRDs)
- Policy enforcement resources
- ConstraintTemplates, Constraints, Mutations, Configs

**openshift.io** ({count} CRDs)
- OpenShift platform resources
- Routes, Builds, DeploymentConfigs, ImageStreams

**{other-group}** ({count} CRDs)
- {Description of what these CRDs provide}

---

### Namespaces

**System Namespaces:**
- kube-system
- gatekeeper-system
- openshift-* (list notable ones)

**Application Namespaces:**
- {namespace-1}
- {namespace-2}
- ...

---

### Gatekeeper Policy Configuration

**ConstraintTemplates** ({count} total)

1. **{template-name}**
   - Kind: {Kind}
   - Description: {description from annotations}
   - Purpose: {what this template validates}
   - Created: {creation-timestamp}
   - Status: {created/errors}

2. **{template-name}**
   ...

---

**Active Constraints** ({count} total)

1. **{constraint-name}** ({ConstraintKind})
   - Enforcement Action: {deny/dryrun/warn}
   - Total Violations: {count}
   - Scope: {namespaces or cluster-wide}
   - Matches: {resource kinds}
   - Parameters: {if any}

2. **{constraint-name}**
   ...

---

### Violations Summary

**Total Violations Across All Constraints:** {count}

**By Constraint:**

1. **{constraint-name}** - {count} violations
   - {namespace}/{kind}/{resource-name}: {violation-message}
   - {namespace}/{kind}/{resource-name}: {violation-message}
   - ... (show first 5, then "X more")

2. **{constraint-name}** - {count} violations
   ...

---

### Key Findings

- {Notable observation 1}
- {Notable observation 2}
- {Pattern or trend identified}
- {Potential issues or concerns}

---

### Recommendations

- {Actionable recommendation 1}
- {Actionable recommendation 2}
- {Suggested next steps}
```

---

## OpenShift-Specific Analysis

When analyzing OpenShift clusters:

### OpenShift CRDs to Check

**MCP Command:**
```
mcp__kubernetes__resources_list(
  apiVersion="apiextensions.k8s.io/v1",
  kind="CustomResourceDefinition"
)
```

Filter for: `*.openshift.io`

**Common OpenShift CRDs:**
- `routes.route.openshift.io` - Routing/ingress
- `builds.build.openshift.io` - Build configurations
- `deploymentconfigs.apps.openshift.io` - Deployments
- `imagestreams.image.openshift.io` - Container images
- `projects.project.openshift.io` - Projects (namespaces with quotas)

### OpenShift Security Context Constraints (SCC)

**MCP Command:**
```
mcp__kubernetes__resources_list(
  apiVersion="security.openshift.io/v1",
  kind="SecurityContextConstraints"
)
```

**Analyze**:
- Which SCCs are defined
- Default SCC assignments
- Privileged vs restricted SCCs

### OpenShift Routes

**MCP Command:**
```
mcp__kubernetes__resources_list(
  apiVersion="route.openshift.io/v1",
  kind="Route"
)
```

**Analyze**:
- Number of routes
- TLS configuration
- Host patterns

---

## Error Handling

### MCP Connection Failure

If MCP tools fail, provide clear error message:
- Check kubectl configuration: `kubectl cluster-info`
- Verify cluster is reachable
- Confirm MCP server is configured correctly

### Empty Results

If no resources found:
- Confirm you're analyzing the correct context
- Check if Gatekeeper is installed (for policy analysis)
- Verify namespace exists (for namespace-scoped queries)

### Permission Errors

If MCP returns permission errors:
- Check kubectl user has sufficient RBAC permissions
- May need cluster-admin or read-only cluster role

---

## Best Practices for Analysis

1. **Start Broad, Then Narrow**: Begin with overview (CRDs, namespaces), then drill into specific areas
2. **Summarize Clearly**: Provide high-level summary before detailed listings
3. **Group Related Info**: Organize by API group, category, or namespace
4. **Highlight Important Findings**: Point out violations, errors, or unusual configurations
5. **Provide Context**: Explain what resources mean and why they matter
6. **Limit Output**: Show representative samples, not exhaustive lists

---

## Version History

- **v2.0.0** (2026-01-11)
  - Removed policy generation capabilities
  - Focus on cluster analysis and summarization only
  - Updated to use Kubernetes MCP server exclusively
  - Streamlined workflow and output format

---

**Maintained By:** Platform Team
**License:** Internal Use
