---
name: k8s-best-practices-validator
description: Validates Kubernetes/OpenShift clusters against Red Hat best practices and generates OPA Gatekeeper policies to enforce compliance
version: 1.0.0
author: Platform Team
category: infrastructure
prerequisites:
  - Kubernetes MCP Server
  - kubectl with cluster access
  - Active cluster context
---

# Kubernetes Best Practices Validator

You are a Kubernetes/OpenShift cluster validation assistant that analyzes clusters against Red Hat best practices and generates OPA Gatekeeper policies to enforce compliance.

## Capabilities

1. **Cluster Analysis**: Analyze cluster resources against 30+ curated Red Hat best practices
2. **Violation Detection**: Identify non-compliant resources grouped by category
3. **Policy Generation**: Create ConstraintTemplates and Constraints for Gatekeeper
4. **Compliance Reporting**: Generate detailed violation reports
5. **Multi-Namespace Support**: Analyze user-specified namespaces or entire cluster

## When to Use This Skill

Invoke this skill when the user asks:
- "Validate my cluster against best practices"
- "Check for OpenShift/Red Hat best practices violations"
- "Generate policies for compliance"
- "Analyze cluster compliance"
- "What best practice violations exist in namespace X?"
- "Create Gatekeeper policies from best practices"

## Prerequisites

This skill requires:
1. **Kubernetes MCP Server** configured in `.claude/.mcp.json`
2. **kubectl** with access to your cluster
3. **Active cluster context** (minikube, OpenShift, GKE, EKS, etc.)

## Kubernetes MCP Server Integration

**CRITICAL:** This skill MUST use the available Kubernetes MCP server tools for all cluster interactions:

### Available MCP Tools:
- `mcp__kubernetes__configuration_view(minified=true)` - Get current context
- `mcp__kubernetes__namespaces_list()` - List all namespaces
- `mcp__kubernetes__pods_list()` - List pods across all namespaces
- `mcp__kubernetes__pods_list_in_namespace(namespace="ns")` - List pods in specific namespace
- `mcp__kubernetes__pods_get(name="pod", namespace="ns")` - Get pod details
- `mcp__kubernetes__resources_list(apiVersion="v1", kind="Kind", namespace="ns")` - List any resources
- `mcp__kubernetes__resources_get(apiVersion="v1", kind="Kind", name="resource", namespace="ns")` - Get resource details

## 5-Phase Validation Workflow

When invoked, follow this systematic workflow:

### Phase 1: Gather Cluster Information

**IMPORTANT: First, explicitly announce to the user: "Using the Kubernetes Best Practices Validator skill to analyze your cluster."**

1. **Get Current Context:**
   ```
   Use mcp__kubernetes__configuration_view(minified=true)
   ```
   Purpose: Confirm which cluster is being analyzed

2. **List Namespaces:**
   ```
   Use mcp__kubernetes__namespaces_list()
   ```

3. **Determine Namespaces to Analyze:**
   - If user specified namespaces: Use those
   - Default: All namespaces EXCEPT system namespaces:
     - kube-system
     - gatekeeper-system
     - openshift-* (all openshift namespaces)
     - default (optional, ask user)

4. **Retrieve Resources from Each Namespace:**

   For each namespace, gather:

   a. **Pods:**
   ```
   mcp__kubernetes__pods_list_in_namespace(namespace=ns)
   ```

   b. **Deployments:**
   ```
   mcp__kubernetes__resources_list(
     apiVersion="apps/v1",
     kind="Deployment",
     namespace=ns
   )
   ```

   c. **StatefulSets:**
   ```
   mcp__kubernetes__resources_list(
     apiVersion="apps/v1",
     kind="StatefulSet",
     namespace=ns
   )
   ```

   d. **DaemonSets:**
   ```
   mcp__kubernetes__resources_list(
     apiVersion="apps/v1",
     kind="DaemonSet",
     namespace=ns
   )
   ```

   e. **Jobs:**
   ```
   mcp__kubernetes__resources_list(
     apiVersion="batch/v1",
     kind="Job",
     namespace=ns
   )
   ```

   f. **CronJobs:**
   ```
   mcp__kubernetes__resources_list(
     apiVersion="batch/v1",
     kind="CronJob",
     namespace=ns
   )
   ```

   g. **NetworkPolicies:**
   ```
   mcp__kubernetes__resources_list(
     apiVersion="networking.k8s.io/v1",
     kind="NetworkPolicy",
     namespace=ns
   )
   ```

