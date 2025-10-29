

# ExpressJs-101 - Complete Backend with Authentication

## Table of Contents
1. [Introduction to Express.js](#introduction-to-expressjs)
2. [Project Setup & Installation](#project-setup--installation)
3. [Basic API Building](#basic-api-building)
4. [Express Router System](#express-router-system)
5. [Middleware Architecture](#middleware-architecture)
6. [Error Handling](#error-handling)
7. [MongoDB Integration with Mongoose](#mongodb-integration-with-mongoose)
8. [CRUD Operations](#crud-operations)
9. [User Authentication System](#user-authentication-system)
10. [JWT & Protected Routes](#jwt--protected-routes)
11. [Project Structure & File Interaction](#project-structure--file-interaction)
12. [API Documentation](#api-documentation)
13. [Testing & Outputs](#testing--outputs)

---

## Introduction to Express.js

Express.js is a minimal and flexible Node.js web application framework that provides a robust set of features for web and mobile applications. It simplifies the process of building server-side applications with Node.js.

### Key Features:
- **Minimalist**: Lightweight and unopinionated
- **Middleware**: Extensive middleware support
- **Routing**: Powerful routing system
- **HTTP Helpers**: Reduces redundant code
- **Authentication Ready**: Perfect for building secure APIs

---

## Project Setup & Installation

### Initial Setup Process:

```bash
# Initialize Node.js project
npm init -y

# Install Express and core dependencies
npm install express mongoose dotenv express-async-handler bcrypt jsonwebtoken

# Install development dependencies
npm install -D nodemon
```

### Package.json Configuration:
```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  }
}
```

### Environment Setup (.env):
```env
MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/database
PORT=5000
JWT_SECRET=your_super_secret_key
```

---

## Basic API Building

### Server Initialization (server.js)

```javascript
const express = require("express");
const dotenv = require("dotenv").config();
const connectDb = require("./config/dbConnection");
const app = express();

connectDb();
const port = process.env.PORT || 5000;

// Middleware to parse JSON
app.use(express.json());

// Route mounting
app.use("/api/contacts", require("./routes/contactRoutes"));
app.use("/api/users", require("./routes/userRoute"));

// Error handling middleware
app.use(errorHandler);

app.listen(port, () => {
    console.log(`Server running on port ${port}`);
});
```

---

## Express Router System

### Router Architecture:
Your project now uses two main routers:

#### 1. Contact Routes (Protected)
```javascript
// contactRoutes.js
router.use(validateToken); // All contact routes are protected
router.route("/")
    .get(getContacts)
    .post(createContacts);

router.route("/:id")
    .get(getContact)
    .put(updateContact)
    .delete(deleteContact);
```

#### 2. User Routes (Mixed Access)
```javascript
// userRoute.js
router.post("/register", registerUser); // Public
router.post("/login", loginUser); // Public
router.get("/current", validateToken, currentUser); // Protected
```

### Benefits of This Structure:
- **Security**: Sensitive routes are protected
- **Flexibility**: Public and private routes coexist
- **Maintainability**: Clear separation of concerns
- **Scalability**: Easy to add new protected routes

---

## Middleware Architecture

### Middleware Types in Your Project:

#### 1. Built-in Middleware
```javascript
app.use(express.json()); // JSON parsing
```

#### 2. Router Middleware
```javascript
app.use("/api/contacts", contactRoutes);
app.use("/api/users", userRoutes);
```

#### 3. Custom Authentication Middleware
```javascript
// validateTokenHandler.js
const validateToken = asyncHandler(async (req, res, next) => {
    const authHeader = req.headers.authorization || req.headers.Authorization;
    
    if (!authHeader || !authHeader.startsWith("Bearer ")) {
        return res.status(401).json({ error: "Bearer token required" });
    }
    
    const token = authHeader.split(" ")[1];
    const decoded = jwt.verify(token, "Dhruvil123");
    req.user = decoded.user; // Attach user to request
    next(); // Continue to route handler
});
```

#### 4. Error Handling Middleware
```javascript
app.use(errorHandler); // Always last middleware
```

### Middleware Execution Flow:
```mermaid
graph TD
    A[Request] --> B[express.json]
    B --> C{Route Type}
    C -->|Public| D[Public Route]
    C -->|Protected| E[validateToken]
    E --> F[Route Handler]
    D --> G[Response]
    F --> G
    G --> H[Error Handler]
```

---

## Error Handling

### Centralized Error Handler:
```javascript
const errorHandler = (err, req, res, next) => {
    const statusCode = res.statusCode ? res.statusCode : 500;
    
    switch(statusCode) {
        case constants.VALIDATION_ERROR:
            res.json({
                title: "Validation Error",
                message: err.message,
                stackTrace: err.stack
            });
            break;
        // ... other status codes
        default:
            console.log("No Error, ALL good");
            break;
    }
};
```

### Async Error Handling:
```javascript
const asyncHandler = require("express-async-handler");

const getContacts = asyncHandler(async (req, res) => {
    const contacts = await Contact.find({user_id: req.user.id});
    res.status(200).json(contacts);
});
```

---

## MongoDB Integration with Mongoose

### Database Connection:
```javascript
const connectDb = async () => {
    try {
        const connect = await mongoose.connect(process.env.MONGODB_URI);
        console.log("Connection Established: ", connect.connection.host);
    } catch(err) {
        console.log(err);
        process.exit(1);
    }
};
```

### Data Models:

#### User Model:
```javascript
const userSchema = mongoose.Schema({
    username: { type: String, required: true },
    email: { type: String, required: true, unique: true },
    password: { type: String, required: true }
}, { timestamps: true });
```

#### Contact Model (with User Reference):
```javascript
const contactSchema = mongoose.Schema({
    user_id: {
        type: mongoose.Schema.Types.ObjectId,
        required: true,
        ref: "User" // Relationship with User model
    },
    name: { type: String, required: true },
    email: { type: String, required: true },
    phone: { type: String, required: true }
}, { timestamps: true });
```

---

## CRUD Operations

### Protected CRUD Operations:
All contact operations now include user authorization:

```javascript
// In contactController.js
const getContacts = asyncHandler(async (req, res) => {
    const contacts = await Contact.find({ user_id: req.user.id });
    res.status(200).json(contacts);
});

const createContacts = asyncHandler(async (req, res) => {
    const contact = await Contact.create({
        name: req.body.name,
        email: req.body.email,
        phone: req.body.phone,
        user_id: req.user.id // Automatically assign to logged-in user
    });
    res.status(201).json(contact);
});
```

### Authorization Checks:
```javascript
const updateContact = asyncHandler(async (req, res) => {
    const contact = await Contact.findById(req.params.id);
    
    // Check if contact belongs to current user
    if (contact.user_id.toString() !== req.user.id) {
        return res.status(403).json({ error: "Unauthorized User" });
    }
    
    const updatedContact = await Contact.findByIdAndUpdate(
        req.params.id, req.body, { new: true }
    );
    res.status(200).json(updatedContact);
});
```

---

## User Authentication System

### 1. User Registration
```javascript
const registerUser = asyncHandler(async (req, res) => {
    const { username, email, password } = req.body;
    
    // Validation
    if (!username || !email || !password) {
        res.status(400);
        throw new Error("Please Enter all Fields");
    }
    
    // Check if user exists
    const userAvailable = await User.findOne({ email });
    if (userAvailable) {
        return res.status(409).json({ error: "User already exists" });
    }
    
    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = await User.create({ username, email, password: hashedPassword });
    
    res.status(201).json({ _id: user.id, email: user.email });
});
```

### 2. User Login with JWT
```javascript
const loginUser = asyncHandler(async (req, res) => {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (user && (await bcrypt.compare(password, user.password))) {
        const accessToken = jwt.sign(
            { user: { username: user.username, email: user.email, id: user.id } },
            "Dhruvil123",
            { expiresIn: "15m" }
        );
        res.status(200).json({ accessToken });
    } else {
        res.status(401).json({ error: "Invalid credentials" });
    }
});
```

### 3. Password Security:
- **bcrypt**: Industry-standard password hashing
- **Salt Rounds**: 10 (balanced security & performance)
- **No Plain Text**: Passwords never stored in clear text

---

## JWT & Protected Routes

### JWT Implementation:
```javascript
// Token Generation
const accessToken = jwt.sign(
    {
        user: {
            username: user.username,
            email: user.email,
            id: user.id,
        },
    },
    "Dhruvil123", // Secret key
    { expiresIn: "15m" } // Token expiry
);

// Token Verification Middleware
const validateToken = asyncHandler(async (req, res, next) => {
    const token = req.headers.authorization.split(" ")[1];
    const decoded = jwt.verify(token, "Dhruvil123");
    req.user = decoded.user; // Make user data available in controllers
    next();
});
```

### Protected Route Patterns:

#### 1. Individual Route Protection:
```javascript
router.get("/current", validateToken, currentUser);
```

#### 2. Router-Level Protection:
```javascript
router.use(validateToken); // All routes in this router are protected
router.get("/", getContacts);
router.post("/", createContacts);
```

#### 3. Controller-Level Authorization:
```javascript
// Check resource ownership
if (contact.user_id.toString() !== req.user.id) {
    return res.status(403).json({ error: "Unauthorized User" });
}
```

---

## Project Structure & File Interaction

### Updated File Architecture:
```mermaid
graph TD
    A[server.js] --> B[routes/userRoute.js]
    B --> C[controllers/userController.js]
    C --> D[models/userModel.js]
    A --> E[routes/contactRoutes.js]
    E --> F[controllers/contactController.js]
    F --> G[models/contactModel.js]
    E --> H[middleware/validateTokenHandler.js]
    A --> I[middleware/errorHandler.js]
    A --> J[config/dbConnection.js]
    
    style B fill:#e8f5e8
    style C fill:#e8f5e8
    style E fill:#fff3e0
    style H fill:#ffebee
```

### File Responsibilities:

#### Authentication Files:
- **userRoute.js**: User authentication endpoints
- **userController.js**: Registration, login, current user logic
- **userModel.js**: User data schema with password hashing
- **validateTokenHandler.js**: JWT verification middleware

#### Protected Application Files:
- **contactRoutes.js**: All routes protected by validateToken
- **contactController.js**: Business logic with user authorization
- **contactModel.js**: Contact schema with user reference

#### Core Infrastructure:
- **server.js**: Application entry point and middleware setup
- **dbConnection.js**: MongoDB connection management
- **errorHandler.js**: Centralized error handling

---

## API Documentation

### Authentication Endpoints

#### Base URL: `http://localhost:5000/api/users`

| Method | Endpoint | Description | Access |
|--------|----------|-------------|---------|
| **POST** | `/register` | Create new user account | Public |
| **POST** | `/login` | User login, returns JWT token | Public |
| **GET** | `/current` | Get current user info | Private |

### Contacts Endpoints (All Protected)

#### Base URL: `http://localhost:5000/api/contacts`

| Method | Endpoint | Description | Parameters |
|--------|----------|-------------|------------|
| **GET** | `/` | Get all user's contacts | None |
| **POST** | `/` | Create new contact | None |
| **GET** | `/:id` | Get specific contact | `id` |
| **PUT** | `/:id` | Update contact | `id` |
| **DELETE** | `/:id` | Delete contact | `id` |

### Request/Response Examples

#### User Registration:
```json
// Request
{
    "username": "john_doe",
    "email": "john@example.com",
    "password": "securepassword123"
}

// Response
{
    "_id": "507f1f77bcf86cd799439011",
    "email": "john@example.com"
}
```

#### User Login:
```json
// Response
{
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### Protected Contact Creation:
```http
POST /api/contacts
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
    "name": "Alice Johnson",
    "email": "alice@example.com",
    "phone": "123-456-7890"
}
```

---

## Testing & Outputs

### Authentication Flow Testing:

#### 1. User Registration
**Request:** `POST http://localhost:5000/api/users/register`
**Expected Output:**
<img width="1505" height="974" alt="image" src="https://github.com/user-attachments/assets/47bbc7af-035f-44f0-b7e0-55f898b78ddc" />



#### 2. User Login
**Request:** `POST http://localhost:5000/api/users/login`
```json
{
    "email": "test@example.com",
    "password": "testpassword123"
}
```
<img width="1507" height="778" alt="image" src="https://github.com/user-attachments/assets/62636b8a-f0a3-44cd-82de-7473245b8f65" />


#### 3. Access Protected Route
**Request:** `GET http://localhost:5000/api/users/current`
**Headers:** `Authorization: Bearer your_token_here`
**Expected Output:**
<img width="1497" height="969" alt="image" src="https://github.com/user-attachments/assets/79a128ec-cd9e-46ce-a736-f2fdd65efe13" />


### Contact Management Testing:

#### 4. Create Contact (Protected)
**Request:** `POST http://localhost:5000/api/contacts`
**Headers:** `Authorization: Bearer your_token_here`
<img width="1494" height="755" alt="image" src="https://github.com/user-attachments/assets/516e684b-7344-40f3-bb44-16fd19ac4d12" />


#### 5. Get User's Contacts
**Request:** `GET http://localhost:5000/api/contacts`
**Headers:** `Authorization: Bearer your_token_here`
**Expected Output:**

<img width="1507" height="745" alt="image" src="https://github.com/user-attachments/assets/af9d9493-8eaf-4fcf-8332-be372cdb0b48" />


### Error Scenario Testing:

#### 6. Unauthorized Access Attempt
**Request:** `GET http://localhost:5000/api/contacts`
**No Authorization Header**
**Expected Output:**
<img width="1496" height="747" alt="image" src="https://github.com/user-attachments/assets/9306c1c1-2de9-4a24-813e-83ad097f035b" />


#### 7. Access Other User's Contact
**Request:** `PUT http://localhost:5000/api/contacts/other_users_contact_id`
**Headers:** `Authorization: Bearer your_token_here`
**Expected Output:**
<img width="1503" height="744" alt="image" src="https://github.com/user-attachments/assets/333626e2-4296-44f6-a65c-2c250bd3eed3" />


### Console Outputs During Operation:

#### Server Startup:
```
Server is running on the port 5000
Connection Established: cluster.mongodb.net mycontacts-backend
```

#### Token Validation Debug:
```
1. validateToken middleware started
2. Authorization header: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
4. Token extracted: Yes
5. Token first 20 chars: eyJhbGciOiJIUzI1NiIs...
6. Verifying token with secret...
7. Token decoded successfully: { user: { username: 'testuser', ... } }
8. User attached to request: { username: 'testuser', ... }
9. Calling next()...
```

#### User Registration:
```
Hashed Password $2b$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
User Created {
    username: 'testuser',
    email: 'test@example.com',
    password: '$2b$10$N9qo8uLOickgx2ZMRZoMye...',
    _id: new ObjectId('673a1b2c3d4e5f6789012345'),
    createdAt: 2024-01-15T10:28:18.240Z,
    updatedAt: 2024-01-15T10:28:18.240Z,
    __v: 0
}
```

---

## Key Learning Outcomes

### Advanced Express.js Concepts Mastered:
1. **Middleware Chaining**: Complex authentication flows
2. **Route Protection**: JWT-based access control
3. **User Context**: Request-based user data propagation
4. **Resource Ownership**: User-specific data isolation

### Security Implementation:
1. **Password Hashing**: bcrypt with salt rounds
2. **JWT Tokens**: Stateless authentication
3. **Authorization**: Resource-level access control
4. **Token Management**: Expiry and validation

### Database Relationships:
1. **Model References**: User-Contact relationships
2. **Data Isolation**: User-specific queries
3. **Schema Design**: Related data structures

### Production-Ready Features:
1. **Error Handling**: Comprehensive error management
2. **Input Validation**: Request data sanitization
3. **API Security**: Protected endpoints
4. **Code Organization**: Modular and maintainable structure

*This enhanced backend demonstrates professional-grade API development with complete authentication, authorization, and security measures - ready for production deployment.*
