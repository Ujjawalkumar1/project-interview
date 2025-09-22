# 🧠 CodeByte Backend - Complete Project Documentation

## Backend Architecture and Flows

### Overview
- **Stack**: Express.js server, MongoDB with Mongoose, Redis for token blacklist, Judge0 CE API (via RapidAPI) for code execution.
- **Entry point**: `src/index.js` wires middleware, routes, DB init, and server startup.
- **Main modules**:
  - `routes/` expose HTTP endpoints
  - `controllers/` implement business logic
  - `middleware/` handles auth (user/admin) and blacklist check
  - `models/` define MongoDB schemas
  - `utils/problemUtility.js` wraps Judge0 submit/poll logic

### Server bootstrap (`src/index.js`)
- Loads env, connects to MongoDB via `src/config/db.js`, sets CORS (localhost:5173) with credentials, JSON body parsing, cookie parsing, and mounts routers:
  - `/user` → `routes/userAuth.js`
  - `/problem` → `routes/problemCreator.js`
  - `/submission` → `routes/submit.js`

```12:26:src/index.js
app.use(cors({
    origin: 'http://localhost:5173',
    credentials: true 
}))

app.use(express.json());
app.use(cookieParser());

app.use('/user',authRouter);
app.use('/problem',problemRouter);
app.use('/submission',submitRouter);
```

DB is connected, then the server listens on `process.env.PORT`.

### Database (`src/config/db.js`)
Connects using `process.env.DB_CONNECT_STRING` with Mongoose `mongoose.connect(...)`.

### Redis (`src/config/redis.js`)
Creates a Redis client pointed at localhost:6379. It is used for token blacklist checks. Ensure the client is connected before using in production.

### Models
- `models/user.js`
  - Fields: `firstName`, `lastName`, `emailId` (unique, immutable), `age`, `role` (`user|admin`), `problemSolved` (array of `problem` ids), `password` (hashed).
  - Post hook: on user deletion (`findOneAndDelete`), cascade delete all `submission` documents for that user.

- `models/problem.js`
  - Problem definition with `title`, `description`, `difficulty`, `tags`, `visibleTestCases`, `hiddenTestCases`, `startCode`, `referenceSolution`, and `problemCreator`.

- `models/submission.js`
  - Stores each submission with `userId`, `problemId`, `code`, `language (javascript|c++|java)`, `status (pending|accepted|wrong|error)`, `runtime`, `memory`, `errorMessage`, `testCasesPassed`, `testCasesTotal`.

### Authentication and Authorization

#### Login/Register/Logout (`controllers/userAuthent.js`, `routes/userAuth.js`)
- `POST /user/register`:
  - Validates input (`utils/validator.js`), hashes password (bcrypt), creates user, issues a JWT with payload `{ _id, emailId, role: 'user' }`, `expiresIn: 3600s`.
  - Sets cookie `token` (maxAge 1h) and returns basic user info.

- `POST /user/login`:
  - Verifies credentials, issues JWT with user role, sets cookie `token`, returns user info.

- `POST /user/logout` (protected):
  - Reads `token` from cookies, `jwt.decode(token)` to get `exp`.
  - Blacklists token in Redis with key `token:<rawToken>` and sets TTL to token expiration using `expireAt`.
  - Clears cookie and responds success.

```81:99:src/controllers/userAuthent.js
const {token} = req.cookies;
const payload = jwt.decode(token);
await redisClient.set(`token:${token}`,'Blocked');
await redisClient.expireAt(`token:${token}`,payload.exp);
res.cookie("token",null,{expires: new Date(Date.now())});
```

#### Middleware: User vs Admin
- `middleware/userMiddleware.js`:
  - Verifies JWT from cookie using `process.env.JWT_KEY`, loads user from DB, rejects if token key `token:<rawToken>` exists in Redis, then attaches `req.result = user`.

