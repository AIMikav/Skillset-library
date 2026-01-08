# OpenShift MCP Server Setup Guide

This guide documents the setup process for integrating the OpenShift/Kubernetes MCP (Model Context Protocol) server with Claude Code to interact with a local minikube cluster.

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [What is the Kubernetes MCP Server?](#what-is-the-kubernetes-mcp-server)
- [Setup Steps](#setup-steps)
- [Verification](#verification)
- [Available Tools](#available-tools)
- [Configuration Options](#configuration-options)
- [Troubleshooting](#troubleshooting)
- [Resources](#resources)

## Overview

The Kubernetes MCP Server is a Model Context Protocol implementation that allows Claude Code to interact directly with Kubernetes and OpenShift clusters through the Kubernetes API. This integration enables AI-assisted cluster management, resource inspection, and operational tasks.

**Repository**: https://github.com/openshift/openshift-mcp-server

## Prerequisites

Before setting up the MCP server, ensure you have:

1. **Minikube Running**
   ```bash
   minikube status
   ```
   Expected output:
   ```
   minikube
   type: Control Plane
   host: Running
   kubelet: Running
   apiserver: Running
   kubeconfig: Configured
   ```

2. **kubectl Configured**
   ```bash
   kubectl cluster-info
   kubectl config current-context
   ```
   Verify that kubectl is pointing to your minikube cluster.

3. **Node.js and npx Installed**
   ```bash
   which npx
   ```
   The MCP server can be run via `npx` for easy installation and updates.

4. **Claude Code Installed**
   Ensure you have Claude Code CLI installed and configured.

## What is the Kubernetes MCP Server?

The Kubernetes MCP Server is a **native Go-based** implementation that provides:

- **Direct Kubernetes API Integration**: No external tool dependencies
- **Multi-cluster Support**: Can manage multiple clusters simultaneously
- **Comprehensive Toolsets**:
  - Configuration management
  - Pod operations (logs, exec, port-forward)
  - Namespace and event tracking
  - Helm chart management
  - Optional: Kiali, KubeVirt support
- **Cross-platform**: Works on Linux, macOS, and Windows
- **Flexible Deployment**: Run via npx, native binary, or compiled from source

## Setup Steps

### Step 1: Verify Minikube is Running

```bash
# Check minikube status
minikube status

# Verify kubectl connection
kubectl cluster-info

# Confirm current context
kubectl config current-context
```

Your current context should show `minikube`.

### Step 2: Create Claude Code MCP Configuration

Create the MCP configuration file at `.claude/.mcp.json`:

```bash
mkdir -p .claude
```

Add the following configuration to `.claude/.mcp.json`:

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

**Configuration Breakdown**:
- `"kubernetes"`: The name of the MCP server (can be customized)
- `"command": "npx"`: Uses npx to run the server
- `"args"`: Arguments passed to npx
  - `"-y"`: Auto-confirm installation
  - `"kubernetes-mcp-server@latest"`: Always use the latest version

### Step 3: Test the MCP Server

Verify the server can run:

```bash
npx -y kubernetes-mcp-server@latest --help
```

You should see the help output showing available commands and flags.

### Step 4: Restart Claude Code

For the MCP configuration to take effect, you must restart your Claude Code session:

```bash
# Exit current session
exit

# Start new session
claude
```

### Step 5: Verify MCP Integration

Once restarted, Claude Code will automatically load the Kubernetes MCP server. You can verify by:

1. Checking for MCP tools prefixed with `mcp__kubernetes__*`
2. Asking Claude to list pods, namespaces, or other cluster resources
3. Requesting cluster information

## Verification

After setup, verify the integration works:

```bash
# In a new Claude Code session, try commands like:
# "List all pods in the default namespace"
# "Show me all namespaces in the cluster"
# "Get cluster information"
```

Claude will use the MCP tools to interact with your minikube cluster.

## Available Tools

The Kubernetes MCP Server provides various toolsets:

### Core Tools (Default)
- **Resource Management**: Get, list, create, update, delete resources
- **Pod Operations**: View logs, execute commands, port-forward
- **Namespace Operations**: List, create, describe namespaces
- **Event Tracking**: Monitor cluster events
- **ConfigMaps & Secrets**: Manage configurations

### Configuration Tools (Default)
- **Context Management**: Switch between clusters
- **Kubeconfig Operations**: View and modify kubeconfig

### Helm Tools (Default)
- **Chart Management**: List, install, upgrade, uninstall charts
- **Release Operations**: Manage Helm releases

### Optional Toolsets
- **Kiali**: Service mesh visualization (requires Kiali installation)
- **KubeVirt**: Virtual machine management (requires KubeVirt)

## Configuration Options

### Basic Configuration

The default configuration uses `npx` to run the latest version. This is sufficient for most use cases.

### Advanced Configuration

You can customize the MCP server with additional flags:

```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "npx",
      "args": [
        "-y",
        "kubernetes-mcp-server@latest",
        "--read-only",
        "--toolsets", "core,helm",
        "--disable-destructive"
      ]
    }
  }
}
```

**Available Flags**:
- `--port <number>`: Run as SSE server on specified port (default: STDIO mode)
- `--read-only`: Enable read-only mode (no modifications allowed)
- `--toolsets <list>`: Comma-separated list of toolsets to enable (core, helm, kiali, kubevirt)
- `--disable-destructive`: Disable destructive operations (delete, etc.)
- `--disable-multi-cluster`: Disable multi-cluster tools (use only default context)
- `--kubeconfig <path>`: Specify custom kubeconfig file
- `--log-level <level>`: Set logging verbosity (debug, info, warn, error)
- `--config <path>`: Path to configuration file
- `--config-dir <path>`: Path to drop-in configuration directory

### Read-Only Mode

For safer operations, enable read-only mode:

```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "npx",
      "args": [
        "-y",
        "kubernetes-mcp-server@latest",
        "--read-only"
      ]
    }
  }
}
```

### Multi-Cluster Setup

If you have multiple clusters configured in your kubeconfig:

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

The server will automatically support all clusters in your kubeconfig. You can switch contexts through Claude.

## Troubleshooting

### MCP Server Not Loading

**Symptom**: No `mcp__kubernetes__*` tools available after restart.

**Solutions**:
1. Verify `.claude/.mcp.json` exists and is valid JSON
2. Completely exit and restart Claude Code
3. Check for errors in Claude Code startup logs
4. Test the server manually: `npx -y kubernetes-mcp-server@latest --help`

### Connection to Minikube Fails

**Symptom**: MCP tools fail with connection errors.

**Solutions**:
1. Verify minikube is running: `minikube status`
2. Check kubectl can connect: `kubectl get nodes`
3. Verify kubeconfig: `kubectl config view`
4. Ensure current context is minikube: `kubectl config current-context`

### NPX Installation Issues

**Symptom**: `npx` command not found or fails.

**Solutions**:
1. Install Node.js: `brew install node` (macOS) or download from nodejs.org
2. Update npm: `npm install -g npm@latest`
3. Clear npx cache: `npx clear-npx-cache`

### Permission Issues

**Symptom**: Permission denied errors when accessing cluster.

**Solutions**:
1. Check kubeconfig permissions: `ls -la ~/.kube/config`
2. Verify you have access to the cluster: `kubectl auth can-i get pods`
3. Ensure kubeconfig is readable: `chmod 600 ~/.kube/config`

### Tools Not Working as Expected

**Symptom**: MCP tools return errors or unexpected results.

**Solutions**:
1. Update to latest version: The `@latest` tag ensures you always get updates
2. Check cluster resources exist: `kubectl get all -A`
3. Enable debug logging by modifying `.mcp.json`:
   ```json
   {
     "mcpServers": {
       "kubernetes": {
         "command": "npx",
         "args": [
           "-y",
           "kubernetes-mcp-server@latest",
           "--log-level", "debug"
         ]
       }
     }
   }
   ```

### Minikube Not Accessible

**Symptom**: Cluster was accessible but now isn't.

**Solutions**:
1. Restart minikube: `minikube stop && minikube start`
2. Check VM status: `minikube status`
3. Verify network: `minikube ip`
4. Update kubeconfig: `minikube update-context`

## Resources

### Official Documentation
- **OpenShift MCP Server**: https://github.com/openshift/openshift-mcp-server
- **Model Context Protocol**: https://modelcontextprotocol.io/
- **Minikube Documentation**: https://minikube.sigs.k8s.io/docs/

### Related Tools
- **kubectl**: Kubernetes command-line tool
- **Helm**: Kubernetes package manager
- **Claude Code**: AI-powered development assistant

### Community
- Report issues: https://github.com/openshift/openshift-mcp-server/issues
- MCP Specification: https://spec.modelcontextprotocol.io/

## Example Use Cases

Once configured, you can use Claude Code to:

1. **Resource Inspection**
   - "Show me all pods in the kube-system namespace"
   - "List all services across all namespaces"
   - "Get details about the coredns deployment"

2. **Troubleshooting**
   - "Show logs for the failing pod"
   - "What events happened in the default namespace recently?"
   - "Check the status of all nodes"

3. **Resource Management**
   - "Create a new namespace called 'dev'"
   - "Apply this deployment manifest"
   - "Scale the nginx deployment to 3 replicas"

4. **Helm Operations**
   - "List all installed Helm charts"
   - "Install the nginx-ingress chart"
   - "Upgrade the prometheus release"

## Security Considerations

1. **Read-Only Mode**: Consider using `--read-only` flag for safer operations
2. **Destructive Operations**: Use `--disable-destructive` to prevent accidental deletions
3. **Cluster Access**: The MCP server uses your kubeconfig credentials
4. **Network Security**: In STDIO mode (default), the server only communicates via stdin/stdout
5. **SSE Mode**: If using `--port`, be aware the server listens on that port

## Maintenance

### Updating the MCP Server

The configuration uses `@latest`, which automatically pulls the newest version on each run. No manual updates needed.

To pin a specific version:
```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "npx",
      "args": [
        "-y",
        "kubernetes-mcp-server@1.0.0"
      ]
    }
  }
}
```

### Checking Version

```bash
npx -y kubernetes-mcp-server@latest --version
```

## Summary

You've successfully integrated the Kubernetes MCP Server with Claude Code to manage your minikube cluster. This setup enables:

- AI-assisted Kubernetes operations
- Natural language cluster management
- Seamless resource inspection and manipulation
- Helm chart management
- Multi-cluster support (if needed)

The server uses your existing kubeconfig and requires no additional authentication. Simply restart Claude Code after configuration changes, and you're ready to go!

---

**Setup Date**: January 8, 2026
**Minikube Version**: Check with `minikube version`
**Kubectl Version**: Check with `kubectl version`
**MCP Server**: Latest (via npx)