5. **Store resources in memory** for analysis in Phase 2

### Phase 2: Analyze Against Best Practices

For each resource retrieved:

1. Determine which best practices apply based on resource kind
2. For each applicable best practice:
   - Extract the field value from the resource
   - Compare against expected value
   - If violation detected, record:
     - Best practice ID (e.g., BP-SEC-001)
     - Resource identifier (namespace/kind/name)
     - Category
     - Severity
     - Current value
     - Expected value
     - Remediation guidance

3. Build violations list grouped by category

### Phase 3: Generate Violation Report

Present findings to the user in this format:

```markdown
## Best Practices Validation Report

**Cluster:** {context-name}
**Analyzed Namespaces:** {namespace-list}
**Timestamp:** {ISO-8601 timestamp}

### Summary
- **Total Violations:** {count}
- **Critical:** {count}
- **High:** {count}
- **Medium:** {count}
- **Low:** {count}

### Violations by Category

#### Security ({count} violations)
1. **BP-SEC-001: Run Containers as Non-Root** ({count} resources)
   - `{namespace}/{kind}/{name}`
   - `{namespace}/{kind}/{name}`
   - ... (show first 5, then "X more")

2. **BP-SEC-002: Drop All Capabilities** ({count} resources)
   - `{namespace}/{kind}/{name}`

#### Resource Management ({count} violations)
...

#### Networking ({count} violations)
...

### Recommendations

{Provide actionable recommendations based on findings}

---

Would you like me to generate Gatekeeper policies to enforce these best practices?
```

### Phase 4: Create Rego Policies

If user approves policy generation:

1. **Ask for Confirmation:**
   - Enforcement action: dryrun (default, recommended) | deny | warn
   - Target namespaces: (confirm with user)

2. **For Each Unique Violated Best Practice:**

   a. Generate ConstraintTemplate YAML using the Rego template from the Best Practices Catalog below

   b. Generate Constraint instance YAML:
   ```yaml
   apiVersion: constraints.gatekeeper.sh/v1beta1
   kind: {ConstraintKind}
   metadata:
     name: {constraint-name}
   spec:
     enforcementAction: dryrun  # or user-specified
     match:
       kinds:
         - apiGroups: {applicable-groups}
           kinds: {applicable-kinds}
       namespaces: {user-specified or analyzed namespaces}
       excludedNamespaces:
         - kube-system
         - gatekeeper-system
         - openshift-apiserver
         - openshift-authentication
         - openshift-config
     parameters: {if needed}
   ```

3. **Organize by Category:**
   - Security policies → security/
   - Resource management → resource-management/
   - Networking → networking/
   - Storage → storage/
   - Container images → container-images/
   - Pod configuration → pod-configuration/
   - Workload best practices → workload-best-practices/

### Phase 5: Save to Disk

1. **Create Output Directory Structure:**
   ```
   skills/infrastructure/k8s-best-practices-validator/output/
   ├── README.md
   ├── constraint-templates/
   │   ├── security/
   │   ├── resource-management/
   │   ├── networking/
   │   ├── storage/
   │   ├── container-images/
   │   ├── pod-configuration/
   │   └── workload-best-practices/
   └── constraints/
       └── [same categories]
   ```

2. **Save ConstraintTemplates:**
   - File naming: `{template-name}.yaml` (e.g., `k8srequirenonroot.yaml`)
   - Location: `output/constraint-templates/{category}/`

3. **Save Constraints:**
   - File naming: `{descriptive-name}.yaml` (e.g., `require-nonroot-containers.yaml`)
   - Location: `output/constraints/{category}/`

4. **Generate output/README.md** with:
   - Summary of generated policies
   - Deployment instructions (kubectl apply commands)
   - Testing guidance (dryrun → enforcement)
   - How to check violations

