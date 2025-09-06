# Treemap: Design Document

## Overview
This document describes the design and architecture of a Treemap, a SwiftUI-based application for visualizing hierarchical data using a space-filling treemap layout.

## Data Model
- **Node**: Represents a node in the hierarchy. Each node has an `id`, a `size`, and an array of child nodes. Nodes can be nested to represent hierarchical structures. The `prepare()` method ensures that the node's size is at least the sum of its children and sorts children by size.

## Treemap Layout
- **Treemap**: The main struct responsible for computing the layout of nodes. It uses a squarified treemap algorithm to partition the available space among child nodes, alternating between horizontal and vertical splits.
- **TreemapConfig**: Configuration for the treemap, including padding, minimum size thresholds, and label/caption logic.
- **ChildLayout**: Associates a node with its computed frame (CGRect) for rendering.

## Rendering
- **TreemapView**: A SwiftUI view that recursively renders the treemap. It draws rectangles for each node, overlays labels, and handles user interactions such as hover and tap. It uses `GeometryReader` to adapt to available space and recursively renders child nodes using the computed layout.
- **ContentView**: The app's main view. It loads hierarchical data from a JSON file, prepares the root node, and displays the treemap. It manages state for the currently hovered node and zoom interactions.

## User Interaction
- **ZoomController**: An observable object that tracks the currently selected (zoomed) node path. The treemap layout adapts to focus on the selected node if set.
- **Hovering**: The view highlights nodes on hover and updates the hovered path state.
- **Tapping**: Tapping a node can trigger zoom or selection logic (implementation can be extended).

## Algorithm Details
- **Squarified Treemap**: The layout algorithm divides the available rectangle among children to minimize aspect ratio variance, improving readability. It uses a recursive approach, alternating orientation at each level.
- **Node Preparation**: Before layout, the tree is prepared so that each node's size is at least the sum of its children, and children are sorted by size for better visual grouping.

## Extensibility
- The design allows for custom captions, node visibility logic, and child display logic via `TreemapConfig`.
- The data model is Codable, supporting loading from JSON or other sources.

## File Structure
- `Treemap.swift`: Data model, layout algorithm, configuration, and helpers.
- `TreemapView.swift`: SwiftUI view for rendering the treemap.
- `ContentView.swift`: Loads data and hosts the main view.
- `TreemapApp.swift`: App entry point.

## Example Data
- The app expects a `data.json` file containing the root node and its hierarchy.

---

This design enables interactive, scalable, and customizable treemap visualizations in SwiftUI, suitable for a variety of hierarchical data sets.
