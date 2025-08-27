# DevConnect Project - Detailed Folder & File Explanation

## 📁 Project Structure Overview

```
DevConnect/
├── src/                          ← Main source code folder
│   ├── app.js                    ← Server entry point
│   ├── config/                   ← Configuration files
│   │   └── database.js           ← Database connection setup
│   ├── models/                   ← Database schemas (data templates)
│   │   ├── user.js               ← User data structure
│   │   └── connectionRequest.js  ← Connection data structure
│   ├── middlewares/              ← Functions that run before routes
│   │   └── auth.js               ← Authentication checker
│   ├── routes/                   ← API endpoints (URL handlers)
│   │   ├── auth.js               ← Login/signup endpoints
│   │   ├── profile.js            ← Profile management endpoints
│   │   ├── request.js            ← Connection request endpoints
│   │   └── user.js               ← User discovery endpoints
│   └── utils/                    ← Helper functions
│       └── validation.js         ← Input validation functions
├── package.json                  ← Project dependencies and scripts
└── package-lock.json             ← Exact versions of dependencies
```

---

## 🚀 1. `src/` - Main Source Code Folder

**What is it?** This is where all your application code lives. Think of it as the "brain" of your app.

**Why is it important?** It contains all the logic that makes your app work - from handling requests to storing data.

---

## 📄 2. `src/app.js` - The Main Server File

**What does it do?** This is the "boss" of your entire application. It starts the server and connects everything together.

**Think of it as:** The main control center of a building - it turns on all the lights, connects all the rooms, and makes sure everything works together.

### 🔧 What's Inside `app.js`:

```javascript
// 1. Import necessary packages
const express = require("express");
const connectDB = require("./config/database");
const cors = require("cors");
const cookieParser = require("cookie-parser");

// 2. Create the server
const app = express();

// 3. Set up middleware (functions that run before routes)
app.use(cors({
  origin: "http://localhost:5173",  // Allow frontend to connect
  credentials: true                 // Allow cookies to be sent
}));

app.use(express.json());            // Parse JSON data from requests
app.use(cookieParser());            // Parse cookies from requests

// 4. Connect all the route files
const authRouter = require("./routes/auth");
const profileRouter = require("./routes/profile");
const requestRouter = require("./routes/request");
const userRouter = require("./routes/user");

app.use("/", authRouter);           // All auth routes start with "/"
app.use("/", profileRouter);        // All profile routes start with "/"
app.use("/", requestRouter);        // All request routes start with "/"
app.use("/", userRouter);           // All user routes start with "/"

// 5. Connect to database and start server
connectDB()
  .then(() => {
    console.log("Database connected successfully");
    app.listen(3000, () => {
      console.log("Server is running on port 3000");
    });
  })
  .catch((err) => {
    console.error("Database connection failed:", err);
  });
```

### 🎯 Key Points:
- **Server Creation**: `express()` creates a web server
- **Middleware Setup**: CORS allows frontend connection, JSON parser handles data, cookie parser handles cookies
- **Route Connection**: All route files are connected to the main app
- **Database Connection**: Connects to MongoDB before starting the server
- **Server Start**: Listens on port 3000 for incoming requests

---

## ⚙️ 3. `src/config/` - Configuration Folder

**What does it do?** Contains settings and configuration files that your app needs to work properly.

**Think of it as:** The settings panel of your phone - where you configure how things should work.

---

### 📄 `src/config/database.js` - Database Connection

**What does it do?** Connects your app to the MongoDB database.

**Think of it as:** A bridge that connects your app to the data storage warehouse.

```javascript
const mongoose = require("mongoose");

const connectDB = async () => {
  try {
    // Connect to MongoDB using the connection string from environment variables
    await mongoose.connect(process.env.DB_CONNECTION_SECRET);
    console.log("✅ Database connected successfully");
  } catch (err) {
    console.error("❌ Database connection failed. Error:", err);
  }
};

module.exports = connectDB;
```

### 🎯 Key Points:
- **Environment Variables**: Uses `process.env.DB_CONNECTION_SECRET` to keep database password safe
- **Async/Await**: Uses modern JavaScript to handle database connection
- **Error Handling**: Catches and logs any connection errors
- **Module Export**: Makes this function available to other files

