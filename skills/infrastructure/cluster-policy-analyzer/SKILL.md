---
name: cluster-policy-analyzer
description: Dynamically discover and analyze Kubernetes/OpenShift cluster configurations, CRDs, and OPA/Gatekeeper policies in real-time using the Kubernetes MCP server
version: 1.0.0
author: Platform Team
---

# Dynamic Cluster Policy Analyzer

This skill enables real-time discovery and analysis of any Kubernetes or OpenShift cluster's policy configuration using the Kubernetes MCP server. Use this skill to understand cluster CRDs, analyze Gatekeeper/OPA policies, and generate Rego validation rules.

## When to Use This Skill

Invoke this skill when:
- "Analyze my cluster's CRDs"
- "What policies are installed in my cluster?"
- "Show me the Gatekeeper configuration"
- "Discover what ConstraintTemplates exist"
- "Generate a Rego policy for [resource type]"
- "What violations exist in my cluster?"
- "Analyze the cluster policy configuration"
- "Help me understand the CRDs in my cluster"

## Prerequisites

This skill requires:
1. **Kubernetes MCP Server** configured in `.claude/.mcp.json`
2. **kubectl** with access to your cluster
3. **Active cluster context** (minikube, OpenShift, GKE, EKS, etc.)

## Cluster Discovery Workflow

When invoked, follow this systematic discovery process:

### Step 1: Identify Current Cluster Context

**Command**:
```bash
kubectl config current-context
```

**Purpose**: Confirm which cluster we're analyzing

**Expected Output**: Context name (e.g., `minikube`, `production-cluster`, `staging-context`)

### Step 2: Discover All CRDs

**Command**:
```bash
kubectl get crds -o custom-columns=NAME:.metadata.name,GROUP:.spec.group,KIND:.spec.names.kind,SCOPE:.spec.scope --no-headers
```

**Purpose**: Get a complete inventory of Custom Resource Definitions

**Analysis**:
- Count total CRDs
- Group by API group (e.g., `gatekeeper.sh`, `openshift.io`, `serving.kserve.io`)
- Identify policy-related CRDs (templates, constraints, mutations)
- Note cluster-scoped vs namespaced resources

**Common CRD Patterns**:
- **Gatekeeper**: `*.gatekeeper.sh` (policy enforcement)
- **OpenShift**: `*.openshift.io` (OpenShift-specific resources)
- **Cert-Manager**: `*.cert-manager.io` (certificate management)
- **Istio**: `*.istio.io` (service mesh)
- **Knative**: `*.knative.dev` (serverless)

### Step 3: List Namespaces

**Command**:
```bash
kubectl get namespaces -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,AGE:.metadata.creationTimestamp --no-headers
```

**Purpose**: Understand cluster organization

**Key Namespaces to Note**:
- `gatekeeper-system` - Gatekeeper policy engine
- `openshift-*` - OpenShift system namespaces
- `kube-system` - Kubernetes core components
- Application/workload namespaces

### Step 4: Analyze Gatekeeper/OPA Installation (if present)

#### Check for ConstraintTemplates

**Command**:
```bash
kubectl get constrainttemplates -o custom-columns=NAME:.metadata.name,CREATED:.metadata.creationTimestamp --no-headers 2>/dev/null || echo "Gatekeeper not installed or no ConstraintTemplates found"
```

**Purpose**: Discover policy templates

**For Each Template Found**:
```bash
kubectl get constrainttemplate <template-name> -o yaml
```

**Extract**:
- Template name and kind
- Rego policy code
- Parameter schema
- Creation date
- Status (created, errors)

#### Check for Active Constraints

**Command Pattern**:
```bash
# List all constraint types
kubectl get crd -o name | grep 'constraints.gatekeeper.sh'

# For each constraint type, list instances
kubectl get <constraint-kind>.constraints.gatekeeper.sh -A
```

**Purpose**: Identify active policy enforcement

**Analyze**:
- Enforcement action (deny, dryrun, warn)
- Total violations
- Affected namespaces
- Match criteria (kinds, namespaces, labels)

### Step 5: Detailed CRD Schema Analysis

For important CRDs, retrieve full schema:

**Command**:
```bash
kubectl get crd <crd-name> -o yaml
```

