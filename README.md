# HabitSync

HabitSync is a production-grade, cross-platform, social habit-tracking ecosystem. It consists of a modern, reactive Android client built with **Kotlin** and **Jetpack Compose (Material 3)**, backed by **Firebase Auth and Firestore** (with local offline persistence), and a dedicated Node.js **Express backend** deployed on Vercel that interacts with an **S3-compatible Storj bucket** to facilitate secure, proof-based habit verification.

---

## 🏗 Ecosystem Architecture

The repository is organized as a monorepo split into two primary components:
1. **`app/`**: A Kotlin Android application utilizing the Model-View-ViewModel (MVVM) architecture pattern, Repository pattern, and Jetpack Compose for UI.
2. **`backend/`**: An Express.js microservice deployed on Vercel, functioning as a helper to verify Firebase client identities and generate secure, pre-signed S3 upload URLs for habit completion proof storage.

```mermaid
graph TD
    subgraph Client [Android Client - MVVM & Jetpack Compose]
        Auth[AuthRepository]
        Habits[FirebaseHabitRepository]
        Friends[FriendsRepository]
        Challenges[ChallengesRepository]
        Proof[ProofStorageRepository]
    end

    subgraph FirebaseServices [Firebase Backend]
        FAuth[Firebase Authentication]
        Firestore[(Firestore Database)]
    end

    subgraph UploadService [Node.js Proof Backend]
        Express[Express Server]
        AdminSDK[Firebase Admin SDK]
        S3Client[AWS S3 SDK]
    end

    subgraph Storage [Storj Cloud Storage]
        Bucket[(S3-compatible Bucket)]
    end

    %% Client Interactions
    Auth -->|Email & Password / Credentials| FAuth
    Habits -->|Real-time Sync & Persistence| Firestore
    Friends -->|Friend Requests & Leaderboard| Firestore
    Challenges -->|Challenges & Real-time Chat| Firestore
    Proof -->|1. Request Signed URL| Express
    Proof -->|3. Direct Binary Upload| Bucket

    %% Backend Interactions
    Express -->|2. Verify ID Token| FAuth
    Express -->|4. Generate Presigned S3 URL| S3Client
    S3Client -->|S3 Protocol| Bucket
```

---

## 🗄️ Firestore Database Schema

Firestore contains the following core collections. Database structures are strictly guarded by security constraints defined in [firestore.rules](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/firestore.rules).

### 1. `users` Collection
Stores comprehensive profiles for authenticated users.
*   **Path**: `/users/{userId}`
*   **Structure**:
    ```typescript
    interface UserProfile {
      firstName: string;
      lastName: string;
      username: string; // Normalized lowercase
      displayName: string;
      bio: string;
      email: string;
      gender: string;
      profileImageUrl: string | null;
      weightKg: number;
      heightCm: number;
      totalHabitsCreated: number;
      longestStreakEver: number;
      challengesCompleted: number;
      freezeTokensThisMonth: number; // Defaults to 2
      freezeTokensMonthKey: string; // Format: "YYYY-MM"
      updatedAt: Timestamp;
    }
    ```