---

## 🗃️ 4. `src/models/` - Database Models Folder

**What does it do?** Contains "blueprints" that define how data should be stored in the database.

**Think of it as:** Templates or forms that define what information can be stored and how it should look.

---

### 📄 `src/models/user.js` - User Data Blueprint

**What does it do?** Defines what information a user can have and how it should be stored.

**Think of it as:** A user profile form that defines what fields are required and what type of data each field should contain.

```javascript
const mongoose = require("mongoose");
const validator = require("validator");
const jwt = require("jsonwebtoken");
const bcrypt = require("bcrypt");

// Define the user data structure
const userSchema = new mongoose.Schema({
  firstName: {
    type: String,
    required: true,           // Must be provided
    minLength: 3,            // At least 3 characters
    maxLength: 20,           // Maximum 20 characters
  },
  lastName: {
    type: String,
  },
  emailId: {
    type: String,
    required: true,
    unique: true,            // No two users can have same email
    index: true,             // Makes email searches faster
    lowercase: true,         // Always store in lowercase
    trim: true,              // Remove extra spaces
  },
  password: {
    type: String,
    required: true,
  },
  age: {
    type: Number,
    min: 18,                 // Must be 18 or older
  },
  gender: {
    type: String,
    trim: true,
    enum: {
      values: ["male", "female", "other"],  // Only these values allowed
      message: `{VALUE} is not a valid gender type`,
    },
  },
  photoUrl: {
    type: String,
    default: "https://geographyandyou.com/images/user-profile.png",  // Default profile picture
  },
  about: {
    type: String,
    default: "This is for all",  // Default about text
  },
  skills: {
    type: [String],          // Array of strings (multiple skills)
  },
}, {
  timestamps: true,          // Automatically adds createdAt and updatedAt
});

// Custom method to create JWT token
userSchema.methods.getJWT = async function () {
  const user = this;
  const token = await jwt.sign(
    { _id: user._id },           // Token contains user ID
    process.env.JWT_SECRET,      // Secret key from environment
    { expiresIn: "7d" }          // Token expires in 7 days
  );
  return token;
};

// Custom method to check if password is correct
userSchema.methods.validatePassword = async function (passwordInputByUser) {
  const user = this;
  const passwordHash = user.password;
  
  // Compare input password with stored hash
  const isPasswordValid = await bcrypt.compare(
    passwordInputByUser,
    passwordHash
  );
  return isPasswordValid;
};

const userModel = mongoose.model("User", userSchema);
module.exports = userModel;
```

### 🎯 Key Points:
- **Schema Definition**: Defines what fields a user can have
- **Validation Rules**: `required`, `unique`, `min`, `max`, `enum` ensure data quality
- **Custom Methods**: `getJWT()` creates login tokens, `validatePassword()` checks passwords
- **Timestamps**: Automatically tracks when user was created and last updated
- **Default Values**: Provides default values for optional fields

---

### 📄 `src/models/connectionRequest.js` - Connection Data Blueprint

**What does it do?** Defines how connection requests between users are stored.

**Think of it as:** A friendship request form that tracks who sent the request, who received it, and what the status is.

```javascript
const mongoose = require("mongoose");

const connectionRequestSchema = new mongoose.Schema({
  fromUserId: {
    type: mongoose.Schema.Types.ObjectId,  // Reference to User model
    ref: "User",                          // Links to User collection
    required: true,
  },
  toUserId: {
    type: mongoose.Schema.Types.ObjectId,  // Reference to User model
    ref: "User",                          // Links to User collection
    required: true,
  },
  status: {
    type: String,
    required: true,
    enum: {
      values: ["ignored", "interested", "accepted", "rejected"],  // Only these statuses allowed
      message: `{VALUE} is incorrect status type`,
    },
  },
}, { timestamps: true });

// Create an index for faster queries
connectionRequestSchema.index({ fromUserId: 1, toUserId: 1 });

// Prevent users from sending requests to themselves
connectionRequestSchema.pre("save", function (next) {
  const connectionRequest = this;
  if (connectionRequest.fromUserId.equals(connectionRequest.toUserId)) {
    throw new Error("Cannot send connection request to yourself!");
  }
  next();
});

const ConnectionRequestModel = new mongoose.model(
  "ConnectionRequest",
  connectionRequestSchema
);
module.exports = ConnectionRequestModel;
```

