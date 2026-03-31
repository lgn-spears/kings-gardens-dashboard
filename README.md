# King's Gardens — Staff Dashboard

Real-time collaborative dashboard for King's Gardens landscaping crew in Lexington, KY.

## Features

- **Live weather** from NWS + personal Ambient Weather station
- **Frost/freeze alerts** with automatic greenhouse closure warnings
- **Team bulletin board** — post updates visible to all crew
- **Weekly shift schedule** — crew assignments with job sites
- **Delivery calendar** — track vendor visits and truck deliveries
- **Growing Degree Days** — phenology milestones with prior-year comparison

## Setup

### 1. Create a Firebase project

1. Go to [Firebase Console](https://console.firebase.google.com)
2. Click **Add project** — name it (e.g., "kings-gardens-dashboard")
3. Skip Google Analytics (optional)
4. Once created, click the **web** icon (`</>`) to add a web app
5. Copy the `firebaseConfig` object

### 2. Enable services

- **Authentication**: Go to Authentication → Sign-in method → Enable **Email/Password**
- **Firestore**: Go to Firestore Database → Create database → Start in **production mode**

### 3. Set Firestore security rules

In Firestore → Rules, paste:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if request.auth != null;
    }
  }
}
```

### 4. Add your config

Open `index.html` and replace the placeholder `firebaseConfig` object (near the top of the `<script>` tag) with your project's config.

### 5. Deploy

Push to GitHub. GitHub Pages serves `index.html` automatically.

## Tech

Single `index.html` — no build step. Firebase JS SDK loaded via CDN. Hosted on GitHub Pages.