5. **Report to User:**
   ```
   Generated policies saved to:
   {absolute-path}/skills/infrastructure/k8s-best-practices-validator/output/

   Summary:
   - ConstraintTemplates: {count}
   - Constraints: {count}
   - Categories: {list}

   Next steps:
   1. Review files in output/
   2. Read output/README.md for deployment instructions
   3. Apply templates: kubectl apply -f constraint-templates/
   4. Apply constraints: kubectl apply -f constraints/
   5. Check violations before enforcing
   ```

---

## Red Hat Best Practices Catalog

This catalog contains 30+ curated best practices across 7 categories. Each entry includes validation logic and Rego templates for policy generation.

### Category 1: Security (BP-SEC-XXX)

#### BP-SEC-001: Run Containers as Non-Root

**Severity:** High
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
Containers should not run as the root user (UID 0) to minimize attack surface. Running as root increases security risks if a container is compromised.

**Validation:**
- Field Path: `spec.containers[*].securityContext.runAsNonRoot`
- Expected Value: `true`
- Also check: `spec.initContainers[*].securityContext.runAsNonRoot`

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirenonroot
  annotations:
    description: "Requires containers to run as non-root user"
spec:
  crd:
    spec:
      names:
        kind: K8sRequireNonRoot
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirenonroot

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot == true
          msg := sprintf("Container '%s' must set runAsNonRoot to true", [container.name])
          details := {"container": container.name, "type": "container"}
        }

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.initContainers[_]
          not container.securityContext.runAsNonRoot == true
          msg := sprintf("InitContainer '%s' must set runAsNonRoot to true", [container.name])
          details := {"container": container.name, "type": "initContainer"}
        }
```

**Constraint Example:** `require-nonroot-containers.yaml`

**Remediation:**
```yaml
spec:
  containers:
  - name: myapp
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000  # Non-root UID
```

---

#### BP-SEC-002: Drop All Capabilities

**Severity:** High
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
Linux capabilities should be dropped by default and only added back as needed. This follows the principle of least privilege.

**Validation:**
- Field Path: `spec.containers[*].securityContext.capabilities.drop`
- Expected Value: Must contain "ALL"

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdropcapabilities
  annotations:
    description: "Requires containers to drop all capabilities"
spec:
  crd:
    spec:
      names:
        kind: K8sDropCapabilities
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdropcapabilities

        import future.keywords.if
        import future.keywords.in

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          not has_drop_all(container)
          msg := sprintf("Container '%s' must drop all capabilities (capabilities.drop: [ALL])", [container.name])
          details := {"container": container.name}
        }

        has_drop_all(container) if {
          "ALL" in container.securityContext.capabilities.drop
        }
```

**Remediation:**
```yaml
securityContext:
  capabilities:
    drop:
      - ALL
    add:  # Only add what's specifically needed
      - NET_BIND_SERVICE
```

---

#### BP-SEC-003: No Privilege Escalation

**Severity:** High
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
Containers should not allow privilege escalation to prevent attackers from gaining elevated permissions.

**Validation:**
- Field Path: `spec.containers[*].securityContext.allowPrivilegeEscalation`
- Expected Value: `false`

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoprivilegeescalation
  annotations:
    description: "Prevents privilege escalation in containers"
spec:
  crd:
    spec:
      names:
        kind: K8sNoPrivilegeEscalation
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snoprivilegeescalation

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          not container.securityContext.allowPrivilegeEscalation == false
          msg := sprintf("Container '%s' must set allowPrivilegeEscalation to false", [container.name])
          details := {"container": container.name}
        }
```

**Remediation:**
```yaml
securityContext:
  allowPrivilegeEscalation: false
```

---

#### BP-SEC-004: No Privileged Containers

**Severity:** Critical
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
Containers should never run in privileged mode as it grants access to all host devices.

**Validation:**
- Field Path: `spec.containers[*].securityContext.privileged`
- Expected Value: `false` or not set

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoprivilegedcontainers
  annotations:
    description: "Prevents privileged containers"
spec:
  crd:
    spec:
      names:
        kind: K8sNoPrivilegedContainers
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snoprivilegedcontainers

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Container '%s' cannot run in privileged mode", [container.name])
          details := {"container": container.name}
        }
```

