# Jam App 🎵

A collaborative music listening platform inspired by Spotify Jams — built as a learning project with a professional-grade architecture. Users can create persistent rooms, invite friends, and queue YouTube videos that play in real time for everyone in the session.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [API Reference](#api-reference)
- [WebSocket Events](#websocket-events)
- [Data Model](#data-model)
- [Design Decisions](#design-decisions)

---

## Overview

Jam App solves a real problem: existing collaborative listening tools (like Spotify Jams) require all users to have a premium subscription, and none of them support YouTube. Jam App centralizes the concept for YouTube, allowing anyone to join a room and add songs to a shared queue — no premium account required.

The project was built with learning in mind, progressively evolving from a simple Supabase-powered frontend to a full NestJS backend with JWT authentication, PostgreSQL persistence, and Socket.io real-time communication.

---

## Features

- **User accounts** — register and login with JWT-based authentication, sessions persist via localStorage
- **Persistent rooms** — rooms are saved to the database with an admin and member list
- **Role-based access** — each member has a `can_play` flag controlling playback permissions
- **YouTube search** — search videos directly from the app (API key stays on the server)
- **Shared queue** — add songs to a room's queue, visible to all participants in real time
- **Real-time sync** — queue updates, participant joins/leaves propagated instantly via WebSockets
- **YouTube player** — the room admin controls playback (play, pause, skip); the player only renders for the admin
- **Automatic host transfer** — if the admin disconnects, the role is transferred to the next connected user

---

## Architecture

The application is split into two independent projects:

```
Jam-App/
├── BackEnd/       # NestJS REST API + WebSocket Gateway
└── FrontEnd/      # React + Vite SPA
```

### Communication pattern

- **REST (HTTP)** — used for stateless operations: auth, room/user/song CRUD, YouTube search
- **WebSockets (Socket.io)** — used for real-time session state: queue updates, participant presence, playback control

The frontend talks to the backend through two channels simultaneously:
1. HTTP requests via a centralized `api.js` service layer
2. A persistent Socket.io connection managed through `socket.js`

### Request lifecycle (adding a song)

```
User clicks "Add"
  → socket.emit('addSong', { roomId, title, video_id, ... })
  → Gateway receives event
  → Calls SongsService.create() → persists to PostgreSQL
  → server.to(roomId).emit('queueUpdated')
  → All clients in the room fetch updated queue via GET /songs/room/:roomId
```

---

## Tech Stack

### Backend

| Technology | Version | Purpose |
|---|---|---|
| Node.js | 24.11.1 | Runtime |
| TypeScript | ^5.7.3 | Language |
| NestJS | ^11.0.1 | Framework |
| TypeORM | ^0.3.28 | ORM |
| PostgreSQL | latest (Docker) | Database |
| Socket.io | ^4.8.3 | WebSocket transport |
| @nestjs/websockets | ^11.1.17 | WebSocket gateway |
| @nestjs/jwt | ^11.0.2 | JWT generation and validation |
| @nestjs/passport | ^11.0.5 | Authentication strategies |
| passport-jwt | ^4.0.1 | JWT strategy |
| bcrypt | ^6.0.0 | Password hashing |
| class-validator | ^0.14.4 | DTO validation |
| class-transformer | ^0.5.1 | Response serialization |
| nanoid | ^3.3.11 | Room ID generation |
| Docker + docker-compose | latest | Database container |

### Frontend

| Technology | Version | Purpose |
|---|---|---|
| React | ^19.2.4 | UI framework |
| Vite | ^8.0.0 | Build tool |
| React Router DOM | ^7.13.1 | Client-side routing |
| Tailwind CSS | ^4.2.2 | Utility-first styling |
| socket.io-client | ^4.8.3 | WebSocket client |
| react-youtube | ^10.1.0 | YouTube IFrame API wrapper |

### External APIs

| API | Purpose |
|---|---|
| YouTube Data API v3 | Video search |

---

## Project Structure

### Backend (`/BackEnd/src`)

```
src/
├── auth/                  # JWT auth — register, login, strategy, guard
├── users/                 # User entity and CRUD
├── rooms/                 # Room entity, RoomMember, CRUD, member management
├── songs/                 # Song entity, queue endpoints
├── youtube/               # YouTube search proxy
├── jam-session/           # WebSocket gateway
│   └── jam-session/
│       └── jam-session.gateway.ts
└── common/
    └── interfaces/        # Shared TypeScript interfaces (AuthenticatedRequest)
```

### Frontend (`/FrontEnd/src`)

```
src/
├── pages/
│   ├── Auth.jsx           # Login and register
│   ├── Home.jsx           # Create or join a room
│   └── Room.jsx           # Main room view
├── components/
│   ├── ParticipantsList.jsx
│   ├── Queue.jsx
│   └── SearchResults.jsx
└── services/
    ├── api.js             # Centralized HTTP service layer
    └── socket.js          # Socket.io client instance
```

---

## Getting Started

### Prerequisites

- Node.js >= 18
- Docker and docker-compose
- A YouTube Data API v3 key ([get one here](https://console.cloud.google.com))

### 1. Clone the repository

```bash
git clone https://github.com/mateowagner/Jam-App
cd Jam-App
```

### 2. Set up the backend

```bash
cd BackEnd
npm install
```

Create a `.env` file in `/BackEnd`:

```env
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=your_password
DB_DATABASE=jam_db
JWT_SECRET=your_jwt_secret_here
YOUTUBE_API_KEY=your_youtube_api_key_here
```

Start the PostgreSQL database:

```bash
docker-compose up -d
```

Start the backend:

```bash
npm run start:dev
```

The API will be available at `http://localhost:3000`.

### 3. Set up the frontend

```bash
cd ../FrontEnd
npm install
npm run dev
```

The frontend will be available at `http://localhost:5173`.

---

## API Reference

### Auth

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| POST | `/auth/register` | Register a new user | No |
| POST | `/auth/login` | Login and receive JWT | No |

### Rooms

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| POST | `/rooms` | Create a room | Yes |
| GET | `/rooms` | List all rooms | Yes |
| GET | `/rooms/:id` | Get room by ID | Yes |
| PATCH | `/rooms/:id` | Update room | Yes |
| DELETE | `/rooms/:id` | Delete room | Yes |
| POST | `/rooms/:id/members` | Add member to room | Yes |

### Songs

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| POST | `/songs` | Add a song to a room | Yes |
| GET | `/songs/room/:roomId` | Get active queue for a room | Yes |
| PATCH | `/songs/:id/played` | Mark a song as played | Yes |

### YouTube

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| GET | `/youtube/search?q=query` | Search YouTube videos | Yes |

---

## WebSocket Events

### Client → Server

| Event | Payload | Description |
|---|---|---|
| `joinRoom` | `{ roomId, username }` | Join a room session |
| `leaveRoom` | `{ roomId, username }` | Leave a room session |
| `addSong` | `{ roomId, title, video_id, thumbnail, added_by_id }` | Add a song to the queue |
| `playSong` | `{ roomId, songId }` | Notify playback started |
| `pauseSong` | `{ roomId }` | Notify playback paused |
| `skipSong` | `{ roomId, songId }` | Skip current song |

### Server → Client

| Event | Payload | Description |
|---|---|---|
| `roomParticipants` | `[{ username }]` | Full participant list on join |
| `userJoined` | `{ username }` | A new user joined the room |
| `userLeft` | `{ username }` | A user left the room |
| `queueUpdated` | `{ song? / skippedSongId? }` | Queue changed — fetch updated queue |
| `songPlaying` | `{ songId }` | Playback started |
| `songPaused` | — | Playback paused |

---

## Data Model

```
users
  id (uuid, PK)
  email (unique)
  username (unique)
  password (hashed)
  created_at

rooms
  id (nanoid, PK)
  name
  admin_id (FK → users)
  created_at

room_members
  id (uuid, PK)
  room_id (FK → rooms, CASCADE)
  user_id (FK → users)
  can_play (boolean)
  joined_at

songs
  id (uuid, PK)
  room_id (FK → rooms, CASCADE)
  video_id
  title
  thumbnail
  added_by_id (FK → users)
  played (boolean, default false)
  added_at
```

---

## Design Decisions

**Why NestJS over Express?** NestJS enforces a modular, opinionated structure that scales well. Its decorator-based approach (controllers, services, guards, gateways) maps directly to the separation of concerns principles applied throughout the project.

**Why PostgreSQL over an in-memory store?** Rooms, users, and song history are persistent data. Even though the active session state (who's connected) lives in memory on the WebSocket gateway, the underlying data survives server restarts.

**Why Socket.io over native WebSockets?** Socket.io adds room management, automatic reconnection, and fallback transports out of the box — all of which are needed for a multi-room real-time app.

**Why YouTube over Spotify?** The YouTube IFrame API is free and works without user authentication. Spotify's Web Playback SDK requires all users to have a Premium account, which defeats the purpose of a collaborative tool.

**REST + WebSockets hybrid:** Operations that mutate data (add song, skip song) go through the WebSocket gateway, which persists to the database and then broadcasts to the room. Read operations (fetch queue, fetch room) go through REST. This keeps the gateway focused on real-time coordination and REST focused on data access.

---

## Author

Mateo Wagner — built as a learning project to practice full-stack development with NestJS, TypeORM, Socket.io, and React.