```9:41:src/middleware/userMiddleware.js
const {token} = req.cookies;
const payload = jwt.verify(token,process.env.JWT_KEY);
const result = await User.findById(payload._id);
const IsBlocked = await redisClient.exists(`token:${token}`);
if(IsBlocked) throw new Error("Invalid Token");
req.result = result;
next();
```

- `middleware/adminMiddleware.js`:
  - Same as user middleware, but also requires `payload.role === 'admin'`.

```21:35:src/middleware/adminMiddleware.js
if(payload.role!='admin')
    throw new Error("Invalid Token");
const IsBlocked = await redisClient.exists(`token:${token}`);
if(IsBlocked) throw new Error("Invalid Token");
```

### Problem Management (`controllers/userProblem.js`, `routes/problemCreator.js`)
- Admin-only endpoints:
  - `POST /problem/create` creates a problem after verifying the provided `referenceSolution` passes all `visibleTestCases` on Judge0 for each language.
  - `PUT /problem/update/:id` validates updated `referenceSolution` similarly before persisting changes.
  - `DELETE /problem/delete/:id` deletes a problem.

- User endpoints:
  - `GET /problem/problemById/:id` returns problem details (without hidden tests).
  - `GET /problem/getAllProblem` lists basic problem cards.
  - `GET /problem/problemSolvedByUser` returns problems solved by the authenticated user.
  - `GET /problem/submittedProblem/:pid` returns submissions for a problem by the user.

Judge0 validation (admin create/update):
```25:53:src/controllers/userProblem.js
const languageId = getLanguageById(language);
const submissions = visibleTestCases.map(tc => ({
  source_code: completeCode,
  language_id: languageId,
  stdin: tc.input,
  expected_output: tc.output
}));
const submitResult = await submitBatch(submissions);
const resultToken = submitResult.map(v => v.token);
const testResult = await submitToken(resultToken);
for(const test of testResult){ if(test.status_id!=3) return res.status(400).send("Error Occured"); }
```

### Submissions and Execution (`controllers/userSubmission.js`, `routes/submit.js`)
- `POST /submission/run/:id` (user): Runs user code against `visibleTestCases` only and returns per-test results plus aggregate runtime/memory.
- `POST /submission/submit/:id` (user): Persists a `submission` document, runs code against `hiddenTestCases`, aggregates results, updates submission with final status, and updates the user’s `problemSolved` list if accepted.

```41:56:src/controllers/userSubmission.js
const languageId = getLanguageById(language);
const submissions = problem.hiddenTestCases.map(tc=>({
  source_code: code,
  language_id: languageId,
  stdin: tc.input,
  expected_output: tc.output
}));
const submitResult = await submitBatch(submissions);
const resultToken = submitResult.map(v => v.token);
const testResult = await submitToken(resultToken);
```

Aggregation of Judge0 results to compute status, runtime and memory, then saving the submission and updating user solved problems if newly accepted.

### Judge0 Integration Details (`utils/problemUtility.js`)
- Language mapping:
```4:14:src/utils/problemUtility.js
{"c++":54,"java":62,"javascript":63}
```
- Submit batch of testcases:
```20:35:src/utils/problemUtility.js
POST https://judge0-ce.p.rapidapi.com/submissions/batch
headers include x-rapidapi-key from env JUDGE0_KEY
payload: { submissions: [...] }
```
- Poll results until all `status_id > 2` then return `submissions` array:
```60:95:src/utils/problemUtility.js
GET .../submissions/batch?tokens=...&fields=*
while(true){
  const result = await fetchData();
  const done = result.submissions.every(r=>r.status_id>2);
  if(done) return result.submissions;
  await waiting(1000);
}
```

## Token Blacklisting: How it works

### Steps
1. **User logs out**: server reads the JWT from cookies and decodes it to get expiry `exp`.
2. **Blacklist the token** in Redis using key `token:<rawToken>` and set TTL to the token’s natural expiration:
   - `SET token:<token> "Blocked"`
   - `EXPIREAT token:<token> <exp>`
3. **Clear the cookie** so the browser stops sending the token.
4. **On each authenticated request**, middleware checks if the Redis key exists; if yes, requests are rejected.