**Remediation:**
```yaml
securityContext:
  privileged: false  # or remove this field
```

---

#### BP-SEC-005: Use RuntimeDefault Seccomp Profile

**Severity:** Medium
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
Use the RuntimeDefault seccomp profile to restrict syscalls and reduce attack surface.

**Validation:**
- Field Path: `spec.securityContext.seccompProfile.type`
- Expected Value: `RuntimeDefault`

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequireseccomp
  annotations:
    description: "Requires RuntimeDefault seccomp profile"
spec:
  crd:
    spec:
      names:
        kind: K8sRequireSeccomp
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequireseccomp

        import future.keywords.if

        violation[{"msg": msg}] if {
          not input.review.object.spec.securityContext.seccompProfile.type == "RuntimeDefault"
          msg := "Pod must use RuntimeDefault seccomp profile"
        }
```

**Remediation:**
```yaml
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
```

---

#### BP-SEC-006: Read-Only Root Filesystem

**Severity:** Medium
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
Container root filesystem should be read-only to prevent malicious code execution.

**Validation:**
- Field Path: `spec.containers[*].securityContext.readOnlyRootFilesystem`
- Expected Value: `true`

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sreadonlyrootfs
  annotations:
    description: "Requires read-only root filesystem"
spec:
  crd:
    spec:
      names:
        kind: K8sReadOnlyRootFS
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sreadonlyrootfs

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          not container.securityContext.readOnlyRootFilesystem == true
          msg := sprintf("Container '%s' should have readOnlyRootFilesystem set to true", [container.name])
          details := {"container": container.name}
        }
```

**Remediation:**
```yaml
securityContext:
  readOnlyRootFilesystem: true
# Use emptyDir or volumes for writable directories
volumes:
- name: tmp
  emptyDir: {}
```

---

### Category 2: Resource Management (BP-RES-XXX)

#### BP-RES-001: Memory Limits Required

**Severity:** High
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
All containers must specify memory limits to prevent resource exhaustion.

**Validation:**
- Field Path: `spec.containers[*].resources.limits.memory`
- Expected Value: Must be set (non-empty)

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirememorylimits
  annotations:
    description: "Requires memory limits on containers"
spec:
  crd:
    spec:
      names:
        kind: K8sRequireMemoryLimits
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirememorylimits

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%s' must specify memory limit", [container.name])
          details := {"container": container.name, "field": "resources.limits.memory"}
        }
```

**Remediation:**
```yaml
resources:
  limits:
    memory: "512Mi"
  requests:
    memory: "256Mi"
```

---

#### BP-RES-002: Memory Requests Required

**Severity:** High
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
All containers must specify memory requests for proper scheduling.

**Validation:**
- Field Path: `spec.containers[*].resources.requests.memory`
- Expected Value: Must be set (non-empty)

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirememoryrequests
  annotations:
    description: "Requires memory requests on containers"
spec:
  crd:
    spec:
      names:
        kind: K8sRequireMemoryRequests
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirememoryrequests

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          not container.resources.requests.memory
          msg := sprintf("Container '%s' must specify memory request", [container.name])
          details := {"container": container.name, "field": "resources.requests.memory"}
        }
```

**Remediation:**
```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

---

#### BP-RES-003: CPU Requests Required

**Severity:** Medium
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
All containers must specify CPU requests for proper scheduling.

**Validation:**
- Field Path: `spec.containers[*].resources.requests.cpu`
- Expected Value: Must be set (non-empty)

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirecpurequests
  annotations:
    description: "Requires CPU requests on containers"
spec:
  crd:
    spec:
      names:
        kind: K8sRequireCPURequests
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirecpurequests

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          not container.resources.requests.cpu
          msg := sprintf("Container '%s' must specify CPU request", [container.name])
          details := {"container": container.name, "field": "resources.requests.cpu"}
        }
```

**Remediation:**
```yaml
resources:
  requests:
    cpu: "100m"
```

---

#### BP-RES-004: No CPU Limits (Avoid Throttling)

**Severity:** Low
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
Red Hat recommends NOT setting CPU limits to avoid CPU throttling issues that can impact performance.

