# Moves: Design Document

## 1. Introduction

This document outlines the design for **Moves**, a single-file SwiftUI application for composing musical phrases and sequences. The application allows users to create musical lines composed of segments, each with a specific movement type, length, and associated note.

## 2. File Structure

The application is contained entirely within `moves.swift`. The code is organized by grouping related components, following this order:

1.  **Models**: `Line`, `Segment`, `Movement`, `Preset`
2.  **Managers**: `PresetManager`
3.  **Component Views**: `SegmentBlock`
4.  **Main Views**: `ContentView`, `Editor`, `Sidebar`, `Inspector`
5.  **App Entry Point**: `MovesApp`
6.  **Utility Structs**: `ImageExporter`

## 3. Application Architecture

### 3.1. Entry Point

The application's entry point is the `MovesApp` struct, marked with `@main`.

### 3.2. Root View

`ContentView` serves as the root view, establishing the main `NavigationSplitView` layout. It manages the application's primary state, including the array of `lines`, the `currentLine`, the `selectedSegmentID`, and the visibility of the inspector.

### 3.3. View Composition

-   **`NavigationSplitView`**: The UI is structured as a master-detail interface.
-   **`Sidebar`**: Displays a list of presets, grouped by category, which can be loaded into the currently selected line in the editor.
-   **`Editor`**: The main view for displaying and interacting with the musical lines. It renders each `Line` as a horizontal sequence of `SegmentBlock`s.
-   **`SegmentBlock`**: A visual component representing a single segment. Its appearance is determined by the segment's `movement` type and `length`.
-   **`Inspector`**: A side panel for editing the properties of the selected line (e.g., name, repeat count) and the selected segment (e.g., length).

## 4. State Management

-   **`@State`**: `ContentView` uses `@State` to manage the collection of `lines`, the `currentLine` index, the `selectedSegmentID`, the active `preset`, and the `showInspector` boolean.
-   **`@Binding`**: Bindings are used extensively to pass state down to the `Editor` and `Inspector`, allowing them to read and modify the state owned by `ContentView`.

## 5. Data Modeling

-   **`Line`**: An `Identifiable`, `Codable` struct representing a single musical line with a `name`, an array of `Segment`s, and a `repeatCount`.
-   **`Segment`**: An `Identifiable`, `Codable` struct that is the basic building block of a `Line`. It contains a `movement` type, a `length`, and a `note`.
-   **`Movement`**: A `String`-backed `enum` defining the type of musical movement (`repeat`, `progress`, `empty`), with an associated color for visualization.
-   **`Preset`**: An `Identifiable`, `Codable` struct for storing predefined sequences of `Segment`s.

## 6. Data Persistence

-   **`UserDefaults`**: The application uses `UserDefaults` to persist the user's library of presets.
-   **`PresetManager`**: A singleton class that handles loading and saving presets. It can load default presets from a hardcoded JSON string and manages the serialization of `Preset` objects for storage in `UserDefaults`.
-   **Debug Reset**: The `Inspector` provides a "Reset Presets" button to clear the saved presets from `UserDefaults`.

## 7. Common UI Patterns & Features

### 7.1. Image Exporting

An `ImageExporter` utility provides functionality to export the `Editor` view, containing all the musical lines, as a PNG file. It uses `ImageRenderer` to capture the view and `NSSavePanel` to prompt the user for a save location.