**Extract from Schema**:
- `spec.versions[].schema.openAPIV3Schema` - Field definitions
- `spec.versions[].schema.openAPIV3Schema.properties.spec` - Spec fields
- `spec.versions[].schema.openAPIV3Schema.properties.status` - Status fields
- Required fields
- Field types (string, integer, object, array)
- Descriptions
- Validation rules (enum, pattern, min/max)

### Step 6: Find Resource Instances

For relevant CRDs, find actual instances:

**Command Pattern**:
```bash
# List instances across all namespaces
kubectl get <plural-name> -A -o wide

# Get detailed YAML for analysis
kubectl get <plural-name> <instance-name> -n <namespace> -o yaml
```

**Purpose**: Understand real-world usage patterns

**Limit**: Get 3-5 representative examples to avoid overwhelming output

## Rego Policy Generation Workflow

When asked to generate a Rego policy:

### Step 1: Understand the Requirement

Ask clarifying questions if needed:
- What resource types should be validated? (Pod, Deployment, etc.)
- What's the validation rule? (required labels, resource limits, etc.)
- What enforcement action? (deny, dryrun, warn)
- Which namespaces? (all, specific, exclude system namespaces)

### Step 2: Analyze Similar Existing Policies

Search for similar ConstraintTemplates:
```bash
kubectl get constrainttemplates -o yaml
```

Look for patterns matching the requirement:
- Label enforcement → Look for templates checking `metadata.labels`
- Resource limits → Look for templates checking `resources.limits`
- Image validation → Look for templates checking `container.image`

### Step 3: Generate ConstraintTemplate

Create a complete ConstraintTemplate following this structure:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: <lowercase-policy-name>
  annotations:
    description: "<Human-readable description>"
spec:
  crd:
    spec:
      names:
        kind: <CamelCasePolicyName>
      validation:
        openAPIV3Schema:  # Only if parameters needed
          type: object
          properties:
            <param-name>:
              type: <type>
              description: "<description>"
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package <package-name>

        # Import future keywords for cleaner syntax
        import future.keywords.if
        import future.keywords.in

        # Violation rule with helpful message
        violation[{"msg": msg, "details": details}] if {
          # Condition logic here
          msg := sprintf("Helpful message with context: %v", [value])
          details := {"field": "value"}
        }
```

### Step 4: Generate Constraint Instance

Create the Constraint that uses the template:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: <CamelCasePolicyName>
metadata:
  name: <instance-name>
spec:
  enforcementAction: dryrun  # Start with dryrun for testing
  match:
    kinds:
      - apiGroups: ["<group>"]
        kinds: ["<Kind>"]
    namespaces: ["<namespace>"]  # Optional
    excludedNamespaces: ["kube-system", "gatekeeper-system"]  # Optional
  parameters:  # If template accepts parameters
    <param-name>: <value>
```

### Step 5: Provide Testing Instructions

Include commands to:
1. Apply the ConstraintTemplate
2. Verify template creation
3. Apply the Constraint in dryrun mode
4. Check for violations
5. Review and adjust
6. Enable enforcement (change to `deny`)

## Common Rego Patterns Library

### Pattern: Required Labels

```rego
package requiredlabels

import future.keywords.if

required_labels := ["team", "environment", "app"]

violation[{"msg": msg}] if {
  missing := {label | label := required_labels[_]; not input.review.object.metadata.labels[label]}
  count(missing) > 0
  msg := sprintf("Missing required labels: %v", [missing])
}
```

### Pattern: Resource Limits

```rego
package resourcelimits

import future.keywords.if

violation[{"msg": msg}] if {
  container := input.review.object.spec.containers[_]
  not container.resources.limits.memory
  msg := sprintf("Container '%s' must specify memory limit", [container.name])
}
```

### Pattern: Image Registry Enforcement

```rego
package approvedregistry

import future.keywords.if

approved_registries := ["docker.io", "quay.io", "gcr.io"]

violation[{"msg": msg}] if {
  container := input.review.object.spec.containers[_]
  image := container.image
  not startswith_any(image, approved_registries)
  msg := sprintf("Container '%s' uses unapproved registry: %s", [container.name, image])
}

startswith_any(str, prefixes) if {
  startswith(str, prefixes[_])
}
```

### Pattern: Replica Count for Production