### 🎯 Key Points:
- **References**: `fromUserId` and `toUserId` link to actual User documents
- **Status Management**: Tracks the state of each connection request
- **Index**: Makes queries faster when searching by user IDs
- **Pre-save Hook**: Prevents users from sending requests to themselves
- **Validation**: Ensures only valid status values are stored

---

## 🔐 5. `src/middlewares/` - Middleware Functions Folder

**What does it do?** Contains functions that run before your main route handlers to check or process requests.

**Think of it as:** Security guards that check everyone before they enter a building.

---

### 📄 `src/middlewares/auth.js` - Authentication Checker

**What does it do?** Checks if a user is logged in before allowing them to access protected routes.

**Think of it as:** A bouncer at a club who checks your ID before letting you in.

```javascript
const User = require("../models/user");
const jwt = require("jsonwebtoken");

const userAuth = async (req, res, next) => {
  try {
    // 1. Get the token from cookies
    const { token } = req.cookies;

    // 2. Check if token exists
    if (!token) {
      return res.status(401).send("Please login!");
    }

    // 3. Verify the token is valid
    const decodedToken = await jwt.verify(token, process.env.JWT_SECRET);
    const { _id } = decodedToken;

    // 4. Find the user in database
    const user = await User.findById(_id);
    if (!user) {
      throw new Error("User not found");
    }

    // 5. Attach user data to request for use in route handler
    req.user = user;
    next();  // Continue to the actual route handler
  } catch (err) {
    res.status(400).send("Error: " + err.message);
  }
};

module.exports = {
  userAuth,
};
```

### 🎯 Key Points:
- **Token Extraction**: Gets JWT token from cookies
- **Token Verification**: Uses secret key to verify token is valid
- **User Lookup**: Finds the actual user in database
- **Request Attachment**: Adds user data to `req.user` for route handlers
- **Error Handling**: Returns error if authentication fails

---

## 🛣️ 6. `src/routes/` - API Routes Folder

**What does it do?** Contains all the API endpoints that handle different types of requests.

**Think of it as:** Different departments in a company - each handles specific types of work.

---

### 📄 `src/routes/auth.js` - Authentication Routes

**What does it do?** Handles user registration, login, and logout.

**Think of it as:** The reception desk where people sign up, check in, and check out.

```javascript
const express = require("express");
const authRouter = express.Router();
const { validateSignUpData } = require("../utils/validation");
const User = require("../models/user");
const bcrypt = require("bcrypt");

// POST /signup - Create new user account
authRouter.post("/signup", async (req, res) => {
  try {
    // 1. Validate the input data
    validateSignUpData(req);

    const { firstName, lastName, emailId, password } = req.body;

    // 2. Hash the password for security
    const passwordHash = await bcrypt.hash(password, 10);

    // 3. Create new user
    const user = new User({
      firstName,
      lastName,
      emailId,
      password: passwordHash,
    });

    // 4. Save user to database
    const savedUser = await user.save();
    
    // 5. Create JWT token
    const token = await savedUser.getJWT();

    // 6. Set token in cookie
    res.cookie("token", token, {
      expires: new Date(Date.now() + 8 * 3600000),  // 8 hours
    });

    // 7. Send success response
    res.json({ message: "User Added successfully!", data: savedUser });
  } catch (err) {
    res.status(400).send("ERROR : " + err.message);
  }
});

// POST /login - User login
authRouter.post("/login", async (req, res) => {
  try {
    const { emailId, password } = req.body;

    // 1. Find user by email
    const user = await User.findOne({ emailId: emailId });
    if (!user) {
      throw new Error("Invalid credentials");
    }

    // 2. Check if password is correct
    const isPasswordValid = await user.validatePassword(password);

    if (isPasswordValid) {
      // 3. Create JWT token
      const token = await user.getJWT();

      // 4. Set token in cookie
      res.cookie("token", token, {
        expires: new Date(Date.now() + 8 * 3600000),  // 8 hours
      });

      // 5. Send user data
      res.send(user);
    } else {
      throw new Error("Invalid credentials");
    }
  } catch (err) {
    res.status(400).send("ERROR : " + err.message);
  }
});

// POST /logout - User logout
authRouter.post("/logout", async (req, res) => {
  // Clear the token cookie
  res.cookie("token", null, {
    expires: new Date(Date.now()),
  });
  res.send("Logout Successful!!");
});

module.exports = authRouter;
```

