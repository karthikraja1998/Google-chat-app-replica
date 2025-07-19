# Technical Documentation: Google Chat to MERN App Integration

## 1. Introduction

This document outlines the technical architecture and workflow for a MERN (MongoDB, Express.js, React, Node.js) stack chat application that integrates bidirectionally with Google Chat. The application will facilitate 1:1 direct messaging, allowing users to converse with Google Chat contacts directly from the MERN app and vice-versa. The integration leverages OAuth 2.0 for user authentication and webhooks for real-time message reception from Google Chat.

## 2. Goals

- **Bidirectional 1:1 Messaging:** Enable seamless text-based conversations between Google Chat users and MERN app users.

- **User Provisioning:** Automatically create or identify users in the MERN app based on incoming Google Chat messages.

- **Real-time Updates:** Utilize WebSockets for instant message display in the MERN app.

- **Robust Error Handling:** Implement strategies for network issues, API rate limits, and token expiration.

- **Scalable Architecture:** Design a system capable of handling messages across different Google Workspace organizations.

## 3. Architecture Overview

The system comprises three primary components:

- **Google Chat Platform:** Source of incoming messages and destination for outgoing replies.

- **MERN Stack Application:** The core chat application, handling user data, conversation history, real-time messaging, and Google Chat API interactions.

- **Authentication Layer (OAuth 2.0):** Manages user consent and secure access tokens for Google Chat API.

### 3.1. Component Breakdown

- **Google Chat App (Configured in Google Cloud Console):**

  - Acts as the listener for messages in Google Chat.

  - Configured to receive 1:1 direct messages.

  - Has an Endpoint URL pointing directly to the MERN app's webhook.

  - Subscribed to "Message" events.

- **MERN Backend (Node.js/Express.js):**

  - **Webhook Listener:** A public endpoint to receive message events from Google Chat.

  - **Google Chat API Client:** Handles sending messages back to Google Chat using OAuth 2.0 tokens.

  - **User & Conversation Management:** Logic to create/update users and store conversation history in MongoDB.

  - **WebSocket Server (Socket.IO):** Pushes real-time message updates to the frontend.

  - **Authentication & Authorization:** Manages OAuth 2.0 flow, token storage, and user sessions.

- **MERN Frontend (React/Next.js):**

  - User interface for chat conversations.

  - Interacts with the MERN backend via REST APIs and WebSockets.

  - Manages user login via Google OAuth.

- **MongoDB:**

  - Stores user profiles (Google Chat ID/Email, name, avatar).

  - Stores chat conversation history.

  - Stores Google Chat OAuth tokens (securely encrypted).

## 4. Data Models

### 4.1. User Schema (MongoDB)

```json
{
  "_id": "ObjectId",
  "googleId": "String", // Unique Google Chat User ID (e.g., users/12345)
  "email": "String", // Primary identifier from Google Chat profile
  "name": "String",
  "avatarUrl": "String",
  "googleChatAccessToken": "String", // Encrypted OAuth token for sending messages
  "googleChatRefreshToken": "String", // Encrypted OAuth refresh token
  "createdAt": "Date",
  "updatedAt": "Date"
}
```

### 4.2. Conversation Schema (MongoDB)

```json
{
  "_id": "ObjectId",
  "participantOneId": "ObjectId", // Reference to User who initiated / local user
  "participantTwoId": "ObjectId", // Reference to the Google Chat User
  "googleChatSpaceId": "String", // Google Chat Space ID (for 1:1, unique per conversation)
  "createdAt": "Date",
  "updatedAt": "Date"
}
```

### 4.3. Message Schema (MongoDB)

```json
{
  "_id": "ObjectId",
  "conversationId": "ObjectId", // Reference to Conversation
  "senderId": "ObjectId", // Reference to User (either local user or the Google Chat user)
  "content": "String",
  "timestamp": "Date",
  "isFromGoogleChat": "Boolean", // True if message originated from Google Chat
  "status": "String" // e.g., "sent", "failed", "pending"
}
```

## 5. Workflow: Bidirectional Messaging

