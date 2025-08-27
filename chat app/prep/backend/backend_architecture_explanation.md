# Backend Architecture - Complete File-by-File Explanation

## 🏗️ **Backend Architecture Overview**

The backend follows a **MVC (Model-View-Controller) pattern** with additional layers for middleware, routes, and real-time communication. Here's how each file works and their interactions:

## 📁 **File-by-File Breakdown**

### **1. `index.js` - Server Entry Point**
```javascript
// Main server file that orchestrates everything
import express from "express";
import dotenv from "dotenv"; 
import connectDB from "./config/database.js";
import userRoute from "./routes/userRoute.js";
import messageRoute from "./routes/messageRoute.js";
import cookieParser from "cookie-parser";
import cors from "cors";
import { app,server } from "./socket/socket.js";
```

**Purpose**: 
- **Server initialization** and configuration
- **Middleware setup** (CORS, cookie parser, JSON parsing)
- **Route registration** (user and message routes)
- **Database connection** initialization
- **Socket.io server** setup

**Flow**:
1. Loads environment variables
2. Sets up Express middleware
3. Registers API routes
4. Connects to MongoDB
5. Starts HTTP server with Socket.io

---

### **2. `config/database.js` - Database Connection**
```javascript
import mongoose from "mongoose";

const connectDB = async () => {
    await mongoose.connect(process.env.MONGO_URI).then(() => {
        console.log('Database connected');
    }).catch((error)=>{
        console.log(error);
    })
};
```

**Purpose**:
- **MongoDB connection** setup using Mongoose
- **Environment-based** database URI
- **Error handling** for connection failures

**Key Points**:
- Uses `process.env.MONGO_URI` for security
- Async/await pattern for connection
- Called from `index.js` when server starts

---

### **3. `models/` - Database Schemas**

#### **`userModel.js`**
```javascript
const userModel = new mongoose.Schema({
    fullName: { type: String, required: true },
    username: { type: String, required: true, unique: true },
    password: { type: String, required: true },
    profilePhoto: { type: String, default: "" },
    gender: { type: String, enum: ["male", "female"], required: true }
}, { timestamps: true });
```

**Purpose**:
- **User data structure** definition
- **Validation rules** (required fields, unique username)
- **Auto-generated** profile photos based on gender
- **Timestamps** for created/updated tracking

#### **`messageModel.js`**
```javascript
const messageModel = new mongoose.Schema({
    senderId: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
    receiverId: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
    message: { type: String, required: true }
}, { timestamps: true });
```

**Purpose**:
- **Message data structure** with sender/receiver references
- **User relationships** via ObjectId references
- **Message content** storage

#### **`conversationModel.js`**
```javascript
const conversationModel = new mongoose.Schema({
    participants: [{ type: mongoose.Schema.Types.ObjectId, ref: "User" }],
    messages: [{ type: mongoose.Schema.Types.ObjectId, ref: "Message" }]
}, { timestamps: true });
```

