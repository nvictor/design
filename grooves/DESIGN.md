# Grooves: 1FA Design Document

## 1. Introduction

This document outlines the design for **Grooves**, a single-file SwiftUI application that functions as a musical step-sequencer. Users can create rhythmic patterns (grooves), play them back, and manage a library of presets.

## 2. File Structure

The application is contained entirely within `grooves.swift`. The code is organized by grouping related components, following this order:

1.  **Models**: `Beat`, `Preset`
2.  **View Models / Managers**: `PlaybackManager`, `PresetManager`, `SoundService`
3.  **Component Views**: `BeatBlock`
4.  **Main Views**: `ContentView`, `Editor`, `Sidebar`, `Inspector`
5.  **App Entry Point**: `GroovesApp`
6.  **Utility Structs**: `ImageExporter`

## 3. Application Architecture

### 3.1. Entry Point

The application's entry point is the `GroovesApp` struct, marked with `@main`.

### 3.2. Root View

`ContentView` is the root view, setting up the main layout with a `NavigationSplitView`. It initializes the `PlaybackManager` and handles the primary state for the selected preset and inspector visibility.

### 3.3. View Composition

-   **`NavigationSplitView`**: The main interface is a master-detail layout.
-   **`Sidebar`**: Displays a list of available presets, grouped by category, allowing the user to select one to load into the editor.
-   **`Editor`**: The main content view where the user can see and modify the groove pattern. It displays a sequence of `BeatBlock`s.
-   **`BeatBlock`**: A small, tappable `Rectangle` representing a single beat in the sequence. Its appearance changes based on the `Beat` type (`empty`, `low`, `high`).
-   **`Inspector`**: A view for editing properties like BPM (Beats Per Minute) and the total number of beats in the groove.

## 4. State Management

-   **`@StateObject`**: `ContentView` owns a `PlaybackManager` as a `@StateObject` to manage the application's core logic and playback state.
-   **`PlaybackManager`**: An `ObservableObject` that encapsulates the playback logic, including the timer, current beat, BPM, and the `groove` data. It uses a `SoundService` to play audio.
-   **`@State`**: Used for simple view state, such as the currently selected `preset` and the `showInspector` boolean.
-   **`@Binding`**: Used to pass state down to child views like `Editor` and `Inspector`, allowing them to modify the state owned by `ContentView` and `PlaybackManager`.

## 5. Data Modeling

-   **`Beat`**: An `enum` representing the state of a single beat (`empty`, `low`, `high`). It includes helper methods to cycle through states and determine visual style.
-   **`Preset`**: An `Identifiable` struct that stores a named groove pattern, containing a `name` and an array of `Beat`s.

## 6. Data Persistence

-   **`UserDefaults`**: User data is persisted to `UserDefaults`.
-   **`PresetManager`**: A singleton class responsible for loading and saving presets. It loads default presets from a built-in JSON string and handles the encoding/decoding of `Preset` objects to and from `UserDefaults`.
-   **Debug Reset**: The `Inspector` includes a "Reset Presets" button to clear saved data from `UserDefaults`.

## 7. Common UI Patterns & Features

### 7.1. Playback Control

Playback is controlled via toolbar buttons (Play/Stop) and the spacebar key. The `PlaybackManager` uses a `Timer` to advance the sequence and a `SoundService` to play a system "tink" sound for each active beat.

### 7.2. Image Exporting

An `ImageExporter` utility allows the user to export the current `Editor` view as a PNG image using SwiftUI's `ImageRenderer` and `NSSavePanel`.