### 🎯 Key Points:
- **Signup Flow**: Validate → Hash password → Create user → Generate token → Set cookie
- **Login Flow**: Find user → Check password → Generate token → Set cookie
- **Logout Flow**: Clear token cookie
- **Password Security**: Passwords are hashed before storing
- **Cookie Management**: JWT tokens are stored in cookies for automatic sending

---

### 📄 `src/routes/profile.js` - Profile Management Routes

**What does it do?** Handles viewing and editing user profiles.

**Think of it as:** A profile editor where users can view and update their information.

```javascript
const express = require("express");
const profileRouter = express.Router();
const { userAuth } = require("../middlewares/auth");
const { validateEditProfileData } = require("../utils/validation");

// GET /profile/view - View user's own profile
profileRouter.get("/profile/view", userAuth, async (req, res) => {
  try {
    const user = req.user;  // User data from auth middleware
    res.send(user);
  } catch (err) {
    res.status(400).send("ERROR : " + err.message);
  }
});

// PATCH /profile/edit - Edit user profile
profileRouter.patch("/profile/edit", userAuth, async (req, res) => {
  try {
    // 1. Validate that only allowed fields are being updated
    if (!validateEditProfileData(req)) {
      throw new Error("Invalid Edit Request");
    }

    const loggedInUser = req.user;

    // 2. Update user fields with new data
    Object.keys(req.body).forEach((key) => (loggedInUser[key] = req.body[key]));

    // 3. Save changes to database
    await loggedInUser.save();

    // 4. Send success response
    res.send({
      message: `${loggedInUser.firstName}, your profile updated successfully`,
      data: loggedInUser,
    });
  } catch (err) {
    res.status(400).send("ERROR : " + err.message);
  }
});

module.exports = profileRouter;
```

### 🎯 Key Points:
- **Protected Routes**: Uses `userAuth` middleware to ensure user is logged in
- **Profile Viewing**: Returns current user's profile data
- **Profile Editing**: Allows updating specific fields with validation
- **Field Validation**: Only allows certain fields to be updated
- **User Data Access**: Uses `req.user` from auth middleware

---

### 📄 `src/routes/request.js` - Connection Request Routes

**What does it do?** Handles sending and managing connection requests between users.

**Think of it as:** A matchmaking service that handles friend requests.

