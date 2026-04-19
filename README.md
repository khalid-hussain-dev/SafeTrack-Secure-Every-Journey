# SafeTrack

SafeTrack is a native Android app for secure, trusted-contact location sharing. It combines Firebase Authentication, Cloud Firestore, Firebase Realtime Database, Firebase Storage, Room, and MapLibre to let a verified user share their live location, manage trusted contacts, and view tracked devices on a map.

The current codebase is centered around Java `Activity` classes plus a foreground `LocationService`. Live updates are optimized with movement-aware upload rules, and the app temporarily increases update frequency when a trusted contact is actively viewing a device.

## What The App Currently Does

- Authenticates users with Firebase email/password sign-in.
- Requires email verification before access to the main app.
- Creates a SafeTrack identity backed by the Firebase UID.
- Lets users add trusted contacts by SafeTrack ID.
- Publishes live device location from a foreground service, including background operation.
- Restarts location sharing after reboot, app replacement, or task removal when tracking is enabled.
- Displays live location in a MapLibre map view.
- Tracks watcher heartbeats so active viewers can trigger higher-frequency location updates.
- Stores a rolling 48-hour local location history in Room.
- Supports remote microphone access requests and uploads recordings to Firebase Storage with Firestore metadata.
- Supports account deletion with cleanup across Firestore, Realtime Database, Firebase Storage, and local Room data.

## Current Limitations

- The camera access button is present in the UI, but there is no implemented remote camera capture or upload flow yet.
- Firestore rules include a `firebase_ids/{userId}/locations` collection, but the current Android client stores recent location history locally in Room and publishes live location through Realtime Database instead.
- There are no substantial automated tests in the repository yet.

## Tech Stack

| Area | Implementation |
| --- | --- |
| Language | Java 11 |
| Android target | `minSdk 24`, `targetSdk 36`, `compileSdk 36` |
| UI | Android Views + Material Components |
| Map | MapLibre Android SDK |
| Auth | Firebase Authentication |
| Structured app data | Cloud Firestore |
| Live presence and live location | Firebase Realtime Database |
| File uploads | Firebase Storage |
| Local persistence | Room |
| Device location | Google Play Services Fused Location Provider |

## Main Application Flow

1. `LauncherActivity` checks the current Firebase session.
2. Unauthenticated users go to `LoginActivity`; newly created users go through `RegisterActivity` and `EmailVerificationActivity`.
3. Verified users enter `MainActivity`.
4. `MainActivity` ensures the Firestore account record exists and syncs trusted contacts into Realtime Database.
5. Once location permissions are granted, `LocationService` starts as a foreground service.
6. `LocationService` reads device location, stores recent samples in Room, and publishes live location to Realtime Database.
7. `MainActivity` listens to live location updates for either the current user or a selected trackee and renders them on the map.
8. When a trusted contact is actively watching a trackee, heartbeat entries in Realtime Database enable boost mode so uploads happen more frequently.

## Data Responsibilities

### Firebase Authentication

- Handles email/password sign-in.
- Gates app access behind verified email status.

### Cloud Firestore

- `directory/{userId}`: searchable SafeTrack ID directory.
- `firebase_ids/{userId}`: account profile document.
- `firebase_ids/{userId}/users/{contactId}`: trusted contacts saved by the user.
- `firebase_ids/{userId}/mic_stream/control`: remote microphone control flag.
- `firebase_ids/{userId}/mic_stream/{streamId}`: uploaded microphone recording metadata.

### Firebase Realtime Database

- `live_locations/{userId}`: latest live latitude, longitude, speed, accuracy, mode, and timestamp.
- `watcher_heartbeats/{trackeeId}/{trackerId}`: active viewer heartbeats.
- `trusted_contacts/{ownerId}/{contactId}`: trusted-contact access map used for read authorization.

### Room

- `location_history`: local rolling history of recent points for the signed-in user.

### Firebase Storage

- `firebase_ids/{userId}/mic_stream/data/*`: recorded audio files uploaded after a remote mic request completes.

## Key Classes

- `LauncherActivity`: entry router based on auth and verification state.
- `LoginActivity`, `RegisterActivity`, `EmailVerificationActivity`: authentication flow.
- `MainActivity`: map UI, permission handling, live location display, remote mic control, diagnostics, and settings.
- `ChangeTrackee`: add/remove trackees and open tracking mode for a selected contact.
- `LocationService`: foreground background-location publisher with watcher-aware boost logic.
- `LocationReceiver`: restarts the location service after reboot, app update, or scheduled restart.
- `SafeTrackRepository`: Firestore path helpers and account bootstrap.
- `LiveLocationRepository`: Realtime Database path helpers and live-location payload builder.
- `LocationHistoryStore`: Room-backed local retention of recent location samples.
- `AccountDeletionManager`: coordinated deletion across Firebase + local storage.