**Validation:**
- Field Path: `spec.containers[*].resources.limits.cpu`
- Expected Value: Should NOT be set

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snocpulimits
  annotations:
    description: "Disallows CPU limits to prevent throttling (Red Hat best practice)"
spec:
  crd:
    spec:
      names:
        kind: K8sNoCPULimits
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snocpulimits

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          container.resources.limits.cpu
          msg := sprintf("Container '%s' has CPU limit set (%v). Red Hat recommends not setting CPU limits to avoid throttling.", [container.name, container.resources.limits.cpu])
          details := {"container": container.name, "cpu_limit": container.resources.limits.cpu}
        }
```

**Remediation:**
```yaml
# Remove CPU limits, keep only requests
resources:
  requests:
    cpu: "100m"
  # limits.cpu removed
```

---

### Category 3: Networking (BP-NET-XXX)

#### BP-NET-001: Network Policies Required

**Severity:** High
**Applies To:** Namespace

**Description:**
Every namespace should have at least one NetworkPolicy to implement network segmentation and zero-trust networking.

**Validation:**
- Check if namespace has at least one NetworkPolicy resource
- This requires cross-resource validation (audit mode or external data)

**Note:** This best practice is checked differently - by verifying NetworkPolicy existence in the namespace, not by analyzing individual pod specs.

**Implementation:**
During analysis, for each namespace, check:
```
policies = mcp__kubernetes__resources_list(
  apiVersion="networking.k8s.io/v1",
  kind="NetworkPolicy",
  namespace=ns
)

if len(policies) == 0:
  # Violation: namespace has no NetworkPolicy
```

**Remediation:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: my-namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

---

#### BP-NET-002: No Host Network

**Severity:** High
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
Pods should not use the host network namespace as it bypasses network policies.

**Validation:**
- Field Path: `spec.hostNetwork`
- Expected Value: `false` or not set

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snohostnetwork
  annotations:
    description: "Prevents use of host network"
spec:
  crd:
    spec:
      names:
        kind: K8sNoHostNetwork
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snohostnetwork

        import future.keywords.if

        violation[{"msg": msg}] if {
          input.review.object.spec.hostNetwork == true
          msg := "Pods must not use host network (spec.hostNetwork must be false)"
        }
```

**Remediation:**
```yaml
spec:
  hostNetwork: false  # or remove this field
```

---

#### BP-NET-003: No Host Port Binding

**Severity:** High
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
Containers should not bind to host ports as it can cause port conflicts and security issues.

**Validation:**
- Field Path: `spec.containers[*].ports[*].hostPort`
- Expected Value: Should NOT be set

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snohostport
  annotations:
    description: "Prevents containers from binding to host ports"
spec:
  crd:
    spec:
      names:
        kind: K8sNoHostPort
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snohostport

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          port := container.ports[_]
          port.hostPort
          msg := sprintf("Container '%s' must not bind to host port %v", [container.name, port.hostPort])
          details := {"container": container.name, "hostPort": port.hostPort}
        }
```

**Remediation:**
```yaml
ports:
- containerPort: 8080
  # hostPort removed
```

---

### Category 4: Storage (BP-STO-XXX)

#### BP-STO-001: No hostPath Volumes in Production

**Severity:** High
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
hostPath volumes provide access to the host filesystem and should not be used in production.

**Validation:**
- Field Path: `spec.volumes[*].hostPath`
- Expected Value: Should NOT be present

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snohostpath
  annotations:
    description: "Prevents use of hostPath volumes"
spec:
  crd:
    spec:
      names:
        kind: K8sNoHostPath
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snohostpath

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          msg := sprintf("Volume '%s' uses hostPath which is not allowed in production", [volume.name])
          details := {"volume": volume.name, "hostPath": volume.hostPath.path}
        }
```

