# Clustermap: Design Document

This document outlines the design for a SwiftUI-based Kubernetes treemap visualizer.

## 1. Connecting to a Kubernetes Cluster

The application connects to the Kubernetes API server to fetch resource data and the Kubernetes Metrics Server for real-time usage data.

### Authentication

Authentication is handled via a local `kubeconfig` file. The application parses `~/.kube/config` to get the cluster URL, certificate, token, and other settings.

-   **Client Certificates:** Authenticates using client certificate and key data from the `kubeconfig`. The implementation uses `SecIdentityCreateWithCertificate` to create a `SecIdentity` on-the-fly from certificate data for TLS client authentication.
-   **Bearer Tokens:** Supports static bearer tokens from the `kubeconfig`.
-   **Exec Plugins:** Provides robust support for `exec` plugins to obtain dynamic credentials.
    -   **GKE Support:** Includes specific logic to locate and execute the `gke-gcloud-auth-plugin`, searching common installation paths (e.g., Homebrew, Google Cloud SDK) to ensure it works out-of-the-box for most users. Provides detailed, user-friendly error messages if the plugin is not found.
    -   **Environment:** Correctly resolves environment variables specified in the `kubeconfig` for the `exec` command.
-   **TLS Verification:** A custom `URLSessionDelegate` (`TLSDelegate`) handles TLS validation.
    -   Respects the `insecure-skip-tls-verify` flag.
    -   Includes smart defaults to automatically skip verification for common development patterns (minikube, localhost, private IPs).
    -   Supports custom Certificate Authorities (CAs) for clusters like GKE, using a `SecPolicyCreateBasicX509` policy for reliable validation.

### Networking

-   **API Client:** `URLSession` makes REST API calls to the Kubernetes server and the Metrics Server. A custom `User-Agent` (`Clustermap/1.0`) is sent with all requests.
-   **Performance:** Resource fetching is highly concurrent. The app uses Swift's modern concurrency features (`async/await` and `withThrowingTaskGroup`) to fetch data for all namespaces in parallel, significantly speeding up load times for large clusters.
-   **Data Parsing:** Swift's `Codable` protocol decodes JSON responses into native Swift objects. The `Yams` library is used to parse the `kubeconfig` file.

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

The view uses `GeometryReader` to read the available space and proportionally divide it among its children based on their `value`.

```swift
struct TreemapView: View {
    @EnvironmentObject private var viewModel: ClusterViewModel
    let node: TreeNode
    let maxLeafValue: Double
    let path: [UUID]
    @State private var hoveredPath: [UUID]?
    // ...
}
```

The implementation includes:
- **ZoomController**: A dedicated controller that manages the zoom state, determining which children to render based on the user's selection. This allows for drilling down into nested layers of the hierarchy.
- **TreemapLayoutCalculator**: Computes optimal treemap layouts using a squarified algorithm, ensuring that rectangles are as close to square as possible for better readability and comparison.
- **Interactive Features**: Smooth hover effects, click-to-zoom functionality, and breadcrumb-style navigation managed via the `selectedPath`.
- **Value Formatting:** Resource values are formatted for human readability. For example, CPU values less than 1 core are displayed in millicores (e.g., "500m"), and memory is shown in Gi, Mi, or Ki.

The application uses a dual coloring strategy to enhance clarity:
- **Parent Nodes (Namespaces, Deployments):** Are assigned a stable, distinct color based on a hash of their name. This provides a clear and consistent visual structure for the hierarchy.
- **Leaf Nodes (Pods):** Are colored using a heatmap scale from green (low usage) to orange/red (high usage). The color is determined by the pod's resource consumption relative to the most resource-intensive pod in the cluster, making it easy to spot performance hotspots at a glance.

Text color automatically adjusts for readability against all backgrounds.

## 4. User Interface

The main interface features:

-   A central `TreemapView` displaying the cluster resources with zoom and hover interactions.
-   An Inspector sidebar containing:
    - **Connection:** A text field to specify the `kubeconfig` path.
    - **Display:** A segmented picker to switch between sizing metrics (Count, CPU, Memory).
    - **Console:** An interactive console view that displays timestamped logs with different severity levels (Info, Success, Error), each with a distinct color and icon. Users can select, copy, and clear logs.
-   A Toolbar with buttons to reload data and toggle the inspector sidebar.

## 5. Security & Deployment

-   **App Sandbox:** The application is **not sandboxed** (`com.apple.security.app-sandbox` is `false`). This is a deliberate choice to allow the app to execute `exec` helper tools like `gke-gcloud-auth-plugin`, which is essential for authenticating with major cloud providers.
-   **Entitlements:** The app requests the following entitlements:
    -   `com.apple.security.files.user-selected.read-only`: To read the user-specified `kubeconfig` file.
    -   `com.apple.security.network.client`: To connect to Kubernetes API servers on the network.
-   **Network Security:** The `Info.plist` is configured with `NSAllowsArbitraryLoads` set to `true`. This is necessary to connect to local clusters (e.g., minikube) and cloud provider clusters (like GKE) that use IP addresses in their server certificates, which would otherwise be blocked by App Transport Security (ATS).
-   **Local Use:** The app is designed to run on macOS and connect directly to the cluster via the user's `kubeconfig`. It requires that the **Kubernetes Metrics Server** is installed and running in the cluster.
-   **Team/Enterprise Use:** For wider distribution, a backend-for-frontend (BFF) approach would be recommended to avoid distributing cluster credentials to client applications.