*   **Subcollections**:
    *   `habits`: The user's tracking log.
        *   **Path**: `/users/{userId}/habits/{habitId}`
        *   **Structure**:
            ```typescript
            interface Habit {
              title: string;
              streak: number;
              isCompletedToday: boolean;
              lastCompletedDate: number | null; // Epoch millisecond
              proofImageUrl: string | null;
              category: "CUSTOM" | "READING" | "HYDRATION" | "FITNESS" | "SLEEP";
              reminderTime: string; // HH:MM
              visibility: "PUBLIC" | "PRIVATE";
              completionDates: number[]; // Array of Epoch milliseconds
            }
            ```
    *   `notifications`: System-triggered, in-app notification center items.
        *   **Path**: `/users/{userId}/notifications/{notificationId}`
        *   **Structure**:
            ```typescript
            interface Notification {
              title: string;
              message: string;
              type: "friend_request" | "challenge_invite" | "challenge_activity" | "evening_reminder" | "streak_milestone";
              isRead: boolean;
              timestamp: Timestamp;
            }
            ```
    *   `friendRequests`: Tracks incoming invitations to form friendship links.
        *   **Path**: `/users/{userId}/friendRequests/{fromUserId}`
        *   **Structure**:
            ```typescript
            interface FriendRequest {
              fromUserId: string;
              fromUsername: string;
              fromDisplayName: string;
              createdAtMillis: number;
            }
            ```
    *   `friends`: Stores direct references to accepted friends.
        *   **Path**: `/users/{userId}/friends/{friendUserId}`
        *   **Structure**:
            ```typescript
            interface Friend {
              userId: string;
              username: string;
              displayName: string;
              profileImageUrl: string | null;
              joinedAtMillis: number;
            }
            ```
    *   `badges`: Achievements unlocked by the user.
        *   **Path**: `/users/{userId}/badges/{badgeId}`
        *   **Structure**:
            ```typescript
            interface Badge {
              title: string;
              description: string;
              earnedAtMillis: number;
            }
            ```

### 2. `usernames` Collection
Enforces unique, case-insensitive global username registration.
*   **Path**: `/usernames/{username}` (document ID is the lowercase username)
*   **Structure**:
    ```typescript
    interface UniqueUsername {
      userId: string;
      createdAt: Timestamp;
    }
    ```

### 3. `challenges` Collection
Facilitates social, multi-user competitive habits with real-time synced progress.
*   **Path**: `/challenges/{challengeId}`
*   **Structure**:
    ```typescript
    interface Challenge {
      name: string;
      rule: string;
      durationDays: number;
      category: "CUSTOM" | "READING" | "HYDRATION" | "FITNESS" | "SLEEP";
      creatorId: string;
      creatorUsername: string;
      createdAtMillis: number;
      endAtMillis: number | null;
      status: "ACTIVE" | "ENDED";
      invitees: string[]; // List of user IDs invited
      participants: {
        [userId: string]: {
          username: string;
          displayName: string;
          profileImageUrl: string | null;
          dailyCompletions: {
            [dayIndex: string]: { // Day indexes starting from "0"
              completed: boolean;
              completedAtMillis: number;
              proofImageUrl: string | null;
            }
          }
        }
      }
    }
    ```
*   **Subcollections**:
    *   `comments`: Real-time chat messages inside the challenge lobby.
        *   **Path**: `/challenges/{challengeId}/comments/{commentId}`
        *   **Structure**:
            ```typescript
            interface ChallengeComment {
              senderId: string;
              senderUsername: string;
              senderDisplayName: string;
              senderPhotoUrl: string | null;
              text: string;
              timestampMillis: number;
            }
            ```

### 4. `leaderboard` Collection
A global caching registry aggregating total user scores for optimized read operations.
*   **Path**: `/leaderboard/{userId}`
*   **Structure**:
    ```typescript
    interface LeaderboardEntry {
      username: string;
      displayName: string;
      profileImageUrl: string | null;
      score: number; // Aggregated score metric
      lastUpdated: Timestamp;
    }
    ```

---

## 🔐 Firestore Security Policies

Security constraints are configured in [firestore.rules](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/firestore.rules). They implement the following rules:

1.  **User Profiles (`/users/{userId}`)**:
    *   Any authenticated user can read another user's profile.
    *   Only the owner (`request.auth.uid == userId`) can write to their own profile.
    *   Username changes transactionally enforce regex matches (`^[a-z0-9]{3,12}$`).
2.  **User Habits & Notifications**:
    *   Only the owner of the user profile can read or write documents under `habits` or `notifications`.
3.  **Global Leaderboard**:
    *   Any authenticated user can read the leaderboard.
    *   Updates can only be made if the user is writing to their own record (`request.auth.uid == userId`).
