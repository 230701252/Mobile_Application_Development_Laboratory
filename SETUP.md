# Quick Setup Steps

The files in this folder are the **app-specific source code** (lib/, pubspec.yaml,
custom Android config). To get a runnable Flutter project, do this:

## 1. Generate the Flutter scaffold
```bash
cd path/to/placement_hub
flutter create . --org com.example --project-name placement_hub
```
This creates `android/`, `ios/`, `linux/`, `web/`, etc. wrappers without
overwriting `lib/` or `pubspec.yaml`.

## 2. Merge the included Android config
After `flutter create .`, manually merge these into the generated files:
- Take snippets from `android/app/build.gradle` (Firebase plugin + BOM) into the generated `android/app/build.gradle`
- Take snippets from `android/build.gradle` (google-services classpath) into the generated `android/build.gradle`
- The included `AndroidManifest.xml` shows the required permissions — add them to the generated manifest

## 3. Configure Firebase (one-time)
```bash
# Install FlutterFire CLI
dart pub global activate flutterfire_cli

# Log in & connect to a Firebase project
firebase login
flutterfire configure
```
This generates `lib/firebase_options.dart`. Then in `lib/main.dart`,
swap `Firebase.initializeApp()` for:
```dart
await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
```

In the Firebase console, enable:
- Authentication → Email/Password
- Firestore Database (test mode for development)
- Storage (test mode for development)

Place `google-services.json` (downloaded from Firebase) into `android/app/`.

## 4. Run on your USB-debugged phone
```bash
flutter pub get
flutter devices       # confirm your phone is listed
flutter run
```

That's it — the app installs and launches on your phone.

## Firestore data model

```
users/{uid}                      → name, email, role, branch, year, company
users/{uid}/connections/{otherUid}
users/{uid}/saved/{materialId}

materials/{id}                   → title, description, fileUrl, uploaderId,
                                   uploaderName, company, tags, likes[], commentCount
materials/{id}/comments/{cid}    → userId, userName, text, createdAt

requests/{id}                    → fromId, fromName, toId, status, createdAt

chats/{chatId}                   → participants[], lastMessage, updatedAt
chats/{chatId}/messages/{mid}    → senderId, text, createdAt
```

`chatId` is generated as the two user UIDs sorted and joined with `_`.

## Firestore security rules (paste into Firebase console)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    match /users/{uid} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && request.auth.uid == uid;
      match /{sub=**} { allow read, write: if request.auth != null && request.auth.uid == uid; }
    }
    match /materials/{mid} {
      allow read: if request.auth != null;
      allow create: if request.auth != null;
      allow update, delete: if request.auth != null && request.auth.uid == resource.data.uploaderId
        || (request.auth != null && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['likes','commentCount']));
      match /comments/{cid} {
        allow read: if request.auth != null;
        allow create: if request.auth != null;
      }
    }
    match /requests/{rid} {
      allow read, create: if request.auth != null;
      allow update: if request.auth != null && request.auth.uid == resource.data.toId;
    }
    match /chats/{cid} {
      allow read, write: if request.auth != null && request.auth.uid in resource.data.participants;
      allow create: if request.auth != null;
      match /messages/{mid} {
        allow read, create: if request.auth != null;
      }
    }
  }
}
```
