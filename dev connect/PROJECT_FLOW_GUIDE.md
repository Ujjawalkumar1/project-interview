# DevConnect Backend - Project Understanding Flow

## 🎯 How to Understand This Project Step by Step

### Step 1: Start with the Entry Point (`src/app.js`)
**Why start here?** This is where everything begins - the server setup and configuration.

**What to look for:**
- Server creation with Express
- Middleware setup (CORS, JSON parsing, cookies)
- Route connections
- Database connection
- Server startup

**Key concepts:**
- `express()` creates the server
- `app.use()` adds middleware (functions that run before routes)
- CORS allows frontend to talk to backend
- Routes are connected with `app.use("/", router)`

---

### Step 2: Database Connection (`src/config/database.js`)
**Why this next?** The app needs to connect to MongoDB before starting.

**What to look for:**
- Mongoose connection setup
- Environment variable usage (`process.env.DB_CONNECTION_SECRET`)
- Error handling

**Key concepts:**
- `mongoose.connect()` connects to MongoDB
- Environment variables keep secrets safe
- `async/await` for database operations

---

### Step 3: Data Models (Start with `src/models/user.js`)
**Why models first?** They define what data looks like and how it behaves.

**What to look for:**
- Schema definition (what fields a user has)
- Field types and validation rules
- Custom methods (`getJWT`, `validatePassword`)
- Timestamps

**Key concepts:**
- `mongoose.Schema` defines data structure
- `required`, `unique`, `enum` are validation rules
- `methods` add functions to the model
- `timestamps` automatically adds `createdAt` and `updatedAt`

---

### Step 4: Connection Model (`src/models/connectionRequest.js`)
**Why this next?** It defines how users connect with each other.

**What to look for:**
- References to User model (`ref: "User"`)
- Status enum (ignored, interested, accepted, rejected)
- Index for performance
- Pre-save hook (prevents self-requests)

**Key concepts:**
- `ObjectId` references link to other documents
- `enum` restricts values to specific options
- `index()` speeds up queries
- `pre("save")` runs before saving

---

### Step 5: Validation Utilities (`src/utils/validation.js`)
**Why utilities next?** They ensure data is correct before processing.

**What to look for:**
- Input validation functions
- Email and password validation
- Allowed field checking for profile updates

**Key concepts:**
- `validator` library checks email format and password strength
- `Object.keys()` gets field names from request body
- `every()` checks if all fields are allowed

---

### Step 6: Authentication Middleware (`src/middlewares/auth.js`)
**Why middleware next?** It protects routes that need login.

**What to look for:**
- JWT token extraction from cookies
- Token verification
- User lookup and attachment to request

**Key concepts:**
- `req.cookies` gets cookies from request
- `jwt.verify()` checks if token is valid
- `req.user` attaches user data for route handlers
- `next()` continues to the actual route

---

### Step 7: Route Files (Start with `src/routes/auth.js`)
**Why routes next?** They handle the actual API endpoints.

**What to look for:**
- Route definitions (`POST /signup`, `POST /login`, etc.)
- Request body processing
- Database operations
- Response sending

**Key concepts:**
- `express.Router()` creates route groups
- `req.body` contains data from frontend
- `bcrypt.hash()` encrypts passwords
- `res.cookie()` sets cookies
- `res.json()` sends JSON response

---

### Step 8: Profile Routes (`src/routes/profile.js`)
**Why this next?** It handles user profile operations.

**What to look for:**
- Protected routes (uses `userAuth` middleware)
- Profile viewing and editing
- Validation before updates

**Key concepts:**
- `userAuth` middleware ensures user is logged in
- `req.user` contains logged-in user data
- `Object.keys().forEach()` updates multiple fields
- `save()` persists changes to database

---

### Step 9: Request Routes (`src/routes/request.js`)
**Why this next?** It handles connection requests between users.

**What to look for:**
- Sending connection requests
- Reviewing/rejecting requests
- Status management
- Duplicate prevention

**Key concepts:**
- URL parameters (`req.params`) get data from URL
- `$or` queries check multiple conditions
- Status transitions (interested → accepted/rejected)
- Error handling for edge cases

---

### Step 10: User Routes (`src/routes/user.js`)
**Why this last?** It handles user discovery and connections.

**What to look for:**
- Feed generation (finding new users)
- Connection listing
- Request management
- Pagination

**Key concepts:**
- `populate()` fetches related user data
- `$nin` (not in) excludes users
- Pagination with `skip()` and `limit()`
- `Set` for efficient duplicate removal

---

## 🔄 How Data Flows Through the System

### 1. User Registration Flow
```
Frontend → POST /signup → auth.js → validation.js → user.js model → database
```

### 2. User Login Flow
```
Frontend → POST /login → auth.js → user.js model → JWT creation → cookie set
```

### 3. Protected Route Flow
```
Frontend → Any protected route → auth.js middleware → route handler → response
```

### 4. Connection Request Flow
```
Frontend → POST /request/send → request.js → connectionRequest.js model → database
```

### 5. Feed Generation Flow
```
Frontend → GET /feed → user.js → complex query → populate users → response
```

---

## 🗂️ Folder Structure Explanation

### `src/` - Main source code
- **`app.js`** - Server entry point and configuration
- **`config/`** - Configuration files (database connection)
- **`models/`** - Database schemas and models
- **`routes/`** - API endpoint handlers
- **`middlewares/`** - Functions that run before routes
- **`utils/`** - Helper functions (validation, etc.)

---

## 🎯 Key Learning Points for Each File

### `app.js`
- How Express server is set up
- Middleware order matters
- CORS configuration
- Route organization

### `config/database.js`
- Environment variables usage
- Async database connection
- Error handling patterns

### `models/user.js`
- Schema definition
- Field validation
- Custom methods
- Password hashing

### `models/connectionRequest.js`
- References between models
- Indexes for performance
- Pre-save hooks
- Status management

### `utils/validation.js`
- Input sanitization
- Field validation
- Security practices

### `middlewares/auth.js`
- JWT token handling
- Cookie processing
- User authentication flow

### `routes/auth.js`
- User registration/login
- Password handling
- Cookie management
- Error responses

### `routes/profile.js`
- Protected routes
- Data updates
- Field validation

### `routes/request.js`
- Connection management
- Status transitions
- Duplicate prevention

### `routes/user.js`
- Complex queries
- Data population
- Pagination
- Feed algorithms

---

## 🚀 How to Practice Understanding

1. **Start with `app.js`** - Understand server setup
2. **Follow a user registration** - Trace from frontend to database
3. **Follow a login** - See how JWT is created and stored
4. **Follow a protected route** - See how middleware works
5. **Follow a connection request** - See how models interact
6. **Follow feed generation** - See complex queries and pagination

---

## 💡 Interview Tips

When explaining this project:
1. **Start with the problem** - "It's a networking app for developers"
2. **Explain the tech stack** - "Node.js, Express, MongoDB, Mongoose"
3. **Show the flow** - "User signs up → JWT created → can access protected routes"
4. **Highlight key features** - "Connection requests, profile management, user discovery"
5. **Mention improvements** - "Better validation, rate limiting, security headers"

This flow will help you understand how everything connects and works together!