**Purpose**:
- **Conversation grouping** of messages between users
- **Participant tracking** (who's in the conversation)
- **Message history** organization

---

### **4. `middleware/isAuthenticated.js` - JWT Authentication**
```javascript
import jwt from "jsonwebtoken";

const isAuthenticated = async(req, res, next) => {
    try {
        const token = req.cookies.token;
        if(!token) {
            return res.status(401).json({message: "User not authenticated."})
        };
        const decode = await jwt.verify(token, process.env.JWT_SECRET_KEY);
        if(!decode) {
            return res.status(401).json({message: "Invalid token"});
        };
        req.id = decode.userId;
        next();
    } catch (error) {
        console.log(error);
    }
};
```

**Purpose**:
- **JWT token validation** from HTTP-only cookies
- **User authentication** for protected routes
- **Request enhancement** with user ID
- **Security middleware** for API protection

**Flow**:
1. Extracts JWT token from cookies
2. Verifies token using secret key
3. Decodes user ID and adds to request object
4. Allows/denies access to protected routes

---

### **5. `routes/` - API Endpoint Definitions**

#### **`userRoute.js`**
```javascript
import express from "express";
import { getOtherUsers, login, logout, register } from "../controllers/userController.js";
import isAuthenticated from "../middleware/isAuthenticated.js";

const router = express.Router();

router.route("/register").post(register);
router.route("/login").post(login);
router.route("/logout").get(logout);
router.route("/").get(isAuthenticated, getOtherUsers);
```

**Purpose**:
- **User-related API endpoints** definition
- **HTTP method mapping** (POST for register/login, GET for logout/users)
- **Middleware integration** (authentication for protected routes)

**Endpoints**:
- `POST /register` - User registration (public)
- `POST /login` - User authentication (public)
- `GET /logout` - User logout (public)
- `GET /` - Get all users (protected)

#### **`messageRoute.js`**
```javascript
import express from "express";
import { getMessage, sendMessage } from "../controllers/messageController.js";
import isAuthenticated from "../middleware/isAuthenticated.js";

const router = express.Router();

router.route("/send/:id").post(isAuthenticated, sendMessage);
router.route("/:id").get(isAuthenticated, getMessage);
```

**Purpose**:
- **Message-related API endpoints** definition
- **Protected routes** (all require authentication)
- **Parameter handling** (receiver ID in URL)

**Endpoints**:
- `POST /send/:id` - Send message to user (protected)
- `GET /:id` - Get conversation with user (protected)

---

### **6. `controllers/` - Business Logic**

#### **`userController.js`**
Contains four main functions:

**`register`**:
```javascript
export const register = async (req, res) => {
    // 1. Validate input data
    // 2. Check if username exists
    // 3. Hash password with bcrypt
    // 4. Generate profile photo URL
    // 5. Create user in database
    // 6. Return success response
};
```

**`login`**:
```javascript
export const login = async (req, res) => {
    // 1. Validate credentials
    // 2. Find user by username
    // 3. Compare password with bcrypt
    // 4. Generate JWT token
    // 5. Set HTTP-only cookie
    // 6. Return user data
};
```

**`logout`**:
```javascript
export const logout = (req, res) => {
    // 1. Clear JWT cookie
    // 2. Return success message
};
```

**`getOtherUsers`**:
```javascript
export const getOtherUsers = async (req, res) => {
    // 1. Get logged-in user ID from middleware
    // 2. Find all users except current user
    // 3. Exclude password field
    // 4. Return user list
};
```

#### **`messageController.js`**
Contains two main functions:

**`sendMessage`**:
```javascript
export const sendMessage = async (req, res) => {
    // 1. Get sender ID from middleware
    // 2. Get receiver ID from URL params
    // 3. Find or create conversation
    // 4. Create new message
    // 5. Update conversation with message
    // 6. Emit real-time message via Socket.io
    // 7. Return new message
};
```

**`getMessage`**:
```javascript
export const getMessage = async (req, res) => {
    // 1. Get user IDs from request
    // 2. Find conversation between users
    // 3. Populate messages with full data
    // 4. Return message history
};
```

---

### **7. `socket/socket.js` - Real-time Communication**
```javascript
import {Server} from "socket.io";
import http from "http";
import express from "express";

const app = express();
const server = http.createServer(app);
const io = new Server(server, {
    cors: {
        origin: ['http://localhost:5173'],
        methods: ['GET', 'POST'],
    },
});

const userSocketMap = {}; // {userId -> socketId}

io.on('connection', (socket) => {
    const userId = socket.handshake.query.userId
    if(userId !== undefined) {
        userSocketMap[userId] = socket.id;
    } 
    io.emit('getOnlineUsers', Object.keys(userSocketMap));

    socket.on('disconnect', () => {
        delete userSocketMap[userId];
        io.emit('getOnlineUsers', Object.keys(userSocketMap));
    })
});
```

**Purpose**:
- **WebSocket server** setup with Socket.io
- **User-socket mapping** for real-time messaging
- **Online status tracking** and broadcasting
- **CORS configuration** for frontend connection

**Key Functions**:
- `getReceiverSocketId()`: Maps user ID to socket ID
- Connection handling: Maps users to sockets
- Disconnection handling: Removes users from online list
- Real-time broadcasting: Sends online users to all clients

---

## 🔄 **Complete Request Flow**

### **User Registration Flow**:
```
1. Frontend → POST /api/v1/user/register
2. userRoute.js → userController.js → register()
3. register() → userModel.js → Database
4. Response → Frontend
```

### **User Login Flow**:
```
1. Frontend → POST /api/v1/user/login
2. userRoute.js → userController.js → login()
3. login() → userModel.js → Database
4. JWT token → HTTP-only cookie
5. User data → Frontend
6. Frontend → Socket.io connection with user ID
```

### **Message Sending Flow**:
```
1. Frontend → POST /api/v1/message/send/:receiverId
2. messageRoute.js → isAuthenticated middleware
3. middleware → messageController.js → sendMessage()
4. sendMessage() → conversationModel.js → messageModel.js → Database
5. Socket.io → Emit "newMessage" to receiver
6. Response → Frontend
```

### **Real-time Message Reception**:
```
1. Receiver's socket → Listens for "newMessage"
2. Frontend → Updates Redux state
3. UI → Re-renders with new message
```

## 🎯 **Key Interview Points**

### **Architecture Benefits**:
- **Separation of Concerns**: Each file has a specific responsibility
- **Modularity**: Easy to maintain and extend
- **Security**: JWT authentication, password hashing
- **Scalability**: Clean structure for adding features

### **Technical Highlights**:
- **MVC Pattern**: Models, Views (API responses), Controllers
- **Middleware Pattern**: Authentication, CORS, parsing
- **Real-time Communication**: Socket.io integration
- **Database Relationships**: User-Message-Conversation models
- **Security Best Practices**: JWT, bcrypt, HTTP-only cookies

### **Common Interview Questions**:
1. **"How does authentication work?"** → JWT tokens in HTTP-only cookies
2. **"How do you handle real-time messaging?"** → Socket.io with user-socket mapping
3. **"What's the database structure?"** → Three models with relationships
4. **"How do you ensure security?"** → Middleware, password hashing, CORS
5. **"How would you scale this?"** → Load balancers, Redis for sessions, message queues

This architecture demonstrates modern Node.js/Express.js best practices with real-time capabilities!