## Project Structure

```text
app/
  src/main/java/com/xiiidimensions/safetrack/
    AccountDeletionManager.java
    AppPreferences.java
    ChangeTrackee.java
    EmailVerificationActivity.java
    LauncherActivity.java
    LiveLocationRepository.java
    LocationHistory*.java
    LocationReceiver.java
    LocationService.java
    LoginActivity.java
    MainActivity.java
    RegisterActivity.java
    SafeTrackDatabase.java
    SafeTrackRepository.java
    Trackee.java
    TrackeeAdapter.java
    TrustedContactSync.java
  src/main/assets/
    osm_raster_style.json
firestore.rules
database.rules.json
firebase.json
```

## Setup

### Prerequisites

- Android Studio with Android SDK 36 installed.
- A Firebase project with:
  - Authentication
  - Cloud Firestore
  - Realtime Database
  - Firebase Storage
- A physical Android device or emulator with location support.

### Firebase Configuration

1. Create a Firebase project for the app package `com.xiiidimensions.safetrack`.
2. Enable Email/Password sign-in in Firebase Authentication.
3. Download `google-services.json` and place it in [app/google-services.json](app/google-services.json).
4. Apply the Firestore rules from [firestore.rules](firestore.rules).
5. Apply the Realtime Database rules from [database.rules.json](database.rules.json).

### Local Run

1. Open the project in Android Studio.
2. Sync Gradle.
3. Run the `app` module on a device or emulator.

From the terminal, you can also build with:

```powershell
.\gradlew.bat assembleDebug
```

### Device Permissions

For the full live-tracking flow, the app needs:

- Foreground location permission
- Background location permission (`Allow all the time` on supported Android versions)
- Notification permission on Android 13+
- Microphone permission for remote mic capture

## Security Notes

- The repository includes both Firestore and Realtime Database rules files. Keep them aligned with the client data model as the app evolves.
- Trusted-contact access is enforced through Firestore ownership rules and Realtime Database trusted-contact maps.
- Account deletion currently clears the signed-in user's Firestore documents, Realtime Database live-sharing nodes, uploaded microphone data, and local Room history.

## Architecture Diagram

The Mermaid source is also available in [docs/architecture-flow.mmd](docs/architecture-flow.mmd).

```mermaid
flowchart TD
    Launcher["LauncherActivity"] --> AuthCheck{"FirebaseAuth current user?"}
    AuthCheck -->|No session or anonymous| Login["LoginActivity"]
    AuthCheck -->|Signed in, not verified| Verify["EmailVerificationActivity"]
    AuthCheck -->|Verified user| MainSelf["MainActivity (own device mode)"]

    Login -->|Email/password sign in| FirebaseAuth["Firebase Authentication"]
    Register["RegisterActivity"] -->|Create account + send verification| FirebaseAuth
    Login -->|Open sign up| Register
    FirebaseAuth --> Verify
    Verify -->|Email verified| MainSelf

    MainSelf -->|Ensure account document| Firestore["Cloud Firestore"]
    MainSelf -->|Sync trusted contacts| RTDB["Realtime Database"]
    MainSelf -->|Request permissions + start| LocationService["LocationService"]
    MainSelf -->|Listen for own live location| RTDB
    MainSelf --> Map["MapLibre map"]

    Trackees["ChangeTrackee"] -->|Lookup SafeTrack ID in directory| Firestore
    Trackees -->|Save/remove trusted contacts| Firestore
    Trackees -->|Mirror trusted contact access| RTDB
    MainSelf -->|Open trackee manager| Trackees

    MainViewer["MainActivity (tracking mode)"] -->|Listen to trackee live location| RTDB
    MainViewer -->|Write watcher heartbeat| RTDB
    MainViewer --> Map
    Trackees -->|Launch tracking mode| MainViewer

    Receiver["LocationReceiver"] -->|Boot, package replaced, restart action| LocationService
    LocationService -->|Read device position| Fused["FusedLocationProviderClient"]
    Fused -->|Location samples| LocationService
    LocationService -->|Save recent history (48h)| Room["Room / location_history"]
    LocationService -->|Publish latest live location| RTDB
    RTDB -->|watcher_heartbeats| LocationService

    MainViewer -->|Mic access request| Firestore
    Firestore -->|mic_stream/control listener| MainSelf
    MainSelf --> Recorder["MediaRecorder"]
    Recorder -->|Upload audio file| Storage["Firebase Storage"]
    Storage -->|Download URL| Firestore
```

## Mermaid Source

The raw diagram source is stored separately in [docs/architecture-flow.mmd](docs/architecture-flow.mmd) for reuse in other documentation or diagram tooling.