### Code
Blacklist during logout:
```81:99:src/controllers/userAuthent.js
await redisClient.set(`token:${token}`,'Blocked');
await redisClient.expireAt(`token:${token}`,payload.exp);
```
Reject blacklisted tokens in middleware:
```29:36:src/middleware/userMiddleware.js
const IsBlocked = await redisClient.exists(`token:${token}`);
if(IsBlocked) throw new Error("Invalid Token");
```

Notes:
- Redis client must be connected (`await redisClient.connect()`) in production. In `src/index.js` the connect is commented out; enable it when Redis is available.
- Blacklisting uses the raw token string as part of the key; ensure cookie transport is secure in production (Secure, SameSite settings).

## Judge0 Flow: End-to-end Steps

### For Admin (Problem creation/update validation)
1. For each `referenceSolution` and each `visibleTestCases`:
   - Map language → Judge0 `language_id`.
   - Build `submissions` array `{ source_code, language_id, stdin, expected_output }`.
   - Call `submitBatch(submissions)`.
   - Extract tokens and poll `submitToken(tokens)` until all are finished.
   - If any `status_id != 3` (Accepted), abort with `400`.

### For User (Run/Submit)
1. Run (`/submission/run/:id`):
   - Build submissions from `visibleTestCases` only and get per-test results.
2. Submit (`/submission/submit/:id`):
   - Insert a `submission` with status `pending` and `testCasesTotal`.
   - Build submissions from `hiddenTestCases` and execute via Judge0.
   - Aggregate `status`, `testCasesPassed`, `runtime` (sum of times), `memory` (max), `errorMessage` if present.
   - Save `submission`; if accepted and problem not already in `user.problemSolved`, push and save user.

## Environment Variables
- `PORT`: Express server port.
- `DB_CONNECT_STRING`: MongoDB connection string.
- `JWT_KEY`: JWT signing secret.
- `JUDGE0_KEY`: RapidAPI key for Judge0 CE.
- Redis defaults to localhost:6379 (configure as needed).
----------------------------------------------------------------------------------------------------------------------










This is the backend for **CodeByte**, a full-featured coding practice platform inspired by LeetCode — built entirely from scratch using **Node.js**, **Express.js**, **MongoDB**, **Redis**, and integrated with **Judge0** for real-time code execution.

---

## 🚀 Features

### 🔐 Authentication
- Secure **JWT-based** login and signup for users and admins
- Middleware for protected routes and role-based access (admin/user)

### 📘 Problem Management
- Full **CRUD operations** for coding problems
- Each problem includes metadata: title, description, difficulty, tags, examples, test cases, etc.
- Admin-only access to problem creation and editing

### 💻 Online Code Execution (Judge0)
- Integrated with **Judge0 API** to support real-time code compilation and execution in multiple languages (C++, Java, Python, etc.)
- Supports custom input and auto-evaluation using test cases
- Displays output, status, and error messages


### 💬 Submissions & History
- Track user submissions per problem
- Store success/fail status, runtime, language used
- Filter past attempts, and retry solutions

### ⚙️ Other Highlights
- **Redis caching** for faster performance on frequent API calls
- Organized **MVC architecture** for clean separation of concerns
- **Rate limiting**, **error handling**, and **input validation**
- Scalable structure for future additions like contests or discussion forums

---

## 🧪 Tech Stack

| Layer         | Technology            |
|---------------|------------------------|
| Runtime       | Node.js                |
| Framework     | Express.js             |
| Database      | MongoDB (Mongoose ORM) |
| Caching       | Redis                  |
| Auth          | JWT                    |
| Code Runner   | Judge0 API             |
| Architecture  | MVC                    |

---

## 📁 Project Structure & File Organization

### **1. Root Configuration**
- **`package.json`** - Project dependencies and scripts
- **`README.md`** - Project documentation and features overview

### **2. Entry Point**
- **`src/index.js`** - Main server file that initializes Express app, connects to databases, and sets up all routes

