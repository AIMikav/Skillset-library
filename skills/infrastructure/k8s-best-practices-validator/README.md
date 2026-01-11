# Kubernetes Best Practices Validator Skill

A Claude Code skill that validates Kubernetes/OpenShift clusters against Red Hat best practices and automatically generates OPA Gatekeeper policies to enforce compliance.

## Overview

This skill enables Claude to:
- Analyze cluster resources against 30+ curated Red Hat best practices
- Identify violations grouped by category (Security, Resource Management, Networking, etc.)
- Generate ready-to-deploy Gatekeeper policies (ConstraintTemplates + Constraints)
- Save policies as YAML files organized by category
- Provide detailed compliance reports and remediation guidance

## Quick Start

### Prerequisites

1. **kubectl** or **oc** CLI installed and configured
2. **Active cluster context** pointing to your cluster
3. **Kubernetes MCP Server** configured in the project (already set up in `.claude/.mcp.json`)

### Verify Setup

```bash
# Check cluster access
kubectl cluster-info

# Verify current context
kubectl config current-context

# Test MCP server (optional)
kubectl get namespaces
```

### Start Using the Skill

```bash
# Navigate to repository directory
cd Skillset-library

# Start Claude Code
claude
```

The skill is automatically available when Claude Code starts in this directory.

## Usage Examples

### Example 1: Validate Entire Cluster

```
You: Validate my cluster against Red Hat best practices
```

Claude will:
1. List all namespaces (excluding system namespaces by default)
2. Analyze pods, deployments, statefulsets, etc.
3. Check against 30+ best practices
4. Generate a violation report grouped by category
5. Offer to create Gatekeeper policies

### Example 2: Validate Specific Namespaces

```
You: Check best practices violations in app-prod and app-staging namespaces
```

Claude will analyze only the specified namespaces.

### Example 3: Generate Policies

```
You: Generate Gatekeeper policies for the violations found
```

Claude will:
1. Create ConstraintTemplates for each violated best practice
2. Create Constraint instances
3. Save files organized by category
4. Provide deployment instructions

### Example 4: Security-Focused Analysis

```
You: Analyze security best practices in my production namespace
```

Claude will focus on security-related violations.

## How It Works

### Phase 1: Gather Cluster Information

Claude uses the Kubernetes MCP server to retrieve:
- Current context
- Namespaces (user-specified or all non-system)
- Pods, Deployments, StatefulSets, DaemonSets, Jobs, CronJobs
- NetworkPolicies

**MCP Tools Used:**
- `mcp__kubernetes__configuration_view()`
- `mcp__kubernetes__namespaces_list()`
- `mcp__kubernetes__pods_list_in_namespace()`
- `mcp__kubernetes__resources_list()`

### Phase 2: Analyze Against Best Practices

For each resource, Claude checks applicable best practices from the catalog:
- Security (non-root, capabilities, privilege escalation, etc.)
- Resource Management (limits, requests)
- Networking (network policies, host network)
- Storage (no hostPath)
- Container Images (approved registries, no :latest)
- Pod Configuration (probes, replicas)
- Workload Best Practices (labels, anti-affinity)

### Phase 3: Generate Violation Report

Claude presents findings grouped by category:

```markdown
## Best Practices Validation Report

**Cluster:** minikube
**Analyzed Namespaces:** app-prod, app-staging
**Timestamp:** 2026-01-11T14:30:00Z

### Summary
- Total Violations: 42
- Critical: 5
- High: 18
- Medium: 12
- Low: 7

### Violations by Category

#### Security (15 violations)
1. BP-SEC-001: Run Containers as Non-Root (8 resources)
   - app-prod/Deployment/web-server
   - app-staging/Deployment/api-service
   ...

#### Resource Management (12 violations)
...
```

### Phase 4: Create Rego Policies