4.  **Global Usernames Registry**:
    *   Read access is public to allow lookup during search/signup.
    *   Write access requires authentication, and the record's payload `userId` must match the authenticated client's UID.
5.  **Social Friendship System**:
    *   Authenticated users can read, update, or write to `friendRequests` if they are the sender or receiver.
    *   Bidirectional acceptance rules ensure a user can only write to their own `/friends/{friendId}` document.
6.  **Challenges**:
    *   Authenticated users can create, read, and delete challenges.
    *   Writing to `participants` data is heavily constrained: users can only modify their own participation state inside the challenge document.

---

## 📡 Backend API Reference

The backend operates as an Express API configured in [backend/server.js](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/backend/server.js). It serves three critical routes:

### 1. `POST /proof-upload-url`
Generates a pre-signed S3 URL pointing to the Storj gateway, enabling direct binary uploads from the client.
*   **Authentication**: Requires HTTP Bearer Authorization (`Authorization: Bearer <Firebase_ID_Token>`).
*   **Request Headers**:
    ```http
    Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtp...
    Content-Type: application/json
    ```
*   **Request Body**:
    ```json
    {
      "habitId": "my-morning-run-habit",
      "contentType": "image/jpeg"
    }
    ```
*   **Response (Success - 200 OK)**:
    ```json
    {
      "uploadUrl": "https://gateway.storjshare.io/habitsync-proofs/proof_123456_my-morning-run.jpg?AWSAccessKeyId=...",
      "fileUrl": "https://<backend_domain>/proof-file/proof_123456_my-morning-run.jpg",
      "objectKey": "proof_123456_my-morning-run.jpg"
    }
    ```
*   **Internal Mechanics**:
    1.  Uses `verifyFirebaseAuth` middleware to parse and validate the JWT via `firebase-admin`.
    2.  Checks for the presence of variables `STORJ_ACCESS_KEY`, `STORJ_SECRET_KEY`, `STORJ_ENDPOINT`, and `STORJ_BUCKET`.
    3.  Instantiates `S3Client` and calls `PutObjectCommand` inside `getSignedUrl` from `@aws-sdk/s3-request-presigner` with an expiration window of 300 seconds (5 minutes).
    4.  Saves the payload with the unique file name structure: `proofs/<userId>/<habitId>/<timestamp>.jpg`.

### 2. `GET /proof-file/*`
Generates a temporary, 2-minute pre-signed redirect URL to securely serve uploaded images without exposing public buckets.
*   **Authentication**: None.
*   **Response (Redirect - 302 Found)**:
    Redirects user directly to the Storj download URL containing the signed query token.
    ```http
    Location: https://gateway.storjshare.io/habitsync-proofs/proofs/123456/habitId/timestamp.jpg?AWSAccessKeyId=...&Expires=...
    ```

### 3. `GET /health`
*   **Response (200 OK)**:
    ```json
    {
      "ok": true
    }
    ```

---

## 📱 Android Client Subsystems

The Android application is built with modern, jetpack-first development patterns:

### 1. App Entry & Onboarding Workflow
Managed by [AppEntryViewModel.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/viewmodel/AppEntryViewModel.kt) and [OnboardingViewModel.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/viewmodel/OnboardingViewModel.kt).
*   **Cold Start Routing**: The initial route resolves in `HabitSyncNavHost.kt`.
*   **Persistence Store**: App utilizes Android **DataStore Preferences** (`onboarding_prefs` implemented in [OnboardingPreferences.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/data/local/OnboardingPreferences.kt)) to track if the introductory pager flow has been completed.
*   **Flow States**:
    ```
                 [ Cold Start App ]
                         │
                         ▼
             Is Onboarding Completed?
               ├── No ──> [ Onboarding Screen ] ──> Mark Done ──> [ Login Screen ]
               └── Yes ─> Is Firebase Session Active?
                            ├── No ──> [ Login Screen ]
                            └── Yes ─> [ Home Overview Screen ]
    ```