---

## 🔧 Configuration Layer (`src/config/`)

### **`db.js`**
- **Purpose**: MongoDB connection setup using Mongoose
- **Function**: Connects to MongoDB using environment variable `DB_CONNECT_STRING`
- **Key Features**: 
  - Async connection function
  - Environment-based configuration
  - Error handling for connection failures

### **`redis.js`**
- **Purpose**: Redis client configuration for caching and token blacklisting
- **Function**: Creates Redis connection to Redis Cloud instance for session management
- **Configuration**: 
  - Host: Redis Cloud instance
  - Authentication via environment variables
  - Used for token blacklisting and caching

---

## 🗄️ Data Models (`src/models/`)

### **`user.js`**
- **Purpose**: User schema with authentication and profile data
- **Key Fields**: 
  - `firstName`: String (required, 3-20 chars)
  - `lastName`: String (3-20 chars, optional)
  - `emailId`: String (required, unique, immutable, lowercase)
  - `age`: Number (6-80 range, optional)
  - `role`: String (enum: 'user', 'admin', default: 'user')
  - `problemSolved`: Array of ObjectIds (references to problems)
  - `password`: String (required, hashed)
- **Special Features**: 
  - Password hashing with bcrypt
  - Role-based access control
  - Tracks solved problems
  - Cascade deletion of submissions when user is deleted
  - Timestamps for creation and updates

### **`problem.js`**
- **Purpose**: Coding problem schema with comprehensive metadata
- **Key Fields**: 
  - `title`: String (required)
  - `description`: String (required)
  - `difficulty`: String (enum: 'easy', 'medium', 'hard', required)
  - `tags`: String (enum: 'array', 'linkedList', 'graph', 'dp', required)
  - `visibleTestCases`: Array of test cases with explanations
  - `hiddenTestCases`: Array of test cases for evaluation
  - `startCode`: Array of starter code in different languages
  - `referenceSolution`: Array of complete solutions in different languages
  - `problemCreator`: ObjectId (reference to user, required)
- **Test Case Structure**:
  - `input`: String (required)
  - `output`: String (required)
  - `explanation`: String (for visible test cases only)
- **Code Structure**:
  - `language`: String (required)
  - `initialCode`/`completeCode`: String (required)

### **`submission.js`**
- **Purpose**: User code submission tracking and evaluation results
- **Key Fields**: 
  - `userId`: ObjectId (reference to user, required)
  - `problemId`: ObjectId (reference to problem, required)
  - `code`: String (required)
  - `language`: String (enum: 'javascript', 'c++', 'java', required)
  - `status`: String (enum: 'pending', 'accepted', 'wrong', 'error', default: 'pending')
  - `runtime`: Number (milliseconds, default: 0)
  - `memory`: Number (kilobytes, default: 0)
  - `errorMessage`: String (default: '')
  - `testCasesPassed`: Number (default: 0)
  - `testCasesTotal`: Number (default: 0)
- **Special Features**:
  - Indexed on userId and problemId for performance
  - Timestamps for tracking submission history
  - Comprehensive performance metrics


---

## 🎮 Business Logic Controllers (`src/controllers/`)

### **`userAuthent.js`**
#### **`register`**
- **Purpose**: User registration with password hashing and JWT token generation
- **Process**: 
  - Validates input data
  - Hashes password using bcrypt (salt rounds: 10)
  - Creates user with 'user' role
  - Generates JWT token (1 hour expiry)
  - Sets HTTP-only cookie
  - Returns user data (excluding password)

#### **`login`**
- **Purpose**: User authentication with bcrypt password comparison
- **Process**: 
  - Validates email and password
  - Finds user by email
  - Compares password hash
  - Generates JWT token
  - Sets cookie and returns user data

#### **`logout`**
- **Purpose**: Token blacklisting in Redis for security
- **Process**: 
  - Extracts token from cookies
  - Decodes JWT payload
  - Adds token to Redis blacklist
  - Sets expiration time
  - Clears cookie