**Remediation:**
```yaml
# Use PersistentVolumeClaims instead
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

---

### Category 5: Container Images (BP-IMG-XXX)

#### BP-IMG-001: Use Approved Registries

**Severity:** Critical
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
Container images must be pulled from approved, trusted registries (e.g., registry.redhat.io, quay.io).

**Validation:**
- Field Path: `spec.containers[*].image`
- Expected Value: Must start with approved registry prefix

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sapprovedregistries
  annotations:
    description: "Requires container images from approved registries"
spec:
  crd:
    spec:
      names:
        kind: K8sApprovedRegistries
      validation:
        openAPIV3Schema:
          type: object
          properties:
            registries:
              type: array
              items:
                type: string
              description: "List of approved registry prefixes"
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sapprovedregistries

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          image := container.image
          not image_from_approved_registry(image)
          msg := sprintf("Container '%s' uses unapproved registry. Image: %s. Approved registries: %v", [container.name, image, input.parameters.registries])
          details := {"container": container.name, "image": image}
        }

        image_from_approved_registry(image) if {
          registry := input.parameters.registries[_]
          startswith(image, registry)
        }
```

**Constraint Example:**
```yaml
spec:
  parameters:
    registries:
      - "registry.redhat.io/"
      - "quay.io/yourorg/"
      - "registry.access.redhat.com/"
```

**Remediation:**
```yaml
image: registry.redhat.io/ubi8/ubi:latest
```

---

#### BP-IMG-002: No :latest Tag

**Severity:** Medium
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**Description:**
Container images should use specific version tags, not :latest, for reproducibility and security.

**Validation:**
- Field Path: `spec.containers[*].image`
- Expected Value: Must have tag other than "latest" or no tag

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowlatesttag
  annotations:
    description: "Prevents use of :latest image tag"
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowLatestTag
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowlatesttag

        import future.keywords.if
        import future.keywords.contains

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          image := container.image
          endswith(image, ":latest")
          msg := sprintf("Container '%s' uses :latest tag. Image: %s. Use a specific version tag.", [container.name, image])
          details := {"container": container.name, "image": image}
        }

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          image := container.image
          not contains(image, ":")
          msg := sprintf("Container '%s' has no tag (defaults to :latest). Image: %s. Use a specific version tag.", [container.name, image])
          details := {"container": container.name, "image": image}
        }
```

**Remediation:**
```yaml
image: myregistry.com/myapp:v1.2.3  # Specific version
```

---

### Category 6: Pod Configuration (BP-POD-XXX)

#### BP-POD-001: Liveness Probe Required

**Severity:** Medium
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet

**Description:**
Containers should have liveness probes so Kubernetes can restart unhealthy pods.

**Validation:**
- Field Path: `spec.containers[*].livenessProbe`
- Expected Value: Must be set

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirelivenessprobe
  annotations:
    description: "Requires liveness probes on containers"
spec:
  crd:
    spec:
      names:
        kind: K8sRequireLivenessProbe
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirelivenessprobe

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          not container.livenessProbe
          msg := sprintf("Container '%s' must have a livenessProbe configured", [container.name])
          details := {"container": container.name}
        }
```

**Remediation:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

---

#### BP-POD-002: Readiness Probe Required

**Severity:** Medium
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet

**Description:**
Containers should have readiness probes so Kubernetes knows when pods are ready to receive traffic.

**Validation:**
- Field Path: `spec.containers[*].readinessProbe`
- Expected Value: Must be set

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirereadinessprobe
  annotations:
    description: "Requires readiness probes on containers"
spec:
  crd:
    spec:
      names:
        kind: K8sRequireReadinessProbe
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirereadinessprobe

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          container := input.review.object.spec.containers[_]
          not container.readinessProbe
          msg := sprintf("Container '%s' must have a readinessProbe configured", [container.name])
          details := {"container": container.name}
        }
```

**Remediation:**
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

#### BP-POD-003: Minimum Replica Count for Production

**Severity:** Medium
**Applies To:** Deployment, StatefulSet

**Description:**
Production deployments should have at least 2 replicas for high availability.

**Validation:**
- Field Path: `spec.replicas`
- Expected Value: >= 2 (for production environments)

**Note:** This check should look for environment label or namespace to determine if production.

**Remediation:**
```yaml
spec:
  replicas: 2  # or more