### 2. Authentication Flow
Managed by [AuthRepository.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/data/repository/AuthRepository.kt) and [AuthViewModel.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/viewmodel/AuthViewModel.kt).
*   **Credential Integration**: Supports Google API **`CredentialManager`** to record and automatically fill out saved passwords.
*   **Offline Support**: Firebase Auth checks local token states for instantaneous session restorations.
*   **Recovery Screen**: Users recover accounts using an offline recovery validation sequence verifying their custom `username`, `email`, and date of birth `dobMillis` against matching transactional Firestore records.

### 3. Habits Engine
Managed by [FirebaseHabitRepository.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/data/repository/FirebaseHabitRepository.kt) and [HomeViewModel.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/viewmodel/HomeViewModel.kt).
*   **Offline Persistence**: Initialized in `HabitSyncApp.kt`, enabling SQLite-backed query resolution and optimistic UI updates when disconnected.
*   **Default Presets**: Includes quick-start templates configured with specific categories and types:
    *   *Reading* (READING category, 30-minute target)
    *   *Drink Water* (HYDRATION category, 3-liter target)
    *   *Running* (FITNESS category, 2 km target)
    *   *Sleep Early* (SLEEP category, "Before 11 PM" target)
*   **Proof Upload Sequence**:
    1.  User updates habit state as completed.
    2.  If photographic proof is provided (Camera/Gallery bitmap), [ProofStorageRepository.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/data/repository/ProofStorageRepository.kt) requests an upload URL from the Vercel backend.
    3.  Once the signed URL is returned, the app uploads the binary payload as `image/jpeg` directly to the S3 bucket using a `PUT` operation.
    4.  The habit document in Firestore is updated with the returned redirect URL (`/proof-file/<key>`).
    5.  A social completion event is created in the global `/feed` matching the user's progress.

### 4. Social Networking & Real-time Leaderboards
Managed by [FriendsRepository.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/data/repository/FriendsRepository.kt) and [FriendsViewModel.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/viewmodel/FriendsViewModel.kt).
*   **Bidirectional Acceptance**: Friend requests utilize Firestore transactional batches. Upon approval, documents are concurrently created in `/users/{userId}/friends/{friendId}` and `/users/{friendId}/friends/{userId}`, and the incoming request is deleted.
*   **Leaderboard Aggregation**: Ranks the current user and their friends.
*   **Performance Optimization**: Because Firestore's `whereIn` operator caps search parameters at **10 items**, the app chunks query operations, aggregates results, computes delta ranking shifts against cached snapshots, and emits a sorted ranking list.

### 5. Multi-User Challenges Engine
Managed by [ChallengesRepository.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/data/repository/ChallengesRepository.kt) and [ChallengesViewModel.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/viewmodel/ChallengesViewModel.kt).
*   **Lobbies & Invites**: Challenges can be created with distinct start and end times. Creators send invites, which appear in recipients' `InboxSheet` collections.
*   **Real-time Lobby Chat**: Inside active challenge documents, participants can exchange real-time comments mapped to a Firebase snapshot listener.
*   **Progress Tracking**: Daily habit completions within the challenge are keyed by `dayIndex` (calculated relative to the start time) in the participants map.
*   **Settlement System**: Upon expiration, the client triggers `settleChallengeIfEnded()`. This evaluates the challenge completion score for all participants, marks the challenge as `status = "ENDED"`, and awards badges/points to the winning user.

### 6. Profile Operations & Freeze Tokens
Managed by [ProfileRepository.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/data/repository/ProfileRepository.kt) and [ProfileViewModel.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/viewmodel/ProfileViewModel.kt).
*   **Transactional Username Changes**: Releasing/changing a username checks the uniqueness index `/usernames/{username}`, deletes the previous index, updates the new index, and rewrites the user profile.
*   **Streak-Freeze Tokens**: Users are granted **2 freeze tokens** per month. The profile repository validates and resets these tokens using a key format (`YYYY-MM`). If a user misses a day, they can consume a token if the balance is greater than 0, maintaining their active habit streak.