Claude generates OPA Gatekeeper policies:
- **ConstraintTemplates** with Rego validation logic
- **Constraints** with enforcement configuration (dryrun by default)
- Organized by category for easy navigation

### Phase 5: Save to Disk

Files are saved to:
```
skills/infrastructure/k8s-best-practices-validator/output/
├── README.md (deployment guide)
├── constraint-templates/
│   ├── security/
│   │   ├── k8srequirenonroot.yaml
│   │   ├── k8sdropcapabilities.yaml
│   │   └── ...
│   ├── resource-management/
│   ├── networking/
│   └── ...
└── constraints/
    ├── security/
    │   ├── require-nonroot-containers.yaml
    │   └── ...
    └── ...
```

## Best Practices Catalog

The skill includes 30+ curated Red Hat best practices:

### Security (6 practices)
- **BP-SEC-001:** Run containers as non-root
- **BP-SEC-002:** Drop all capabilities
- **BP-SEC-003:** No privilege escalation
- **BP-SEC-004:** No privileged containers
- **BP-SEC-005:** Use RuntimeDefault seccomp profile
- **BP-SEC-006:** Read-only root filesystem

### Resource Management (4 practices)
- **BP-RES-001:** Memory limits required
- **BP-RES-002:** Memory requests required
- **BP-RES-003:** CPU requests required
- **BP-RES-004:** No CPU limits (avoid throttling)

### Networking (3 practices)
- **BP-NET-001:** Network policies required
- **BP-NET-002:** No host network
- **BP-NET-003:** No host port binding

### Storage (1 practice)
- **BP-STO-001:** No hostPath volumes in production

### Container Images (2 practices)
- **BP-IMG-001:** Use approved registries
- **BP-IMG-002:** No :latest tag

### Pod Configuration (3 practices)
- **BP-POD-001:** Liveness probe required
- **BP-POD-002:** Readiness probe required
- **BP-POD-003:** Minimum replica count for production

### Workload Best Practices (1 practice)
- **BP-WRK-001:** Required labels (app, team, environment)

Each practice includes:
- Description and rationale
- Validation logic
- Rego template for policy generation
- Remediation guidance


### When to Use Which Skill

| Task | Skill |
|------|-------|
| "What policies are installed?" | cluster-policy-analyzer |
| "Show existing ConstraintTemplates" | cluster-policy-analyzer |
| "Validate against best practices" | k8s-best-practices-validator |
| "Generate compliance policies" | k8s-best-practices-validator |
| "Debug policy violations" | cluster-policy-analyzer |
| "Compare dev and prod policies" | cluster-policy-analyzer |

## Troubleshooting

### MCP Server Not Working

**Symptom:** Claude can't execute kubectl commands

**Solutions:**
1. Verify `.claude/.mcp.json` exists in project root
2. Restart Claude Code session
3. Check kubectl is in PATH: `which kubectl`
4. Test MCP manually: `npx -y kubernetes-mcp-server@latest`

### Cluster Access Issues

**Symptom:** kubectl commands fail with connection errors

**Solutions:**
1. Verify cluster is running: `kubectl cluster-info`
2. Check context: `kubectl config current-context`
3. Validate credentials haven't expired
4. For OpenShift: `oc login` if needed

### No Violations Found (Unexpected)

**Symptom:** Report shows 0 violations but you expect some

**Solutions:**
1. Check analyzed namespaces (might be excluding target namespaces)
2. Verify resources exist in analyzed namespaces
3. Confirm MCP retrieved resources successfully

### Skill Not Activating

**Symptom:** Claude doesn't use the skill

**Solutions:**
1. Use trigger phrases: "validate best practices", "compliance check"
2. Explicitly ask: "Use the k8s-best-practices-validator skill"
3. Verify skill file exists: `ls skills/infrastructure/k8s-best-practices-validator/SKILL.md`

## Version History

- **v1.0.0** (2026-01-11)

---

**License:** Internal Use
**Last Updated:** 2026-01-11