```

---

### Category 7: Workload Best Practices (BP-WRK-XXX)

#### BP-WRK-001: Required Labels

**Severity:** Low
**Applies To:** Pod, Deployment, StatefulSet, DaemonSet, Service

**Description:**
All resources should have standard labels for organization and tracking (app, team, environment).

**Validation:**
- Field Path: `metadata.labels`
- Expected Value: Must contain required labels

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
  annotations:
    description: "Requires standard labels on resources"
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
              description: "List of required label keys"
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        import future.keywords.if

        violation[{"msg": msg, "details": details}] if {
          required := input.parameters.labels[_]
          not input.review.object.metadata.labels[required]
          msg := sprintf("Resource must have label '%s'", [required])
          details := {"missing_label": required}
        }
```

**Constraint Example:**
```yaml
spec:
  parameters:
    labels:
      - app
      - team
      - environment
```

**Remediation:**
```yaml
metadata:
  labels:
    app: myapp
    team: platform
    environment: production
```

---

## Rego Pattern Library

Use these reusable patterns when generating policies:

### Pattern 1: Container Iteration

```rego
# Iterate through all containers and initContainers
violation[{"msg": msg}] if {
  container := input.review.object.spec.containers[_]
  # check container
}

violation[{"msg": msg}] if {
  container := input.review.object.spec.initContainers[_]
  # check initContainer
}
```

### Pattern 2: Field Existence Check

```rego
# Check if a field exists and has expected value
violation[{"msg": msg}] if {
  not input.review.object.spec.field == expected_value
  msg := "Field must be set to expected value"
}
```

### Pattern 3: Field Must NOT Exist

```rego
# Check that a field is not set
violation[{"msg": msg}] if {
  input.review.object.spec.field
  msg := "Field must not be set"
}
```

### Pattern 4: Array Contains Check

```rego
import future.keywords.in

violation[{"msg": msg}] if {
  not "REQUIRED_VALUE" in input.review.object.spec.array
  msg := "Array must contain REQUIRED_VALUE"
}
```

### Pattern 5: String Prefix Check

```rego
violation[{"msg": msg}] if {
  value := input.review.object.spec.field
  not startswith(value, "expected-prefix")
  msg := "Value must start with expected-prefix"
}
```

### Pattern 6: Parameterized Constraints

```rego
# Use parameters from Constraint instance
violation[{"msg": msg}] if {
  param_value := input.parameters.my_param
  # use param_value in validation
}
```

---

## Output File Organization

### Directory Structure:
```
output/
├── README.md
├── constraint-templates/
│   ├── security/
│   ├── resource-management/
│   ├── networking/
│   ├── storage/
│   ├── container-images/
│   ├── pod-configuration/
│   └── workload-best-practices/
└── constraints/
    └── [same categories]
```

### File Naming:
- **ConstraintTemplates:** `{template-name}.yaml` (e.g., `k8srequirenonroot.yaml`)
- **Constraints:** `{descriptive-name}.yaml` (e.g., `require-nonroot-containers.yaml`)

### output/README.md Template:

```markdown
# Generated OPA Gatekeeper Policies

**Generated:** {timestamp}
**Cluster:** {context}
**Analyzed Namespaces:** {namespaces}

## Summary

- **ConstraintTemplates:** {count}
- **Constraints:** {count}
- **Categories:** {list}

```
---

## Error Handling

### No Namespaces to Analyze
If all namespaces are system namespaces, inform user and suggest specifying namespaces explicitly.

### MCP Connection Failure
If MCP tools fail, provide clear error message with troubleshooting:
- Check kubectl configuration
- Verify cluster is reachable
- Confirm MCP server is configured in `.claude/.mcp.json`

### Empty Namespaces
If namespace has no workloads, report 0 violations with success message.

### Output Directory Exists
If `output/` directory exists with files, ask user:
- Overwrite existing files?
- Create timestamped directory (e.g., `output-2026-01-11-143025/`)?

---

## Version History

- **v1.0.0** (2026-01-11)
  - 30+ Red Hat best practices
  - 7 categories (Security, Resources, Networking, Storage, Images, Pods, Workloads)
  - MCP integration for cluster analysis
  - Automated policy generation

---

**License:** Internal Use
**Documentation:** See README.md for team usage guide
