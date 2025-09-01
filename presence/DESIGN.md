# Presence: Design Document

## 1. Introduction

This document outlines the design for **Presence**, a single-file SwiftUI application used to track the status of a group of attendees. Users can see a grid of avatars, change their status with a tap, and manage the attendee list.

## 2. File Structure

The application is contained entirely within `presence.swift`. The code is organized by grouping related components, following this order:

1.  **Models**: `Attendee`
2.  **View Models**: `AttendeeViewModel`
3.  **Component Views**: `AvatarCard`, `IconPicker`, `RenameSheet`
4.  **Main View**: `ContentView`
5.  **App Entry Point**: `PresenceApp`
6.  **ViewModifiers**: `ShakeEffect`

## 3. Application Architecture

### 3.1. Entry Point

The application's entry point is the `PresenceApp` struct, marked with `@main`.

### 3.2. Root View

`ContentView` is the root view. It displays a `LazyVGrid` of attendees and manages the application's "edit mode" state. It initializes the `AttendeeViewModel` to handle all data logic.

### 3.3. View Composition

-   **`AvatarCard`**: The primary UI component, displaying an attendee's avatar and name. The card's background color reflects the attendee's current state. It also handles tap gestures to change state and context menus for editing.
-   **`RenameSheet`**: A sheet view presented for renaming an attendee.
-   **`IconPicker`**: A popover view that displays a grid of emoji icons for the user to choose a new avatar.
-   **`ShakeEffect`**: A `ViewModifier` that applies a subtle, random offset to a view, used to indicate that the application is in "edit mode".

## 4. State Management

-   **`@StateObject`**: `ContentView` owns an `AttendeeViewModel` as a `@StateObject`, which serves as the single source of truth for the attendee data.
-   **`AttendeeViewModel`**: An `ObservableObject` that encapsulates all business logic. It manages the `@Published` array of `attendees` and is responsible for loading from and saving to `UserDefaults`.
-   **`@State`**: `ContentView` uses a simple `@State` boolean, `isEditing`, to toggle the UI between normal and edit modes.
-   **`@Binding`**: Bindings are used to pass an `Attendee` object to the `RenameSheet` and `IconPicker` so they can modify the original data.

## 5. Data Modeling

-   **`Attendee`**: An `Identifiable`, `Codable` struct that represents a single person. It includes properties for `id`, `name`, `avatar` (an emoji string), and `state` (an integer representing their status).

## 6. Data Persistence

-   **`UserDefaults`**: The application persists the list of attendees directly to `UserDefaults`.
-   **`AttendeeViewModel`**: The view model contains the `saveAttendees()` and `loadAttendees()` methods, which handle encoding the `attendees` array to JSON `Data` and decoding it back. There is no separate `PresetManager`.

## 7. User Interaction

-   **Status Change**: Tapping an `AvatarCard` cycles the attendee's `state` through three possible values, visually represented by a change in the card's background color.
-   **Edit Mode**: A long press on the main view toggles `isEditing` mode.
-   **Deleting Attendees**: In edit mode, a delete icon appears on each `AvatarCard`. Tapping it removes the attendee from the list.
-   **Adding Attendees**: A dedicated "Add" card is always present at the end of the grid to append a new, default attendee to the list.
-   **Editing Attendees**: A context menu on each `AvatarCard` provides options to "Rename" or "Change Icon", which present the appropriate sheet or popover.