```rego
package productionreplicas

import future.keywords.if

is_production if {
  input.review.object.metadata.labels.environment == "production"
}

violation[{"msg": msg}] if {
  is_production
  replicas := object.get(input.review.object.spec, "replicas", 1)
  replicas < 2
  msg := "Production deployments must have at least 2 replicas for high availability"
}
```

### Pattern: Namespace Labels

```rego
package namespacelabels

import future.keywords.if

violation[{"msg": msg}] if {
  input.review.kind.kind == "Namespace"
  not input.review.object.metadata.labels.owner
  msg := "All namespaces must have an 'owner' label"
}
```

### Pattern: Security Context

```rego
package requiresecuritycontext

import future.keywords.if

violation[{"msg": msg}] if {
  container := input.review.object.spec.containers[_]
  not container.securityContext.runAsNonRoot
  msg := sprintf("Container '%s' must set runAsNonRoot to true", [container.name])
}
```

### Pattern: ConfigMap/Secret References

```rego
package requireconfigvalidation

import future.keywords.if

violation[{"msg": msg}] if {
  container := input.review.object.spec.containers[_]
  env_var := container.env[_]
  env_var.valueFrom.configMapKeyRef
  not configmap_exists(env_var.valueFrom.configMapKeyRef.name)
  msg := sprintf("ConfigMap '%s' referenced but not found", [env_var.valueFrom.configMapKeyRef.name])
}

# Note: This requires external data provider or pre-sync
configmap_exists(name) if {
  # Implementation depends on cluster configuration
  false  # Placeholder
}
```

## Debugging Workflow

When troubleshooting policy issues:

### Check Template Status

```bash
# Get template
kubectl get constrainttemplate <name> -o yaml

# Check for compilation errors
kubectl get constrainttemplate <name> -o jsonpath='{.status.byPod[*].errors}'
```

### Check Constraint Status

```bash
# Get constraint
kubectl get <constraint-kind> <name> -o yaml

# Check violations
kubectl get <constraint-kind> <name> -o jsonpath='{.status.violations}'

# Count violations
kubectl get <constraint-kind> <name> -o jsonpath='{.status.totalViolations}'
```

### View Gatekeeper Logs

```bash
# Controller logs (admission webhook)
kubectl logs -n gatekeeper-system -l control-plane=controller-manager --tail=100

# Audit logs (periodic scans)
kubectl logs -n gatekeeper-system -l control-plane=audit-controller --tail=100
```

### Test Policy Locally

Use OPA CLI to test Rego:

```bash
# Save Rego to file
cat > policy.rego <<'EOF'
package test

violation[{"msg": msg}] {
  not input.metadata.labels.team
  msg := "Missing team label"
}
EOF

# Create test input
cat > input.json <<'EOF'
{
  "metadata": {
    "name": "test-pod",
    "labels": {
      "app": "myapp"
    }
  }
}
EOF

# Test with OPA
opa eval -d policy.rego -i input.json 'data.test.violation'
```

## Analysis Output Format

When presenting cluster analysis, structure output as:

```markdown
## Cluster Analysis: <context-name>

### Overview
- **Context**: <context-name>
- **Total CRDs**: <count>
- **Namespaces**: <count>
- **Gatekeeper**: <installed/not-installed>

### CRD Summary by Group
- **gatekeeper.sh**: <count> CRDs (policy enforcement)
- **openshift.io**: <count> CRDs (OpenShift platform)
- **<other-group>**: <count> CRDs

### Policy Configuration (if Gatekeeper installed)

#### ConstraintTemplates (<count>)
1. **<template-name>**
   - Kind: <Kind>
   - Purpose: <description>
   - Rego: <brief-summary>

#### Active Constraints (<count>)
1. **<constraint-name>** (<Kind>)
   - Enforcement: <deny/dryrun/warn>
   - Violations: <count>
   - Scope: <namespaces/kinds>

### Key Findings
- Notable CRDs for policy generation
- Existing policy patterns
- Potential policy gaps
- Recommended actions
```

## OpenShift-Specific Considerations

When analyzing OpenShift clusters:

### OpenShift CRDs to Check

```bash
# List OpenShift-specific CRDs
kubectl get crd | grep openshift.io

# Common OpenShift CRDs:
# - routes.route.openshift.io (routing/ingress)
# - builds.build.openshift.io (builds)
# - deploymentconfigs.apps.openshift.io (deployments)
# - imagestreams.image.openshift.io (container images)
# - projects.project.openshift.io (namespaces with quotas)
```