### 7. Evening Reminders
Managed by [ReminderRepository.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/data/repository/ReminderRepository.kt).
*   **Trigger Mechanism**: Triggers locally on the client at or after **20:00 (8:00 PM)**.
*   **Idempotency**: Generates a unique notification ID format `evening_<year>_<day_of_year>`. If a document with this ID already exists under the user's notifications collection, it skips creation, avoiding duplicate alerts for the same day.

---

## 🎨 Design System & Theming

The client styling leverages modern Material 3 Jetpack Compose components customized via:
*   [Theme.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/ui/theme/Theme.kt): Declares system-wide `LightColorScheme` (dominated by gold-orange glow, charcoal-gray surfaces) and `DarkColorScheme` (optimized for deep black screens, amber indicators). Controls system status and navigation bar color synchronization.
*   [Color.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/ui/theme/Color.kt): Houses design tokens including dynamic primary glow accents (`Primary`/`DarkPrimary`), background, surfaces, success green, and gamified medal colors (gold, silver, bronze).
*   [Type.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/ui/theme/Type.kt): Specifies typographic scales for title layers (Sans-Serif Headlines with extra bold parameters and letter-spacing modifications) and body copy.

---

## ⚡ Optimizations & Caching Strategies

To ensure responsiveness, offline-first reliability, and database efficiency, HabitSync includes several core optimizations:

1. **Firestore Offline Persistence**: Real-time snapshot listeners register against the local SQLite database cache. This allows instant query resolution and immediate, optimistic UI updates. When network connectivity is restored, modifications are synced back asynchronously.
2. **Compose LazyLayout Structures**: Custom screens like `FeedScreen`, `LeaderboardScreen`, and `FriendsScreen` utilize `LazyColumn` for layout virtualization. Only currently visible items are rendered, dramatically minimizing composition cycles and memory overhead.
3. **Query Chunking & Merging**: Due to Firestore's limitation restricting `whereIn` arrays to a maximum of 10 items, the client chunk queries in sets of 10 inside [FriendsRepository.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/data/repository/FriendsRepository.kt#L137) and stitches the flows back together in real-time.
4. **Pre-Signed Upload URL Pattern**: Decoupling the file upload mechanism by utilizing pre-signed URLs offloads high-bandwidth binary uploads from the Node.js Express server. The client uploads images directly to Storj S3 gateway storage.

---

## 🚀 Deployment & CI/CD

### 1. Express Backend Deployment
The Express app is ready to deploy on **Vercel** as a Serverless Function configured in [backend/vercel.json](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/backend/vercel.json):
*   To deploy locally via command line:
    ```bash
    cd backend
    npm install -g vercel
    vercel --prod
    ```
*   Define all S3 bucket credentials and Firebase Project key JSONs inside Vercel's Environment Variables panel.

### 2. Firestore Indexes
Composite indexing rules must be created manually inside the Firebase Console to avoid exceptions when retrieving ranked leaderboard values:
*   **Collection ID**: `leaderboard`
*   **Fields Indexed**:
    *   `score` (Descending)
    *   `lastUpdated` (Descending)
*   **Query Scope**: Collection

---

## 🧪 Testing

> [!NOTE]
> **No automated testing scripts or source files were found in the codebase.**

