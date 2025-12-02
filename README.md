# Simple Realtime Chat App

A modern, single-channel realtime chat application built with SvelteKit and PocketBase.

## Tech Stack

- **Frontend**: SvelteKit (Svelte 5)
- **Backend**: PocketBase (SQLite + Realtime API)
- **UI Components**: shadcn-svelte
- **Styling**: Tailwind CSS
- **Package Manager**: PNPM

## Features

- 🚀 **Realtime Messaging**: Instant message delivery using PocketBase subscriptions.
- 🔐 **Authentication**: Secure email/password login and registration.
- 💅 **Modern UI**: Clean, responsive interface with dark mode support.
- 📜 **Auto-scroll**: Automatically scrolls to the latest message.
- 👤 **User Profiles**: Displays user names and avatars.

## Setup & Running

1.  **Install Dependencies**
    ```bash
    pnpm install
    ```

2.  **Environment Setup**
    Copy the example environment file and adjust if necessary.
    ```bash
    cp .env.example .env
    ```

3.  **Start PocketBase**

    Start the backend server.
    ```bash
    pnpm pocketbase
    # or directly:
    # ./pb/pocketbase serve
    ```
    The Admin UI is available at `http://127.0.0.1:8090/_/`.

3.  **Start the Application**
    Run the development server.
    ```bash
    pnpm dev
    ```
    Open `http://localhost:5173` in your browser.

## Database Schema

The app uses a `messages` collection in PocketBase:
- `text`: Text (Message content)
- `user`: Relation (Single, to `users` collection)

## License

MIT