### OpenShift Security Context Constraints (SCC)

```bash
# List SCCs
oc get scc

# Get specific SCC
oc get scc restricted-v2 -o yaml
```

### OpenShift Routes

```bash
# List routes
oc get routes -A
```

## Real-Time Capabilities

This skill leverages the MCP server to provide:

 **Live Cluster State** - Always current, never stale
 **Multi-Cluster Support** - Works with any configured context
 **Dynamic Discovery** - Adapts to any CRD schema
 **Cross-Platform** - Kubernetes, OpenShift, GKE, EKS, AKS
 **No Pre-Configuration** - Discovers cluster state on-demand

## Example Usage Scenarios

### Scenario 1: New Cluster Analysis

**User**: "Analyze my cluster's policy configuration"

**Claude**:
1. Gets current context
2. Lists all CRDs
3. Identifies Gatekeeper installation
4. Lists ConstraintTemplates and Constraints
5. Summarizes findings
6. Suggests policy improvements

### Scenario 2: Generate Specific Policy

**User**: "Create a policy to require team labels on all pods"

**Claude**:
1. Checks existing label policies
2. Generates ConstraintTemplate with Rego
3. Creates Constraint in dryrun mode
4. Provides testing commands
5. Explains enforcement steps

### Scenario 3: Troubleshoot Violations

**User**: "Why is my deployment being rejected?"

**Claude**:
1. Lists active constraints
2. Checks violation messages
3. Retrieves relevant ConstraintTemplate
4. Analyzes Rego logic
5. Explains why resource was rejected
6. Suggests fixes

### Scenario 4: Compare Clusters

**User**: "Show me the differences between dev and prod policies"

**Claude**:
1. Switches to dev context, analyzes
2. Switches to prod context, analyzes
3. Compares ConstraintTemplates
4. Identifies missing/different policies
5. Suggests alignment

## Best Practices

### 1. Start with Discovery
Always begin by understanding what's already in the cluster before creating new policies.

### 2. Use Dryrun First
New constraints should start with `enforcementAction: dryrun` to assess impact.

### 3. Provide Context in Messages
Violation messages should include resource name, namespace, and actionable guidance.

### 4. Scope Appropriately
Exclude system namespaces unless specifically required.

### 5. Test Locally
Use OPA CLI to test Rego before deploying to cluster.

### 6. Document Policies
Add descriptions in annotations explaining policy intent.

### 7. Version Control
Store ConstraintTemplates and Constraints in Git.

## Commands Reference

```bash
# Cluster context
kubectl config current-context
kubectl config get-contexts

# CRDs
kubectl get crds
kubectl get crd <name> -o yaml

# Gatekeeper
kubectl get constrainttemplates
kubectl get constrainttemplate <name> -o yaml
kubectl get constraints -A

# Specific constraint type
kubectl get <kind>.constraints.gatekeeper.sh -A
kubectl get <kind> <name> -o yaml

# Logs
kubectl logs -n gatekeeper-system -l control-plane=controller-manager
kubectl logs -n gatekeeper-system -l control-plane=audit-controller

# Configuration
kubectl get config -n gatekeeper-system config -o yaml

# Sync status
kubectl get configpodstatus -n gatekeeper-system
kubectl get constraintpodstatus -n gatekeeper-system
```

## Integration with Team Workflow

### For Team Members

1. **Set up MCP server** (one-time):
   - Follow setup guide in project README
   - Configure `.claude/.mcp.json`

2. **Access this skill**:
   - Skill is available in `.claude/skills/cluster-policy-analyzer/`
   - Invoked automatically when relevant

3. **Switch contexts**:
   ```bash
   kubectl config use-context <cluster-name>
   ```

4. **Invoke skill**:
   - "Analyze my cluster"
   - "Generate a policy for..."
   - "What CRDs are in my cluster?"

### For Administrators

- Store this skill in version control
- Share with team via Git repository
- Update patterns as new requirements emerge
- Maintain library of common policies

---

**Version**: 1.0.0
**Maintained By**: Platform Team
**Last Updated**: 2026-01-08
**Supports**: Kubernetes 1.20+, OpenShift 4.x, Gatekeeper 3.x