#### **`adminRegister`**
- **Purpose**: Admin user creation (admin-only)
- **Process**: 
  - Requires admin middleware
  - Creates user with admin role
  - Returns success message

#### **`deleteProfile`**
- **Purpose**: User account deletion with cascade cleanup
- **Process**: 
  - Deletes user document
  - Cascade deletion handled by Mongoose hooks
  - Returns success message

### **`userProblem.js`**
#### **`createProblem`**
- **Purpose**: Admin creates new coding problems with test case validation
- **Process**: 
  - Validates reference solution against visible test cases
  - Uses Judge0 API to test code execution
  - Creates problem in database
  - Links to problem creator

#### **`updateProblem`**
- **Purpose**: Admin modifies existing problems
- **Process**: 
  - Validates problem ID
  - Tests updated reference solution
  - Updates problem document
  - Returns updated problem

#### **`deleteProblem`**
- **Purpose**: Admin removes problems
- **Process**: 
  - Validates problem ID
  - Deletes problem document
  - Returns success message

#### **`getProblemById`**
- **Purpose**: Fetches problem details
- **Process**: 
  - Retrieves problem data
- **Returns**: Problem data

#### **`getAllProblem`**
- **Purpose**: Lists all problems (title, difficulty, tags only)
- **Returns**: Array of problem summaries

#### **`solvedAllProblembyUser`**
- **Purpose**: Shows user's solved problems
- **Process**: 
  - Populates problemSolved array
  - Returns problem details

#### **`submittedProblem`**
- **Purpose**: Displays user's submission history for a problem
- **Returns**: Array of submission objects

### **`userSubmission.js`**
#### **`submitCode`**
- **Purpose**: Processes code submissions through Judge0 API
- **Process**: 
  - Creates submission record
  - Submits code to Judge0 for hidden test cases
  - Processes results and updates submission
  - Marks problem as solved if accepted
- **Returns**: Submission results with performance metrics

#### **`runCode`**
- **Purpose**: Tests code against visible test cases only
- **Process**: 
  - Submits code to Judge0 for visible test cases
  - Returns execution results without storing submission
- **Use Case**: Code testing before final submission


---

## 🔒 Security Middleware (`src/middleware/`)

### **`userMiddleware.js`**
- **Purpose**: JWT token verification for protected routes
- **Process**: 
  - Extracts token from cookies
  - Verifies JWT signature
  - Checks Redis blacklist
  - Validates user existence
  - Attaches user data to `req.result`
- **Security Features**: 
  - HTTP-only cookies
  - Token blacklisting
  - User validation

### **`adminMiddleware.js`**
- **Purpose**: Admin-only route protection
- **Process**: 
  - Extends userMiddleware functionality
  - Verifies admin role
  - Same security features as user middleware
- **Use Case**: Admin-only operations (problem CRUD)

---

## 🛣️ API Routes (`src/routes/`)

### **`userAuth.js`**
- **`POST /user/register`** - User registration
- **`POST /user/login`** - User login
- **`POST /user/logout`** - User logout (protected)
- **`POST /user/admin/register`** - Admin registration (admin-only)
- **`DELETE /user/deleteProfile`** - Profile deletion (protected)
- **`GET /user/check`** - Token validation check (protected)

### **`problemCreator.js`**
- **`POST /problem/create`** - Create problem (admin-only)
- **`PUT /problem/update/:id`** - Update problem (admin-only)
- **`DELETE /problem/delete/:id`** - Delete problem (admin-only)
- **`GET /problem/problemById/:id`** - Get problem details (protected)
- **`GET /problem/getAllProblem`** - List all problems (protected)
- **`GET /problem/problemSolvedByUser`** - User's solved problems (protected)
- **`GET /problem/submittedProblem/:pid`** - Submission history (protected)

### **`submit.js`**
- **`POST /submission/submit/:id`** - Submit code for evaluation (protected)
- **`POST /submission/run/:id`** - Test code against visible cases (protected)


---

## 🛠️ Utility Functions (`src/utils/`)**

