# Count

Count is a single-page web application for planning, logging, and reviewing
strength-training workouts. It combines a complete workout logger with an
integrated AI coaching assistant ("GymBro") that gives feedback based on the
user's recorded training history.

The application is implemented as a single, self-contained HTML file written in
vanilla JavaScript, with no frameworks, build tooling, or external package
dependencies.

## Contents

- [Overview](#overview)
- [Features](#features)
- [Technical Implementation](#technical-implementation)
- [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [Running the Project](#running-the-project)
- [Future Improvements](#future-improvements)
- [Credits](#credits)

## Overview

Consistent progress in resistance training depends on tracking performance over
time. Count provides a structured workflow for this:

1. The user selects a built-in workout template or creates a custom one.
2. During a session, every set is logged with its weight, repetitions, and type,
   while a rest timer manages pacing between sets.
3. On completion, the application calculates session statistics, detects personal
   records, and stores the workout to the user's history.
4. Past sessions can be reviewed, edited, repeated, or shared, and the AI coach
   can answer training questions using the recorded data as context.

All user data — workout history, templates, preferences, and coaching
conversations — is persisted to a cloud account through the GammalTech Web SDK.

## Features

### Authentication and Data Persistence

Users authenticate through the GammalTech SDK, which handles login and token
verification. All application data is stored against the user's cloud account,
so history and settings are available on any device. If the SDK cannot be
reached, the application installs a `localStorage`-based fallback so the
interface remains functional in a local demo mode.

### Onboarding

First-time users complete a three-step setup that captures their name, primary
training goal, and age. Six goals are supported (build muscle, gain strength,
lose fat, body recomposition, athletic performance, and general fitness). The
selected goal is later included in the context sent to the AI coach.

### Home Dashboard

The home screen presents a personalized greeting, an AI-generated motivational
message tailored to the user's current situation, and three summary cards
showing the current training streak, total completed workouts, and the muscle
group trained with the most volume in the past week. A prompt to train is shown
when the user has been inactive for two or more days.

### Workout Templates

The application ships with six built-in templates (Push, Pull, Legs, Full Body,
Upper, and Lower). Users can also create custom workouts from the exercise
library, as well as duplicate, rename, edit, and delete them. Free accounts may
keep up to three custom templates; the limit is removed with Premium.

### Exercise Library

The library contains more than sixty exercises across six muscle groups (chest,
back, shoulders, arms, legs, and core), each labelled with its equipment type.
The list supports debounced text search and muscle-group filtering. Each exercise
can display an animated form-demonstration GIF, retrieved and cached from the
ExerciseDB API, along with an AI-generated form tip.

### Set Logging

The logging screen records each exercise set in detail:

- Entry of weight and repetitions per set, with a control to mark a set complete.
- Four set types — warm-up, working, drop, and failure — selected by tapping the
  set badge to cycle through them.
- A reference column showing the weight and repetitions recorded for that set in
  the previous session.
- Automatic placeholders that pre-fill each set with the user's previous values.
- Automatic copying of an entered value into the remaining empty sets of the same
  exercise.

### Rest Timer

A rest timer starts automatically after each completed set. It offers preset
durations (30, 60, 120, and 180 seconds) and a custom picker in five-second
steps. The remaining time is shown as a circular countdown, time can be added or
removed in 30-second increments while it runs, and an audio cue plays when the
period ends. The end-of-rest cue differs depending on whether the next action is
another set or a new exercise.

### Exercise Management

Exercises can be managed freely during a session:

- Adding exercises from the library through a searchable sheet.
- Replacing an exercise while keeping the existing set count.
- Removing an exercise from the current workout.
- Reordering exercises by drag-and-drop on desktop, or by long-press drag on
  touch devices.
- Deleting an individual set by swiping the set row.
- Adding a free-text notes field per exercise for cues such as seat height or
  tempo.

### Per-Exercise Preferences

Each exercise can override the global settings with its own weight unit
(kilograms or pounds) and barbell type. Five bar types are supported, each with
its correct bar weight (Olympic, Short, EZ, Hex, and none).

### Session Controls

An elapsed-time counter runs for the duration of the workout, and the workout
name can be edited inline at any point during the session.

### Draft Auto-Save and Recovery

While a workout is in progress, its state is saved as a draft every thirty
seconds. If the application is closed before the workout is finished, the next
session offers to restore the draft. Drafts older than twenty-four hours are
discarded automatically.

### Finishing a Workout

When the user finishes, the application checks for incomplete sets. If any are
found, it offers to complete them automatically using previous best values, to
save only the sets that were logged, or to return to the workout. It then
calculates total and per-muscle training volume, updates the training streak,
detects whether any exercise reached a new personal record, and records the
session to history.

### Post-Workout Summary

After a workout, a summary screen reports the duration, number of completed sets,
and total volume, and displays an AI-generated review of the session that
references specific exercises and figures.

### Workout History

All completed sessions are listed in the history screen, grouped by week and
loaded in pages of twenty for performance. Selecting an entry opens a detailed
view, and a per-entry menu provides options to edit the session, save it as a
template, share it, or delete it. Swipe gestures are also supported: swiping a
history entry one way repeats the workout with its values pre-filled, and the
other way deletes it.

### Sharing

A workout can be shared as formatted text through the device share dialog or the
clipboard. It can also be shared as a link: the workout is encoded into a
base64 fragment in the URL, which a recipient can open to import the workout as
one of their own templates.

### AI Coach (Premium)

The coach screen provides a chat interface with the GymBro assistant. The system
prompt sent to the model is assembled from the user's profile, a summary of their
twenty most recent sessions, and any plateaus detected automatically (an exercise
held at the same weight across three or more sessions). Conversations are saved
to the user's account so that context is retained between sessions.

### Premium

Premium is a one-time purchase processed through the GammalTech wallet. It
removes the custom-template limit and unlocks the AI coach.

### Audio Feedback

The application provides audio cues for completing a set, finishing a rest
period, and completing a workout, using a combination of embedded audio clips
and tones synthesized with the Web Audio API.

## Technical Implementation

### Application Structure

The entire application — markup, styles, logic, and audio assets — is contained
in a single `index.html` file. A global `STATE` object holds all runtime data:
the current user, their stored data, the active workout, and transient UI state.
Each screen is produced by a dedicated function that returns an HTML string.

### Routing

Navigation is handled by a hash-based router. A `ROUTES` table maps each URL
hash to a screen function and an authentication requirement. The `navigate()`
function updates the hash and triggers a render, and a listener on the
`hashchange` event ensures the browser's back and forward controls behave
correctly.

### Rendering

A central `render()` function reads the current hash, selects the matching
screen, enforces the authentication requirement, and writes the resulting HTML
into the page. To avoid unnecessary work, interactive parts of the workout
screen are updated in place — individual exercise cards and set rows are
re-rendered on their own rather than redrawing the entire screen.

### Data Persistence and the Save Queue

All writes to cloud storage pass through a save queue. The queue retains only the
most recent pending payload and performs one save at a time. This prevents a
slower request and a faster request issued close together from completing out of
order and overwriting newer data — a safeguard against silent data loss.

### Resilience

The application is designed to degrade gracefully. If the backend SDK fails to
load, a local stand-in backed by `localStorage` is installed so the interface
still works. If the AI service or the exercise-GIF service is unavailable, the
application falls back to pre-written messages and placeholder graphics rather
than failing visibly.

### Other Implementation Details

- Search inputs are debounced to limit DOM updates while typing.
- Swipe animations are driven by `requestAnimationFrame` for smooth movement.
- Retrieved exercise GIFs are cached in memory, with a short back-off period
  after a failed request.
- User-supplied and AI-supplied text is HTML-escaped before insertion into the
  page.

## Technology Stack

- HTML and CSS — hand-written, mobile-first, dark theme.
- JavaScript (ES6+) — vanilla, with no framework or build step.
- GammalTech Web SDK v3.0 — authentication, cloud data storage, payments, and AI.
- ExerciseDB API (via RapidAPI) — exercise demonstration GIFs.
- Web Audio API — synthesized audio cues.

## Project Structure

```
count/
├── index.html      The complete application (HTML, CSS, JavaScript, audio)
├── README.md       Project documentation
└── .gitignore
```

The `index.html` file is fully self-contained; no additional files are required
for the application to run.

## Running the Project

Clone the repository and open the application:

```
git clone https://github.com/haytham-cod/count.git
cd count
```

Open `index.html` in any modern web browser.

For full functionality — authentication, cloud synchronization, payments, and
the AI coach — the application must be served from a domain registered with
GammalTech. When that backend is unavailable, the application runs in a local
demo mode using browser storage, which is sufficient for reviewing the interface
and user experience.

## Future Improvements

- Visual charts for volume and strength trends over time.
- Per-exercise rest-timer durations remembered across sessions.
- Expanded comparison of current and past performance for each exercise.
- Optional offline-first support.

## Credits

- Authentication, cloud storage, payments, and AI are provided by Gammal Tech.
- Exercise demonstration GIFs are provided by ExerciseDB via RapidAPI.

## Author

Mohammed Haytham

- Email: mohammedgouda639@yahoo.com
- LinkedIn: https://www.linkedin.com/in/mohammed-haytham-59810a3b7