```javascript
const express = require("express");
const requestRouter = express.Router();
const { userAuth } = require("../middlewares/auth");
const ConnectionRequest = require("../models/connectionRequest");
const User = require("../models/user");

// POST /request/send/:status/:toUserId - Send connection request
requestRouter.post("/request/send/:status/:toUserId", userAuth, async (req, res) => {
  try {
    const fromUserId = req.user._id;  // Current user
    const toUserId = req.params.toUserId;  // Target user
    const status = req.params.status;  // "interested" or "ignored"

    // 1. Validate status
    const allowedStatus = ["ignored", "interested"];
    if (!allowedStatus.includes(status)) {
      return res.status(400).json({ message: "Invalid status type: " + status });
    }

    // 2. Check if target user exists
    const toUser = await User.findById(toUserId);
    if (!toUser) {
      return res.status(404).json({ message: "User not found!" });
    }

    // 3. Check if request already exists
    const existingConnectionRequest = await ConnectionRequest.findOne({
      $or: [
        { fromUserId, toUserId },
        { fromUserId: toUserId, toUserId: fromUserId },
      ],
    });
    if (existingConnectionRequest) {
      return res.status(400).send({ message: "Connection Request Already Exists!!" });
    }

    // 4. Create new connection request
    const connectionRequest = new ConnectionRequest({
      fromUserId,
      toUserId,
      status,
    });

    const data = await connectionRequest.save();

    // 5. Send success response
    res.json({
      message: req.user.firstName + " is " + status + " in " + toUser.firstName,
      data,
    });
  } catch (err) {
    res.status(400).send("ERROR: " + err.message);
  }
});

// POST /request/review/:status/:requestId - Review connection request
requestRouter.post("/request/review/:status/:requestId", userAuth, async (req, res) => {
  try {
    const loggedInUser = req.user;
    const { status, requestId } = req.params;  // "accepted" or "rejected"

    // 1. Validate status
    const allowedStatus = ["accepted", "rejected"];
    if (!allowedStatus.includes(status)) {
      return res.status(400).json({ message: "Status not allowed!" });
    }

    // 2. Find the connection request
    const connectionRequest = await ConnectionRequest.findOne({
      _id: requestId,
      toUserId: loggedInUser._id,  // Request must be sent to current user
      status: "interested",        // Request must be in "interested" status
    });
    if (!connectionRequest) {
      return res.status(404).json({ message: "Connection request not found" });
    }

    // 3. Update the status
    connectionRequest.status = status;
    const data = await connectionRequest.save();

    // 4. Send success response
    res.json({ message: "Connection request " + status, data });
  } catch (err) {
    res.status(400).send("ERROR: " + err.message);
  }
});

module.exports = requestRouter;
```

### 🎯 Key Points:
- **Request Sending**: Users can send "interested" or "ignored" requests
- **Duplicate Prevention**: Checks for existing requests before creating new ones
- **Request Review**: Users can accept or reject incoming requests
- **Status Management**: Tracks different states of connection requests
- **User Validation**: Ensures target users exist and requests are valid

---

### 📄 `src/routes/user.js` - User Discovery Routes

**What does it do?** Handles finding users, showing connections, and managing the user feed.

**Think of it as:** A user directory that helps people discover and connect with others.

```javascript
const express = require("express");
const userRouter = express.Router();
const User = require("../models/user");
const { userAuth } = require("../middlewares/auth");
const ConnectionRequest = require("../models/connectionRequest");

const USER_SAFE_DATA = "firstName lastName photoUrl age gender about skills";

// GET /user/requests/received - Get pending connection requests
userRouter.get("/user/requests/received", userAuth, async (req, res) => {
  try {
    const loggedInUser = req.user;

    // Find requests sent to current user with "interested" status
    const connectionRequests = await ConnectionRequest.find({
      toUserId: loggedInUser._id,
      status: "interested",
    }).populate("fromUserId", USER_SAFE_DATA);  // Get sender's safe data

    res.json({
      message: "Data fetched successfully",
      data: connectionRequests,
    });
  } catch (err) {
    res.status(400).send("ERROR: " + err.message);
  }
});

// GET /user/connections - Get user's accepted connections
userRouter.get("/user/connections", userAuth, async (req, res) => {
  try {
    const loggedInUser = req.user;

    // Find all accepted connections (either sent or received)
    const connectionRequests = await ConnectionRequest.find({
      $or: [
        { toUserId: loggedInUser._id, status: "accepted" },
        { fromUserId: loggedInUser._id, status: "accepted" },
      ],
    })
      .populate("fromUserId", USER_SAFE_DATA)
      .populate("toUserId", USER_SAFE_DATA);

    // Extract the other user from each connection
    const data = connectionRequests.map((row) => {
      if (row.fromUserId._id.toString() === loggedInUser._id.toString()) {
        return row.toUserId;  // If I sent the request, return the receiver
      }
      return row.fromUserId;  // If I received the request, return the sender
    });

    res.json({ data });
  } catch (err) {
    res.status(400).send({ message: err.message });
  }
});

// GET /feed - Get users for discovery (with pagination)
userRouter.get("/feed", userAuth, async (req, res) => {
  try {
    const loggedInUser = req.user;

    // Pagination setup
    const page = parseInt(req.query.page) || 1;
    let limit = parseInt(req.query.limit) || 10;
    limit = limit > 50 ? 50 : limit;  // Maximum 50 users per page
    const skip = (page - 1) * limit;

    // Find all users that current user has interacted with
    const connectionRequests = await ConnectionRequest.find({
      $or: [{ fromUserId: loggedInUser._id }, { toUserId: loggedInUser._id }],
    }).select("fromUserId toUserId");

    // Create a set of users to exclude from feed
    const hideUsersFromFeed = new Set();
    connectionRequests.forEach((req) => {
      hideUsersFromFeed.add(req.fromUserId.toString());
      hideUsersFromFeed.add(req.toUserId.toString());
    });

    // Find users not in the exclusion list
    const users = await User.find({
      $and: [
        { _id: { $nin: Array.from(hideUsersFromFeed) } },  // Not in exclusion list
        { _id: { $ne: loggedInUser._id } },                // Not the current user
      ],
    })
      .select(USER_SAFE_DATA)  // Only safe data
      .skip(skip)              // Skip for pagination
      .limit(limit);           // Limit results

    res.json({ data: users });
  } catch (err) {
    res.status(400).json({ message: err.message });
  }
});

module.exports = userRouter;
```

