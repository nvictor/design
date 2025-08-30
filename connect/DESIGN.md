# Connect: 1FA Design Document

## 1. Introduction

This document outlines the design for **Connect**, a single-file SwiftUI application for creating simple node-and-connector diagrams. The user can select a node and use the arrow keys to create new nodes connected in the chosen direction.

## 2. File Structure

The application is contained entirely within `connect.swift`. The code is organized by grouping related components, following this order:

1.  **Models**: `Node`, `Connector`, `NodeSide`
2.  **Component Views**: `NodeView`, `ConnectorView`
3.  **Custom Shapes**: `ArrowHead`
4.  **Main View**: `ContentView`
5.  **App Entry Point**: `ConnectApp`
6.  **Utility Views**: `KeyCatcherView`

## 3. Application Architecture

### 3.1. Entry Point

The application's entry point is the `ConnectApp` struct, which conforms to the `App` protocol and is marked with `@main`.

### 3.2. Root View

The main view is `ContentView`, which manages the layout of nodes and connectors. It uses a `ZStack` to overlay the connectors and nodes.

### 3.3. View Composition

-   **`NodeView`**: A `RoundedRectangle` that visually represents a single node. Its color changes when it is selected.
-   **`ConnectorView`**: A `Path` that draws a line between two points, with an `ArrowHead` at the destination.
-   **`ArrowHead`**: A custom `Shape` that draws a triangular arrowhead pointing in the direction of the connection.
-   **`KeyCatcherView`**: An `NSViewRepresentable` that captures keyboard events (`NSEvent`) to handle user input for creating new nodes.

## 4. State Management

State is managed within `ContentView` using the `@State` property wrapper for view-specific data that is not shared.

-   **`@State private var nodes: [Node]`**: Stores the array of all nodes in the diagram.
-   **`@State private var connectors: [Connector]`**: Stores the array of all connectors.
-   **`@State private var selectedNodeID: UUID?`**: Keeps track of the currently selected node's ID.

## 5. Data Modeling

Data models are defined as simple `struct`s.

-   **`Node`**: Conforms to `Identifiable`. Represents a single node with a `UUID`, a `position` (`CGPoint`), and a `size` (`CGSize`).
-   **`Connector`**: Conforms to `Identifiable`. Represents a connection between two nodes, defined by a `from` and `to` tuple, each containing a `nodeID` and a `NodeSide`.
-   **`NodeSide`**: An `enum` that specifies which side of a node a connector attaches to (`top`, `bottom`, `left`, `right`).

## 6. Data Persistence

This application does not support data persistence. The diagram state is transient and resets when the application is closed.

## 7. User Interaction

-   **Node Selection**: Tapping a `NodeView` sets it as the `selectedNodeID`.
-   **Node Creation**: When a node is selected, pressing an arrow key creates a new node in that direction, connected to the currently selected node. This logic is handled in the `handleKey` function, which is triggered by the `KeyCatcherView`.
