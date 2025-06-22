# Age of Kindness - Backend

[![Status](https://img.shields.io/badge/status-active-success.svg)]()

This repository contains the serverless backend code for the **Age of Kindness** platform, built using [Google Cloud Functions for Firebase](https://firebase.google.com/docs/functions).

## About this Repository

This backend is responsible for all server-side logic, including:

- **API Endpoints:** Providing a secure interface for the Unity client application.
- **Database Operations:** Interacting with Firestore to store and retrieve user messages.
- **AI Content Vetting:** A multi-layered moderation system to ensure all messages are positive and safe.
- **Geospatial Queries:** Leveraging Firestore's capabilities to handle location-based data.

The client-side Unity application that consumes this backend is located in the [`app` repository](https://github.com/AgeOfKindness/app).

## Technology Stack

- **Platform:** Firebase
- **Core Services:**
  - **Cloud Functions:** For serverless API endpoints (written in TypeScript/Node.js).
  - **Firestore:** As the primary real-time, NoSQL database.
  - **Authentication (optional):** Firebase Authentication for user management.
- **External APIs:**
  - **Google Perspective API / OpenAI Moderation:** For content toxicity analysis.
  - **Other NLP services (e.g., Google Cloud Natural Language):** For sentiment analysis.

## Project Status

**Pre-Alpha:** The project is in the initial scaffolding phase. The current focus is on:
1.  Initializing the Firebase project and setting up the local emulator suite.
2.  Defining the core data models (e.g., `Message`) in Firestore.
3.  Developing the first Cloud Function for submitting a new message.
4.  Implementing the first layer of the AI content vetting pipeline.

## Getting Started (Development)

To run this project locally, you will need:
1.  **Node.js** and **npm/yarn/pnpm**.
2.  The **Firebase CLI** (`npm install -g firebase-tools`).
3.  A Firebase project created on the [Firebase Console](https://console.firebase.google.com/).
4.  Authenticated the Firebase CLI (`firebase login`).

**Installation & Setup:**

1.  Clone this repository: `git clone https://github.com/AgeOfKindness/backend.git && cd backend`
2.  Install dependencies: `npm install` (or your package manager's equivalent).
3.  Configure your local environment to connect to your Firebase project: `firebase use <your-project-id>`.
4.  Set up environment variables for your Cloud Functions (e.g., API keys for vetting services) using `firebase functions:config:set`.
5.  Run the local Firebase Emulator Suite: `firebase emulators:start`. This will start local instances of Functions, Firestore, etc., for safe development.

## Contributing

We welcome contributions! Please see the main `CONTRIBUTING.md` file in the central `AgeOfKindness/.github` repository for details.

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details.
