# Parts: Design Document

## 1. Introduction

This document outlines the design for **Parts**, a single-file SwiftUI application designed for arranging song structures. Users can compose a song by sequencing different parts (e.g., intro, verse, chorus), each with a specified length in bars.

## 2. File Structure

The application is contained entirely within `parts.swift`. The code is organized by grouping related components, following this order:

1.  **Models**: `Segment`, `Part`, `Preset`
2.  **Managers**: `PresetManager`
3.  **Component Views**: `SegmentBlock`
4.  **Main Views**: `ContentView`, `Editor`, `Sidebar`, `Inspector`
5.  **App Entry Point**: `PartsApp`
6.  **Utility Structs**: `ImageExporter`

## 3. Application Architecture

### 3.1. Entry Point

The application's entry point is the `PartsApp` struct, marked with `@main`.

### 3.2. Root View

`ContentView` is the root view, which sets up a `NavigationSplitView`. It manages the application's state, including the current arrangement of `segments`, the selected preset, and the visibility of the inspector.

### 3.3. View Composition

-   **`NavigationSplitView`**: The main UI is a master-detail layout.
-   **`Sidebar`**: Displays a list of presets (e.g., "AABA", "12-bar blues") that the user can select to load a song structure into the editor.
-   **`Editor`**: The main content area where the song structure is visualized. It renders the sequence of `SegmentBlock`s, automatically wrapping them into rows of 16 bars.
-   **`SegmentBlock`**: A visual component representing a single part of the song. Its color and label are determined by the `Part` enum, and its width is proportional to its `length`.
-   **`Inspector`**: A side panel that allows the user to edit the properties of the currently selected segment, such as its `part` type and `length`.

## 4. State Management

-   **`@State`**: `ContentView` uses `@State` to manage the `segments` array, the selected `preset`, the `selectedSegmentID`, and the `showInspector` boolean.
-   **`@Binding`**: Bindings are used to pass state to the `Editor` and `Inspector`. A custom `Binding` (`effectiveSegments`) is used to seamlessly switch between editing the local state and the selected preset's state.

## 5. Data Modeling

-   **`Segment`**: An `Identifiable`, `Codable` struct representing a section of a song. It contains a `part` and a `length` (in bars).
-   **`Part`**: A `String`-backed `enum` that defines the different types of song parts (e.g., `intro`, `verse`, `chorus`, `bridge`). Each part has an associated color for visualization.
-   **`Preset`**: An `Identifiable`, `Codable` struct that stores a named song structure as a sequence of `Segment`s.

## 6. Data Persistence

-   **`UserDefaults`**: The application persists the user's preset library using `UserDefaults`.
-   **`PresetManager`**: A singleton class responsible for loading and saving presets. It loads default song structures from a hardcoded JSON string and handles the serialization of `Preset` objects for storage.
-   **Debug Reset**: The `Inspector` includes a "Reset Presets" button to clear saved data from `UserDefaults`.

## 7. Common UI Patterns & Features

### 7.1. Image Exporting

An `ImageExporter` utility allows the user to export the current song arrangement in the `Editor` view as a PNG file. It uses `ImageRenderer` and `NSSavePanel`.

### 7.2. Automatic Layout

The `Editor` view automatically groups segments into rows, ensuring that each row does not exceed a total length of 16 bars. This provides a clean and readable layout for the song structure.