To build out a comprehensive QA process for this project, the following architecture is recommended:
*   **Android Unit Tests**: Implement **JUnit 5** and **Mockk** in `app/src/test` to validate viewmodels (`HomeViewModel`, `AuthViewModel`) and repo mechanics using mock Firebase instances.
*   **Android Instrumentation UI Tests**: Use **Compose UI Test library** (`androidx.compose.ui:ui-test-junit4`) to test critical login, onboarding, and habit-completion flows on device emulators.
*   **Backend REST Tests**: Implement **Jest** paired with **Supertest** to mock incoming Firebase bearer credentials and assert responses on `/proof-upload-url` and `/proof-file/:key`.
*   **Firestore Rules Assertions**: Use the **@firebase/rules-unit-testing** suite to spin up the local emulator and verify that unauthorized clients cannot edit public usernames or access private habits.

---

## 👁️ Observability & Monitoring

*   **Express Server**: Uses basic stdout streams for error logging. Any middleware failure, S3 signing mismatch, or signature error issues a `500` or `401` status payload accompanied by standard console logs.
*   **Client Monitoring**: Screen views and database operations are wrapped in `runCatching` blocks. Errors caught asynchronously within Coroutine scopes update the corresponding `UiState` with an `errorMessage: String` property. This displays a localized snackbar or dialog message to the user instead of triggering a app crash.
*   **Health Verification**: A dedicated `/health` route is exposed on the Node backend to allow monitoring services to check API availability.

---

## ⚠️ Known Issues & Technical Debt

1.  **Hardcoded Configurations**: Default reminder alarms are hardcoded to `"20:00"` inside [FirebaseHabitRepository.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/data/repository/FirebaseHabitRepository.kt#L189).
2.  **No Image Compression**: Uploading user profile pictures and proof pictures pushes original, uncompressed byte arrays. This increases mobile data consumption and Storj storage billing.
3.  **Missing Backend Rate-Limiting**: The backend generates S3 URLs for any client holding a valid Firebase ID. There is no rate limiter, which leaves the bucket vulnerable to exhaustion attacks.
4.  **Race-Condition Vulnerable Username Checks**: Creating and registering usernames uses Firestore rules to check index document existence. However, the client lack true transactional locking across queries, allowing a small race condition window if two users submit the identical username simultaneously.
5.  **Capped Feeds**: The social feed retrieves a hard limit of 120 elements (`.limit(120)`) in [SocialRepository.kt](file:///C:/Users/user/Downloads/StrangerStrings-main/StrangerStrings-main/app/src/main/java/com/strangerstrings/habitsync/data/repository/SocialRepository.kt#L49) without support for cursor-based pagination. This will cause slow rendering as the feed database scales.

---

## 🤝 Contributing

Contributions to the HabitSync ecosystem are welcome. Please adhere to the following workflow:

1.  **Branch Naming Convention**:
    *   Feature branches: `feature/your-feature-name`
    *   Bug fixes: `bugfix/your-fix-name`
2.  **Formatting & Style**:
    *   Kotlin: Adhere to standard Kotlin coding guidelines. Run `./gradlew ktlintCheck` (if plugin is integrated) before committing.
    *   JavaScript: Use ES modules format and ensure code is ESLint clean.
3.  **Pull Requests**:
    *   Keep PRs focused and single-purpose.
    *   Include clear screenshots/descriptions for UI changes.

---

## 🗺️ Roadmap

- [ ] **Push Notifications**: Integrate Firebase Cloud Messaging (FCM) to trigger morning reminders and notifications for incoming friend requests/challenge comments.
- [ ] **Client-Side Image Resizing**: Compress proof photos on-device before uploading to reduce network traffic.
- [ ] **Rich Interactive Graphs**: Implement native charts visualizing habit completion percentages and streak progressions over time.
- [ ] **Enhanced Rate Limiting**: Introduce Redis-backed IP/ID rate-limiting on the Express endpoint to secure bucket calls.
- [ ] **Automated CI/CD Pipeline**: Set up GitHub Actions to run linters, build APKs, and auto-deploy Express routes to Vercel on commits to main.

---

## 📄 License & Maintainers

*   **License**: Not found in repository (custom/proprietary).
*   **Contributors**: Not found in repository.
