# DevConnect Project Structure - Visual Guide

## 📁 Project Structure Overview

```
DevConnect/
├── src/
│   ├── app.js                    ← 🚀 ENTRY POINT (Start Here!)
│   ├── config/
│   │   └── database.js           ← 🔌 Database Connection
│   ├── models/
│   │   ├── user.js               ← 👤 User Data Structure
│   │   └── connectionRequest.js  ← 🤝 Connection Data Structure
│   ├── middlewares/
│   │   └── auth.js               ← 🔐 Authentication Check
│   ├── routes/
│   │   ├── auth.js               ← 🔑 Login/Signup Routes
│   │   ├── profile.js            ← 👤 Profile Management
│   │   ├── request.js            ← 🤝 Connection Requests
│   │   └── user.js               ← 👥 User Discovery
│   └── utils/
│       └── validation.js         ← ✅ Input Validation
└── package.json
```

## 🔄 How Everything Connects

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Frontend      │    │   Backend       │    │   Database      │
│   (Port 5173)   │    │   (Port 3000)   │    │   (MongoDB)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │ 1. HTTP Request       │                       │
         ├──────────────────────►│                       │
         │                       │                       │
         │                       │ 2. Middleware         │
         │                       │ (CORS, Auth, etc.)    │
         │                       │                       │
         │                       │ 3. Route Handler      │
         │                       │ (auth.js, user.js)    │
         │                       │                       │
         │                       │ 4. Model Operations   │
         │                       │ (user.js, connection) │
         │                       │                       │
         │                       │ 5. Database Query     │
         │                       ├──────────────────────►│
         │                       │                       │
         │                       │ 6. Database Response  │
         │                       │◄──────────────────────┤
         │                       │                       │
         │ 7. JSON Response      │                       │
         │◄──────────────────────┤                       │
         │                       │                       │
```

## 🎯 Step-by-Step Learning Path

### Phase 1: Foundation (Files 1-3)
```
1. app.js           ← Server setup, middleware, routes
2. database.js      ← Database connection
3. user.js          ← User data model
```

### Phase 2: Core Features (Files 4-6)
```
4. connectionRequest.js  ← Connection data model
5. validation.js         ← Input validation
6. auth.js (middleware)  ← Authentication
```

### Phase 3: API Endpoints (Files 7-10)
```
7. auth.js (routes)      ← Login/signup APIs
8. profile.js            ← Profile APIs
9. request.js            ← Connection APIs
10. user.js (routes)     ← User discovery APIs
```

## 🔍 What Each File Does (Simple Terms)

### `app.js` - The Boss
- **What it does**: Sets up the server, connects all parts
- **Think of it as**: The main control center
- **Key parts**: CORS, cookies, routes, database connection

### `database.js` - The Bridge
- **What it does**: Connects to MongoDB database
- **Think of it as**: The bridge between app and database
- **Key parts**: Connection string, error handling

### `user.js` (model) - User Blueprint
- **What it does**: Defines what user data looks like
- **Think of it as**: A template for user information
- **Key parts**: Name, email, password, profile fields

### `connectionRequest.js` - Connection Blueprint
- **What it does**: Defines how users connect
- **Think of it as**: A template for connection requests
- **Key parts**: From user, to user, status

### `validation.js` - The Guard
- **What it does**: Checks if data is correct
- **Think of it as**: A security guard checking IDs
- **Key parts**: Email validation, password strength

### `auth.js` (middleware) - The Bouncer
- **What it does**: Checks if user is logged in
- **Think of it as**: A bouncer at a club
- **Key parts**: JWT verification, user lookup

### `auth.js` (routes) - Login Desk
- **What it does**: Handles signup and login
- **Think of it as**: The reception desk
- **Key parts**: User creation, password hashing, JWT creation

### `profile.js` - Profile Manager
- **What it does**: Handles profile viewing and editing
- **Think of it as**: A profile editor
- **Key parts**: View profile, update profile

### `request.js` - Connection Manager
- **What it does**: Handles connection requests
- **Think of it as**: A matchmaker
- **Key parts**: Send requests, accept/reject requests

### `user.js` (routes) - User Discovery
- **What it does**: Finds users and shows connections
- **Think of it as**: A user directory
- **Key parts**: Feed, connections, requests

## 🚀 Quick Start Guide

### 1. First, understand the server setup
Read `app.js` - see how Express server is created and configured

### 2. Then, understand data storage
Read `database.js` and `models/` - see how data is structured

### 3. Next, understand security
Read `middlewares/auth.js` and `utils/validation.js` - see how data is protected

### 4. Finally, understand the APIs
Read `routes/` - see how different features work

## 💡 Key Concepts to Remember

### Data Flow
1. **Request comes in** → `app.js`
2. **Middleware processes** → CORS, auth, etc.
3. **Route handles** → Specific API logic
4. **Model operates** → Database operations
5. **Response sent back** → JSON data

### Authentication Flow
1. **User logs in** → `auth.js` route
2. **JWT created** → Stored in cookie
3. **Protected route accessed** → `auth.js` middleware checks JWT
4. **User data attached** → Available in route handler

### Connection Flow
1. **User sends request** → `request.js` route
2. **Request saved** → `connectionRequest.js` model
3. **Other user reviews** → Accept/reject
4. **Status updated** → Database updated

This structure helps you understand how each piece fits together!

