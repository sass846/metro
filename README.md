# Metro Route Finder (NextStop)

A metro route planning system that computes optimal paths between stations using graph algorithms, with performance-optimized search and autocomplete.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
  - [High-level flow](#high-level-flow)
  - [Core Components](#core-components)
- [API Endpoints](#api-endpoints)
- [Tech Stack](#tech-stack)
- [Design Tradeoffs](#design-tradeoffs)
- [Possible Improvements](#possible-improvements)
- [How to Run](#how-to-run)
- [Notes](#notes)

---

## Overview

This project implements a metro route planning system that supports **multiple metro networks** (e.g., Delhi, Mumbai, Chennai).  
Each metro network is modeled as a weighted graph and loaded into memory **on demand** to optimize routing performance.
It then uses **Dijkstra’s algorithm** to compute optimal routes.

To avoid repeated database queries on the critical path, the system caches:
- metro graphs (adjacency lists)
- station name indices (tries)

using an **LRU-based eviction strategy**, ensuring only frequently accessed metro systems remain in memory.

---

## Features

- Shortest-path computation using **Dijkstra’s algorithm**
- Supports multiple metro systems with on-demand loading
- Prefix-based station autocomplete using a **Trie**
- In-memory caching of active metro graphs and station indices
- LRU eviction to control memory usage
- REST APIs consumed by a lightweight frontend

---

## Architecture

### High-level flow

1. Client sends a route query specifying:
   - metro system
   - source station
   - destination station
2. Backend checks the in-memory cache for the requested metro system
3. If cache hit:
   - Perform route computation directly in memory
4. If cache miss:
   - Load metro data from the database
   - Build adjacency list and station Trie
   - Insert into cache and evict least recently used metro if needed
5. Compute route and return response

---

## Core Components

### Graph Model
- Each metro system is stored persistently in PostgreSQL
- Metro graphs are loaded into memory only when requested
- Graphs are represented as adjacency lists for efficient traversal
- An LRU cache evicts inactive metro systems to limit memory usage

---

### Routing Engine
- Uses **Dijkstra’s algorithm** for shortest-path computation
- Early termination once the destination is reached
- Runs entirely in memory for cached metro systems
- Route computation does not require database queries

---

### Autocomplete Engine
- Station names are loaded per metro system
- A **Trie** is constructed and kept in memory
- Enables fast prefix-based station lookup
- Trie instances are evicted together with their corresponding metro graphs

---

### Caching Strategy

- Metro graphs and station indices are cached using an **LRU policy**
- Recently accessed metro systems remain resident in memory
- Less popular metro systems are evicted automatically
- Prevents database queries during route computation for active metros
- Allows the system to scale to many metro networks without linear memory growth

---

## API Endpoints

### List available metro systems

Retrieves a list of all supported metro networks.

- **Endpoint:** `/api/v1/metros`
- **Method:** `GET`

**Response:**
```json
[
  {
    "id": 1,
    "name": "Delhi Metro"
  },
  {
    "id": 2,
    "name": "Mumbai Metro"
  }
]
```

---

### 2. Station Autocomplete
Returns station name suggestions for a specific metro system based on a search prefix.

- **Endpoint:** `/api/v1/metros/:metroId/stations`
- **Method:** `GET`
- **Query Parameters:**
  - `prefix` (string): The search term (e.g., `sta`).

**Example Request:**
`GET /api/v1/metros/1/stations?prefix=sta`

**Response:**
```json
[
  "Station A",
  "Station Central",
  "Station Square"
]
```

---

### 3. Get Fastest Route
Computes the shortest path and distance between two stations within a specific metro system.

- **Endpoint:** `/api/v1/metros/:metroId/route`
- **Method:** `POST`

**Request Body:**
```json
{
  "startStation": "Station A",
  "endStation": "Station B"
}
```

**Response:**
```json
{
  "path": [
    "Station A",
    "Station C",
    "Station D",
    "Station B"
  ],
  "distance": 18
}
```

https://metro-rf.vercel.app/

## Tech Stack

- **Backend:** Node.js, Express
- **Database:** PostgreSQL
- **ORM:** Prisma
- **Algorithms:** Dijkstra’s Algorithm, Trie, LRU Cache
- **APIs:** REST
- **Frontend:** React

---

## Design Tradeoffs

### On-demand loading vs. Preloading all metros
Metro graphs are loaded only when requested. This reduces memory usage and startup time while ensuring that recently accessed systems remain fast via caching.

### In-memory graphs vs. Database queries
Keeping graphs and station indices in memory avoids repeated database round-trips during route computation, trading off higher RAM usage for significantly faster response times.

### Dijkstra vs. A*
Dijkstra was chosen for correctness and simplicity. Since reliable heuristic data (e.g., exact geographical coordinates for every station) was not consistently available, A* could not be effectively implemented.

---

## Possible Improvements

- [ ] Persist cached metro graphs using **Redis** for multi-instance deployments.
- [ ] Add transfer penalties between metro lines.
- [ ] Support time-based routing (peak vs. non-peak).
- [ ] Improve observability with metrics and logging.
- [ ] Add path visualization on the frontend.

---

## How to Run

```bash
git clone [https://github.com/sass846/metro.git](https://github.com/sass846/metro.git)
cd backend
npm install
npm run dev
```

### Environment Setup

Create a `.env` file in the root directory and add the following variables.

> **Note:** This project uses Supabase. You need two connection strings: one for the connection pooler (Transaction Mode) and one for direct connections (migrations).

```bash
JWT_SECRET=your_super_secret_key
PORT=8000

# Connect to Supabase via connection pooling (Transaction Mode)
# Use the pooler port (usually 6543) and append ?pgbouncer=true
DATABASE_URL="postgresql://postgres.[project-ref]:[password]@[host]:6543/postgres?pgbouncer=true&connection_limit=1"

# Direct connection to the database. Used for migrations
# Use the direct port (usually 5432)
DIRECT_URL="postgresql://postgres:[password]@[host]:5432/postgres?sslmode=require"
```
---

## Notes

This project was built to explore graph algorithms, backend performance optimization, and scalable system design, with an emphasis on serving routing queries efficiently.
