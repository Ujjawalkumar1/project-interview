# 🔥 DevTinder Backend

**DevTinder** is a Tinder-inspired full-stack web application for developers to **connect**, **collaborate**, and **contribute**!  
This repository contains the complete **backend** codebase that powers the DevTinder platform — built with **Node.js**, **Express.js**, and **MongoDB**.

---

## 🚀 Overview



## 🔗 **🌐Frontend:** [devTinder Frontend](https://github.com/Ujjawalkumar1/devTinder-web.git)

This backend service provides:

- 🔐 Secure **Authentication & Authorization** using **JWT** and **bcrypt**
- 🧠 **Profile Matching System**: Like/dislike other developers and connect
- 📦 **MongoDB** integration for storing user profiles and actions
- 🍪 **Cookie-based session management** for client interactions
- ⚙️ RESTful APIs to interact with the frontend (React)

---

## 🛠️ Tech Stack

- **Node.js**
- **Express.js**
- **MongoDB** with **Mongoose**
- **JWT** for authentication
- **Bcrypt** for password hashing
- **Cookie-Parser** for managing cookies
- **CORS** enabled for secure frontend-backend communication
- **Dotenv** for environment variable management

---
## 📁 Folder Structure

```
devtinder-backend/
├── config/              # MongoDB connection setup and configuration
├── middlewares/         # Custom middleware (e.g., JWT auth)
├── models/              # Mongoose schemas (User, Matches, etc.)
├── routes/              # All API route definitions (auth, user, etc.)
├── utils/               # Utility functions (e.g., logout, profile update)
├── app.js               # Main application entry file (Express config)
├── .env                 # Environment variables (not pushed to Git)
├── .gitignore           # Files and folders to ignore in Git
├── package.json         # Project metadata and dependencies
└── README.md            # Project documentation (this file)
```




---

## 📦 Installation

### ⚙️ Prerequisites

- Node.js (v18 or higher)
- MongoDB (local or cloud - MongoDB Atlas)

### 🧑‍💻 Setup

1. Clone the backend repository:

```bash
git clone https://github.com/Ujjawalkumar1/devTinder.git



2. Install dependencies:
   ```bash
   npm install
   ```

3. Start the development server:
   ```bash
   npm run dev
   ```





