# Vignette: Design Document

## 1. Introduction

This document outlines the design for **Vignette**, a single-file SwiftUI application for creating stylized images from text. It is designed to format short jokes (a setup and a punchline) into a visually appealing, shareable PNG image with a custom font and a procedurally generated background.

## 2. File Structure

The application is contained entirely within `vignette.swift`. The code is organized by grouping related components, following this order:

1.  **Background Views**: `BGLayeredWaves`, `BGLowPoly`, `BGVanishingRays`, `BGRandom`
2.  **Main Views**: `ContentView`, `JokeView`
3.  **App Entry Point**: `VignetteApp`
4.  **Utility Structs**: `ImageExporter`
5.  **ViewModifiers**: `StrokeModifier`

## 3. Application Architecture

### 3.1. Entry Point

The application's entry point is the `VignetteApp` struct, marked with `@main`.

### 3.2. Root View

`ContentView` is the root view, which manages the application's two-step workflow. It contains the state for the joke's text, the selected background style, and the current step in the UI flow.

### 3.3. View Composition

-   **`JokeView`**: The core visual component that lays out the setup and punchline text over a generated background. This view is what gets exported as an image.
-   **`BGRandom`**: A view that randomly selects and displays one of the three procedural background styles.
    -   **`BGLayeredWaves`**: Creates a background with layered, colored wave shapes.
    -   **`BGLowPoly`**: Generates a low-polygon triangle pattern with random colors.
    -   **`BGVanishingRays`**: Draws a series of rays emanating from a central point.
-   **`StrokeModifier`**: A custom `ViewModifier` that adds a solid stroke around text, making it more readable against complex backgrounds.

## 4. State Management

State management is simple and self-contained within `ContentView`, using the `@State` property wrapper for all mutable properties.

-   **`@State private var currentStep`**: An integer that controls the application's wizard-like interface (step 0 for text entry, step 1 for preview and export).
-   **`@State private var style`**: Stores the currently selected `BGRandom.BGStyle`.
-   **`@State private var setup`**: Stores the setup text for the joke.
-   **`@State private var punchline`**: Stores the punchline text.

## 5. Data Modeling

The application does not use any custom `struct`s or `class`es for data modeling. The state is managed entirely with simple types like `String`, `Int`, and the `BGRandom.BGStyle` enum.

## 6. Data Persistence

**Vignette** does not support data persistence. All text and style selections are transient and reset when the application is closed.

## 7. Common UI Patterns & Features

### 7.1. Image Exporting

An `ImageExporter` utility struct provides a static `export` function that takes a SwiftUI `View` and a `CGSize` as input. It uses `ImageRenderer` to render the view to an `NSImage` and then converts it to PNG data. An `NSSavePanel` is presented to the user to choose a save location.

### 7.2. Custom Styling

-   **Custom Font**: The application uses the "Cherry Bomb One" custom font to give the text a distinct, playful look.
-   **Text Stroke**: The `withStroke()` view extension, powered by `StrokeModifier`, is used to improve text legibility by drawing a black outline around the characters.

### 7.3. Procedural Backgrounds

The application features multiple procedurally generated backgrounds, providing visual variety for the exported images. The user can cycle through random backgrounds in the preview step.