### **`problemUtility.js`**
#### **`getLanguageById`**
- **Purpose**: Maps programming languages to Judge0 language IDs
- **Supported Languages**:
  - C++: 54
  - Java: 62
  - JavaScript: 63

#### **`submitBatch`**
- **Purpose**: Submits multiple code submissions to Judge0 API
- **Process**: 
  - Creates batch submission request
  - Uses RapidAPI Judge0 service
  - Returns submission tokens

#### **`submitToken`**
- **Purpose**: Polls Judge0 for execution results
- **Process**: 
  - Polls until all submissions complete
  - Returns execution results
  - Handles async code execution

### **`validator.js`**
#### **`validate`**
- **Purpose**: Input validation for user registration
- **Validation Rules**: 
  - Required fields: firstName, emailId, password
  - Email format validation
  - Strong password requirements
- **Dependencies**: validator.js library

---

## 🔄 Data Flow & Architecture

### **1. Authentication Flow**
```
User Login → JWT Token → Cookie Storage → Middleware Verification → Route Access
```

### **2. Problem Solving Flow**
```
User Views Problem → Writes Code → Submits → Judge0 Execution → Result Storage → Problem Marked Solved
```


---

## 🚀 Key Features & Capabilities

### **🔐 Security**
- JWT-based authentication with Redis blacklisting
- Role-based access control (admin/user)
- Password hashing with bcrypt
- Protected API endpoints
- HTTP-only cookies for token storage

### **💻 Code Execution**
- Real-time code compilation and execution
- Support for C++, Java, JavaScript
- Hidden and visible test case validation
- Performance metrics (runtime, memory)
- Batch submission processing


### **🗄️ Data Management**
- MongoDB with Mongoose ODM
- Redis caching for performance
- Comprehensive user and problem tracking
- Cascade deletion and data integrity

---

## 🎯 Use Cases & Scenarios

### **For Admins:**
- Create and manage coding problems
- Monitor user submissions
- Maintain problem quality

### **For Users:**
- Solve coding problems
- Track learning progress
- Submit and test code

### **For the System:**
- Automatic code evaluation
- Performance tracking
- User progress monitoring
- Content management

---

## 🔧 Environment Variables Required

```env
# Database
DB_CONNECT_STRING=mongodb://your-mongodb-connection-string

# Redis
REDIS_PASS=your-redis-password

# JWT
JWT_KEY=your-jwt-secret-key

# Judge0 API
JUDGE0_KEY=your-rapidapi-judge0-key


# Server
PORT=3000
```

---

## 🚀 Getting Started

### **Installation**
```bash
npm install
```

### **Development**
```bash
npm run dev
```

### **Production**
```bash
node src/index.js
```

---

## 📊 Database Schema Relationships

```
User (1) ←→ (Many) Problem (as creator)
User (1) ←→ (Many) Submission
User (1) ←→ (Many) SolutionVideo
Problem (1) ←→ (Many) Submission
Problem (1) ←→ (Many) SolutionVideo
```

---

## 🔮 Future Enhancements

- **Contest System**: Timed coding competitions
- **Discussion Forum**: Problem-specific discussions
- **Leaderboards**: User ranking systems
- **Code Templates**: Language-specific starter code
- **Performance Analytics**: Detailed user progress tracking
- **Mobile API**: Mobile app support
- **WebSocket**: Real-time updates and notifications

---

## 🏆 Project Highlights

This backend serves as a **complete coding practice platform** that demonstrates:

- **Modern Architecture**: Clean MVC pattern with proper separation of concerns
- **Security Best Practices**: JWT authentication, password hashing, role-based access
- **External Integrations**: Judge0 for code execution
- **Performance Optimization**: Redis caching, database indexing
- **Scalability**: Modular design for easy feature additions
- **Production Ready**: Comprehensive error handling, validation, and logging

The architecture is **scalable, secure, and production-ready** with proper separation of concerns, error handling, and modern development practices suitable for enterprise-level applications.