### 5.1. Initial Setup & User Onboarding

- **Google Cloud Project & Google Chat API:**

  - Ensure a Google Cloud Project is linked to a Google Workspace account.

  - Enable the Google Chat API.

- **Google Chat App Configuration (Google Cloud Console):**

  - Create a new Google Chat App.

  - **App Name & Avatar:** Set descriptive name and avatar.

  - **Features:** Enable "Receive 1:1 messages."

  - **Endpoint URL:** Configure this to your MERN backend's public webhook URL (e.g., `https://your-app.vercel.app/api/webhooks/google-chat`).

  - **Events:** Enable "Message" events.

- **OAuth Consent Screen (Google Cloud Console):**

  - Configure the OAuth consent screen with appropriate application details.

  - **Add Scopes:** For sending messages back to Google Chat, the following scopes are appropriate for text-only 1:1 messages:

    - `https://www.googleapis.com/auth/chat.messages`

    - `https://www.googleapis.com/auth/chat.spaces.messages` (This scope is for sending messages within spaces, which 1:1 chats are considered)

    - `https://www.googleapis.com/auth/userinfo.email` (To get user's email)

    - `https://www.googleapis.com/auth/userinfo.profile` (To get name and avatar)

  - **Authorized Redirect URIs:** Add your MERN app's OAuth callback URL (e.g., `https://your-app.vercel.app/api/auth/google/callback`).

- **User Login (MERN Frontend):**

  - User initiates login via "Login with Google" button.

  - Frontend redirects to Google OAuth 2.0 endpoint with configured scopes.

  - User grants consent.

  - Google redirects back to the MERN backend's OAuth callback.

- **OAuth Callback (MERN Backend):**

  - Receives authorization code from Google.

  - Exchanges code for `access_token` and `refresh_token` with Google.

  - Fetches user profile (email, name, avatar) using the `access_token` from Google's `userinfo` endpoint.

  - **Stores User:** Creates a new user in MongoDB or updates an existing one, storing the `googleId`, `email`, `name`, `avatarUrl`, and securely encrypting and storing `googleChatAccessToken` and `googleChatRefreshToken`.

  - Redirects user to the main chat interface in the frontend.

### 5.2. Receiving Messages from Google Chat

- **User Messages App in Google Chat:**

  - A Google Chat user (let's call them "GoogleUser") sends a 1:1 direct message to your configured Google Chat App.

- **Google Chat Webhook Trigger:**

  - Google Chat sends an HTTP POST request containing the message event payload to your MERN backend's `https://your-app.vercel.app/api/webhooks/google-chat` endpoint.

  - The payload will include:

    - `message.sender.name`: Contains `displayName` and `email` (preferable for mapping).

    - `message.text`: The message content.

    - `space.name`: The unique ID of the 1:1 space (e.g., `spaces/AAAA...`). This is crucial for replying.

    - `message.name`: The message ID.

    - `eventTime`: Timestamp of the message.

    - `type`: `MESSAGE` (or `google.workspace.chat.message.v1.created` if using the Workspace Events API, though for simplicity, the standard webhook sends a `MESSAGE` type).

- **MERN Backend Webhook Listener:**

  - Receives the POST request.

  - **Validate Webhook:** (Optional but recommended for production) Verify the `X-Goog-Signature` header to ensure the request genuinely came from Google.

  - **Extract Data:** Parses the JSON payload to get `sender.email`, `sender.displayName`, `sender.avatarUrl`, `message.text`, and `space.name` (this `space.name` will be the `googleChatSpaceId`).

  - **User Lookup/Creation:**

    - Queries MongoDB `User` collection using `sender.email`.

    - If user exists, retrieves their `_id`.

    - If user **does not** exist, a new `User` document is created using the `email`, `displayName`, and `avatarUrl` from the Google Chat payload.

  - **Conversation Lookup/Creation:**

    - Queries `Conversation` collection using the `googleChatSpaceId` (extracted from `space.name`) and the MERN app user's `_id` (the one who connected via OAuth).

    - If conversation exists, retrieves its `_id`.

    - If conversation **does not** exist, a new `Conversation` document is created, linking the MERN app user and the newly identified/created Google Chat user, and storing `googleChatSpaceId`.

  - **Store Message:** Creates a new `Message` document, associating it with the `conversationId`, marking `isFromGoogleChat: true`, and storing the `content`.

  - **Real-time Update (Socket.IO):** Emits a WebSocket event (e.g., `newMessage`) to the relevant MERN app user's frontend, containing the new message data.

  - **Respond to Webhook:** Sends an HTTP 200 OK response back to Google Chat to acknowledge receipt.

### 5.3. Sending Messages from MERN App to Google Chat

- **MERN Frontend User Replies:**

  - A MERN app user (let's call them "AppUser") types a reply in the chat interface and sends it.

  - Frontend sends a REST API request (e.g., `POST /api/messages`) to the MERN backend with `conversationId` and message `content`.

- **MERN Backend Processing:**

  - Receives the API request.

  - **Store Message:** Immediately creates a new `Message` document in MongoDB, marking `isFromGoogleChat: false`, and setting `status: "pending"`.

  - **Real-time Update (Socket.IO):** Emits a WebSocket event to the AppUser's frontend to display the pending message.

  - **Retrieve Tokens & Space ID:**

    - Fetches the AppUser's `googleChatAccessToken` and `googleChatRefreshToken` from the User document.

    - Fetches the `googleChatSpaceId` from the Conversation document.

  - **Token Refresh (if needed):**

    - Checks if the `access_token` is expired. If so, uses the `refresh_token` to get a new `access_token` from Google's OAuth endpoint. Updates the stored tokens in MongoDB.

    - **Invalid Token:** If refresh fails, updates the message status to "failed" in DB, emits WebSocket update, and the frontend should redirect AppUser to re-authenticate.

  - **Send Message to Google Chat API:**

    - Constructs a POST request to Google Chat API: `https://chat.googleapis.com/v1/spaces/{googleChatSpaceId}/messages`

    - **Headers:** `Authorization: Bearer <access_token>`, `Content-Type: application/json`.

    - **Body:** `{"text": "Your message content"}`.

    - **Retry Logic:** Implements retry attempts for network issues (e.g., exponential backoff).

  - **Handle API Response:**

    - If successful: Updates the Message document status to "sent" in MongoDB.

    - If failed (after retries): Updates the Message document status to "failed" in MongoDB. Emits a WebSocket event to the AppUser's frontend with an error message and a "retry" option.

  - **Prevent Duplicates from App's Own Message:** Crucially, when Google Chat triggers a webhook event for this very message (which it will, as it's a message in a 1:1 space), your webhook listener in Section 5.2 must ignore messages originating from your own Google Chat App's ID or a specific user ID associated with your app. This prevents the message from being re-added to your database. You can identify these by checking the `sender.name` or an `app.id` field in the webhook payload, or by checking if the `sender.email` matches your own app's service account email (if using service account for sending) or the logged-in user's email.

## 6. Real-time Updates (Socket.IO)

- **Server-side:**

  - Initialize `socket.io` with your Express.js app.

  - When a new message is received from Google Chat (in the webhook handler) or sent by the MERN app user, emit a `socket.emit('newMessage', messageData)` event to the relevant connected frontend clients (e.g., only to the user whose conversation is affected).

- **Client-side (React/Next.js):**

  - Connect to the Socket.IO server when the user logs in.

  - Listen for `socket.on('newMessage', (messageData) => { ... })` events.

  - When `newMessage` is received, update the React component state to display the new message instantly.

## 7. Error Handling and Retries

### 7.1. Network Issues / API Failures (Sending Messages)

- **Mechanism:** Implement a retry mechanism for HTTP requests to the Google Chat API (e.g., using a library like `axios-retry`).

- **Retry Strategy:** Use exponential backoff for retries to avoid overwhelming the API.

- **Persistent Failure:** If all retries fail, update the Message status in MongoDB to "failed."

- **Frontend Feedback:** The frontend receives a WebSocket update for the failed message, displaying a small error message and a "retry" button. Clicking this button triggers the send message API request again.

### 7.2. Invalid Token (Sending Messages)

- **Detection:** If the Google Chat API returns an HTTP 401 Unauthorized or specific error codes indicating an invalid/expired token, attempt to use the `refresh_token`.

- **Token Refresh:** If the `refresh_token` is valid, obtain a new `access_token` and update it in the user's MongoDB document. Then, re-attempt the message send.

- **Refresh Token Invalid:** If the `refresh_token` is also invalid or expired, this signifies that the user's Google Chat session is no longer authorized.

  - Update the message status to "failed."

  - Emit a WebSocket event to the frontend, indicating a session expiry.

  - Frontend redirects the user to the login page to re-authenticate via Google OAuth, thereby generating new tokens.

### 7.3. Google Chat API Rate Limits

- **Monitoring:** Monitor Google Chat API usage in the Google Cloud Console.

- **Best Practice:** Implement graceful error handling for HTTP 429 Too Many Requests responses from Google Chat. Instead of failing, the system should pause and retry after the `Retry-After` header duration. This can be integrated into the existing retry mechanism.

## 8. Development Environment & Deployment

### 8.1. Local Development

- **MongoDB:** Run a local MongoDB instance or use a cloud-hosted free tier (e.g., MongoDB Atlas).

- **Node.js/Express.js Backend:**

  - Configure environment variables for Google OAuth client ID, client secret, and callback URL.

  - **Exposing Local Webhook:** Use ngrok (or a similar tunneling service) to expose your local MERN backend's webhook endpoint (e.g., `http://localhost:3001/api/webhooks/google-chat`) to the internet. This public ngrok URL will be configured as the Endpoint URL in your Google Chat App settings in Google Cloud Console.

- **React/Next.js Frontend:** Run locally (e.g., `npm run dev`). Ensure it connects to your local Express.js backend.

- **Google Chat App Configuration:** The ngrok URL will need to be updated in the Google Cloud Console every time your ngrok tunnel changes (unless you use a paid, static ngrok domain).

- **JSON Schema Acquisition:** Your approach of using Pipedream's webhook action to capture an initial Google Chat event payload and inspect its JSON structure is excellent for local development and understanding the data.

### 8.2. Deployment Strategy (Vercel)

- **Next.js Unified Deployment:** Since you're using Next.js, Vercel is an ideal choice as it supports deploying both the frontend (React) and backend (Node.js API routes).

- **API Routes for Backend:** Your Express.js server logic (webhook listener, Google Chat API calls, database interactions, Socket.IO) will primarily reside within Next.js API routes (e.g., `pages/api/webhooks/google-chat.js`, `pages/api/messages.js`, `pages/api/auth/[...nextauth].js` if using NextAuth.js).

- **Environment Variables:** Store sensitive information (Google Client ID/Secret, MongoDB URI, Access Tokens) as environment variables in Vercel.

- **MongoDB Atlas:** Use a managed MongoDB service like MongoDB Atlas for your database in production.

- **Socket.IO on Vercel:** While Vercel supports WebSockets, ensure your Socket.IO setup is compatible with a serverless/edge environment. Often, socket.io needs to be adapted for serverless functions, or you might run it on a dedicated Node.js server and connect Next.js to it. For simpler real-time, Vercel's serverless functions might also support client-side polling or Server-Sent Events (SSE) if full WebSockets prove too complex for the assessment, but Socket.IO is definitely achievable.

## 9. Additional Considerations

### 9.1. Message History

- The MERN app will be the source of truth for conversation history. Messages received from Google Chat and messages sent from the MERN app will all be persistently stored in your MongoDB database.

- When a user views a conversation in the MERN app, it will fetch all relevant messages from its own database, ensuring a complete history independent of Google Chat's retention policies.

### 9.2. Notifications

- The MERN application will primarily focus on making the API call to post the reply message to Google Chat.

- Google Chat itself will handle its native notification mechanisms for the recipient Google Chat user, based on the message being delivered within its platform. Your MERN app doesn't need to implement separate Google Chat notification logic.
  </immersive>