### 🎯 Key Points:
- **Request Management**: Shows pending connection requests
- **Connection Listing**: Displays all accepted connections
- **User Discovery**: Feed shows users you haven't interacted with
- **Pagination**: Limits results and allows page navigation
- **Data Safety**: Only returns safe user data (no passwords, emails)
- **Efficient Queries**: Uses indexes and proper filtering

---

## 🛠️ 7. `src/utils/` - Utility Functions Folder

**What does it do?** Contains helper functions that are used across different parts of the application.

**Think of it as:** A toolbox with common tools that different departments can use.

---

### 📄 `src/utils/validation.js` - Input Validation Functions

**What does it do?** Checks if the data sent by users is correct and safe.

**Think of it as:** A security guard that checks everyone's ID before they enter.

```javascript
const validator = require("validator");

// Validate signup data
const validateSignUpData = (req) => {
  const { firstName, lastName, emailId, password } = req.body;
  
  // Check if names are provided
  if (!firstName || !lastName) {
    throw new Error("name is not valid");
  }
  
  // Check if email is valid format
  else if (!validator.isEmail(emailId)) {
    throw new Error("please enter valid email");
  }
  
  // Check if password is strong enough
  else if (!validator.isStrongPassword(password)) {
    throw new Error("enter strong password");
  }
};

// Validate profile edit data
const validateEditProfileData = (req) => {
  // List of fields that users are allowed to edit
  const allowedEditFields = [
    "firstName",
    "lastName",
    "emailId",
    "photoUrl",
    "gender",
    "age",
    "about",
    "skills",
  ];

  // Check if all fields in request are allowed
  const isEditAllowed = Object.keys(req.body).every((field) =>
    allowedEditFields.includes(field)
  );

  return isEditAllowed;
};

module.exports = {
  validateSignUpData,
  validateEditProfileData,
};
```

### 🎯 Key Points:
- **Input Validation**: Checks email format and password strength
- **Field Validation**: Ensures only allowed fields can be updated
- **Security**: Prevents malicious data from being processed
- **Error Messages**: Provides clear error messages for users
- **Reusable**: Functions can be used across different routes

---

## 📦 8. `package.json` - Project Configuration

**What does it do?** Contains project information, dependencies, and scripts.

**Think of it as:** A recipe book that lists all ingredients (dependencies) and instructions (scripts) needed to run the project.

### Key Sections:
- **Dependencies**: Lists all packages your project needs
- **Scripts**: Commands to run, test, and build your project
- **Project Info**: Name, version, description, author

---

## 🔄 How Everything Works Together

### 1. **Request Flow**:
```
Frontend Request → app.js → Middleware → Route Handler → Model → Database → Response
```

### 2. **Authentication Flow**:
```
Login Request → auth.js route → Password Check → JWT Creation → Cookie Set → Protected Access
```

### 3. **Data Flow**:
```
User Input → Validation → Model → Database → Response
```

### 4. **Connection Flow**:
```
Send Request → request.js → connectionRequest.js model → Database → Notification
```

This detailed explanation should help you understand exactly how each part of your DevConnect project works and why it's structured this way!

