# Clustermap: Design Document

This document outlines the design for a SwiftUI-based Kubernetes treemap visualizer.

## 1. Connecting to a Kubernetes Cluster

The application connects to the Kubernetes API server to fetch resource data.

### Authentication

Authentication is supported via a local `kubeconfig` file. The primary method for local development, the app parses `~/.kube/config` to get the cluster URL, certificate, token, and other settings.

-   **Client Certificates:** Authenticates using the client certificate and key data specified in the `kubeconfig`. The implementation uses a sophisticated keychain-based approach:
    -   First checks if a matching certificate already exists in the macOS Keychain (common with minikube/kubectl usage)
    -   If found, creates a `SecIdentity` using the existing certificate paired with the private key from kubeconfig
    -   If not found, temporarily imports both certificate and private key into the Keychain to create a `SecIdentity`
    -   Uses `SecIdentityCreateWithCertificate` for reliable identity creation with proper certificate-key pairing
    -   Handles keychain access gracefully with automatic cleanup of temporary items
-   **Bearer Tokens:** Supports bearer tokens, including those obtained from `exec` plugins.
-   **TLS Verification:** Respects the `insecure-skip-tls-verify` flag for clusters with self-signed certificates.

### Networking

-   **API Client:** `URLSession` is used for making REST API calls to the Kubernetes server. A custom `URLSessionDelegate` handles TLS server trust validation and client certificate challenges.
-   **Data Parsing:** Swift's `Codable` protocol is used to decode the JSON responses into native Swift objects. A YAML parser (`Yams`) is used as a fallback for parsing `kubeconfig` files.

## 2. Treemap Data Structure

Kubernetes objects are transformed into a hierarchical structure suitable for a treemap.

### Hierarchy Views

The user can switch between two distinct hierarchical views:

-   **By Resource Type:** A structural view showing `Namespace` → `Deployment` → `Pod`. This is useful for understanding the parent-child relationships between resources.
-   **By Namespace:** A categorical view showing `Namespace` → `Kind` (e.g., "Deployments", "CronJobs") → `Resource Name`. This is useful for getting an inventory of all resources running in a namespace.

### Sizing Metrics

The size of each rectangle in the treemap represents a specific metric:

-   **Resource Count:** Size based on the number of child resources.
-   **CPU:** Size based on CPU requests.
-   **Memory:** Size based on memory requests.

### Core Data Model

A recursive `TreeNode` struct serves as the core model for the treemap visualization.

```swift
struct TreeNode: Identifiable {
    let id = UUID()
    let name: String
    let value: Double // Determines the size of the node's rectangle
    let children: [TreeNode]
}
```

## 3. Rendering the Treemap

A custom SwiftUI view recursively renders the treemap.

### Implementation

The view uses `GeometryReader` to read the available space and proportionally divide it among its children based on their `value`.

```swift
struct TreemapView: View {
    let node: TreeNode
    var depth: Int = 0

    var body: some View {
        GeometryReader { geo in
            if node.children.isEmpty {
                // Render the leaf node with a color based on its value
            } else {
                // Recursively split the geometry for child nodes
                TreemapSplitView(children: node.children, ...)
            }
        }
    }
}
```

Cell colors are generated on a logarithmic scale based on the node's value, providing a visual heatmap of resource consumption. Text color automatically adjusts for readability against the background.

## 4. User Interface

The main interface features:

-   A central `TreemapView` displaying the cluster resources.
-   Segmented pickers to switch between hierarchy views and sizing metrics.
-   A settings sidebar for managing cluster connections.

## 5. Security & Deployment

-   **Keychain Access:** The application requires the `keychain-access-groups` entitlement to access certificates in the macOS Keychain. The client certificate authentication implementation:
    -   Searches for existing certificates in Keychain by comparing certificate data to avoid duplicates
    -   Handles keychain errors gracefully (e.g., duplicate items, missing entitlements)
    -   Uses temporary keychain storage with automatic cleanup for imported private keys
    -   Validates certificate-identity pairing to ensure correct authentication credentials
    -   Supports both RSA and EC private keys with automatic key type detection
-   **Local Use:** The app is designed to run on macOS and connect directly to the cluster via the user's `kubeconfig` or a service account token.
-   **Team/Enterprise Use:** For wider distribution, a backend-for-frontend (BFF) approach would be recommended to avoid distributing cluster credentials to client applications.
