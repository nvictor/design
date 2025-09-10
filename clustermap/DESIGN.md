# Clustermap: Design Document

This document outlines the design for a SwiftUI-based Kubernetes treemap visualizer.

## 1. Connecting to a Kubernetes Cluster

The application connects to the Kubernetes API server to fetch resource data and the Kubernetes Metrics Server for real-time usage data.

### Authentication

Authentication is supported via a local `kubeconfig` file. The primary method for local development, the app parses `~/.kube/config` to get the cluster URL, certificate, token, and other settings.

-   **Client Certificates:** Authenticates using the client certificate and key data specified in the `kubeconfig`. The implementation uses a simplified keychain-based approach:
    -   Uses `SecIdentityCreateWithCertificate` to create a `SecIdentity` from certificate data
    -   Handles keychain access gracefully with error logging
    -   Supports certificate-based authentication for TLS client authentication
-   **Bearer Tokens:** Supports bearer tokens, including those obtained from `exec` plugins with full environment variable support.
-   **TLS Verification:** Respects the `insecure-skip-tls-verify` flag for clusters with self-signed certificates. Includes smart defaults for common development patterns (minikube, localhost, private IPs).

### Networking

-   **API Client:** `URLSession` is used for making REST API calls to the Kubernetes server and the Metrics Server. A custom `URLSessionDelegate` handles TLS server trust validation and client certificate challenges.
-   **Data Parsing:** Swift's `Codable` protocol is used to decode the JSON responses into native Swift objects. A YAML parser (`Yams`) is used as a fallback for parsing `kubeconfig` files.

## 2. Treemap Data Structure

Kubernetes objects are transformed into a hierarchical structure suitable for a treemap.

### Hierarchy View

The application uses a single, structural hierarchy to display resources: `Namespace` → `Deployment` → `Pod`. This view is useful for understanding the parent-child and ownership relationships between the core workload resources in the cluster.

### Sizing Metrics

The size of each rectangle in the treemap represents a specific real-time usage metric, fetched from the Kubernetes Metrics Server.

-   **Resource Count:** Size based on the number of child resources.
-   **CPU:** Size based on real-time CPU usage (in cores).
-   **Memory:** Size based on real-time memory usage (in bytes).

### Core Data Model

A recursive `TreeNode` struct serves as the core model for the treemap visualization.

```swift
struct TreeNode: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let value: Double // Determines the size of the node's rectangle
    let children: [TreeNode]
    var isLeaf: Bool { children.isEmpty }
}
```

## 3. Rendering the Treemap

A custom SwiftUI view recursively renders the treemap.

### Implementation

The view uses `GeometryReader` to read the available space and proportionally divide it among its children based on their `value`. The implementation includes sophisticated zoom functionality and treemap layout calculations.

```swift
struct TreemapView: View {
    @EnvironmentObject private var viewModel: ClusterViewModel
    let node: TreeNode
    let path: [UUID]
    @State private var hoveredPath: [UUID]?

    var body: some View {
        GeometryReader { geometry in
            ZStack(alignment: .topLeading) {
                backgroundView
                labelView(geometry: geometry)
                childrenView(geometry: geometry)
            }
        }
        .background(Color(.windowBackgroundColor))
        .clipped()
        .padding(LayoutConstants.mainPadding)
    }
}
```

The implementation includes:
- **ZoomController**: Handles zoom state and determines which children to show based on the selected path
- **TreemapLayoutCalculator**: Computes optimal treemap layouts using a squarified algorithm
- **Interactive Features**: Hover effects, click-to-zoom functionality, and breadcrumb navigation

The application uses a dual coloring strategy to enhance clarity:
- **Parent Nodes (Namespaces, Deployments):** Are assigned a stable, distinct color based on their name. This provides a clear and consistent visual structure for the hierarchy.
- **Leaf Nodes (Pods):** Are colored using a heatmap scale from green (low usage) to red (high usage). The color is determined by the pod's resource consumption relative to the most resource-intensive pod in the cluster, making it easy to spot performance hotspots at a glance.

Text color automatically adjusts for readability against all backgrounds.

## 4. User Interface

The main interface features:

-   A central `TreemapView` displaying the cluster resources with zoom and hover interactions.
-   An Inspector sidebar containing:
    - Connection settings with kubeconfig path configuration
    - Display controls with a segmented picker for sizing metrics (Count, CPU, Memory)
    - Console view showing logs and status messages
-   Toolbar with reload and inspector toggle buttons.

### Hierarchy and Metric Options

The application supports a single hierarchy view:
- **Resource Hierarchy**: Namespace → Deployment → Pod

Three sizing metrics are available, based on real-time usage:
- **Count**: Size based on the number of resources
- **CPU**: Size based on CPU usage (in cores). Values less than 1 core are displayed in millicores (e.g., "50m").
- **Memory**: Size based on memory usage (in bytes)

## 5. Security & Deployment

-   **Keychain Access:** The application requires the `keychain-access-groups` entitlement to access certificates in the macOS Keychain. The client certificate authentication implementation:
    -   Uses `SecIdentityCreateWithCertificate` for identity creation from certificate data
    -   Handles keychain errors gracefully (e.g., missing certificates, access issues)
    -   Supports certificate-based authentication for secure cluster connections
    -   Logs authentication issues to help with debugging connection problems
-   **Local Use:** The app is designed to run on macOS and connect directly to the cluster via the user's `kubeconfig` or a service account token. It requires that the **Kubernetes Metrics Server** is installed and running in the cluster.
-   **Team/Enterprise Use:** For wider distribution, a backend-for-frontend (BFF) approach would be recommended to avoid distributing cluster credentials to client applications.
