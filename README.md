# ExpressJs-101

# Express.js & MongoDB Backend Development - Complete Guide

## Table of Contents
1. [Introduction to Express.js](#introduction-to-expressjs)
2. [Project Setup & Installation](#project-setup--installation)
3. [Basic API Building](#basic-api-building)
4. [Express Router System](#express-router-system)
5. [Middleware Architecture](#middleware-architecture)
6. [Error Handling](#error-handling)
7. [MongoDB Integration with Mongoose](#mongodb-integration-with-mongoose)
8. [CRUD Operations](#crud-operations)
9. [Project Structure & File Interaction](#project-structure--file-interaction)
10. [API Documentation](#api-documentation)
11. [Testing & Outputs](#testing--outputs)

---

## Introduction to Express.js

Express.js is a minimal and flexible Node.js web application framework that provides a robust set of features for web and mobile applications. It simplifies the process of building server-side applications with Node.js.

### Key Features:
- **Minimalist**: Lightweight and unopinionated
- **Middleware**: Extensive middleware support
- **Routing**: Powerful routing system
- **HTTP Helpers**: Reduces redundant code
- **View System**: Supports multiple template engines

---

## Project Setup & Installation

### Initial Setup Process:

```bash
# Initialize Node.js project
npm init -y

# Install Express
npm install express

# Install development dependencies
npm install -D nodemon

# Install production dependencies
npm install mongoose dotenv express-async-handler
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
```

---

## Basic API Building

### Server Initialization (server.js)

```javascript
const express = require("express");
const dotenv = require("dotenv").config();
const app = express();

const port = process.env.PORT || 5000;

// Middleware to parse JSON
app.use(express.json());

// Basic route
app.get("/api/contacts", (req, res) => {
    res.status(200).json({ message: "Get all contacts" });
});

app.listen(port, () => {
    console.log(`Server running on port ${port}`);
});
```

### Key Concepts:
- **app.use(express.json())**: Middleware to parse incoming JSON requests
- **app.listen()**: Starts the server on specified port
- **Route Handling**: Defining endpoints and their responses

---

## Express Router System

### Router Creation (contactRoutes.js)

```javascript
const express = require("express");
const router = express.Router();
const {
    getContacts, 
    createContacts, 
    getContact, 
    updateContact, 
    deleteContact
} = require("../controllers/contactController");

// Consolidated routing
router.route("/")
    .get(getContacts)
    .post(createContacts);

router.route("/:id")
    .get(getContact)
    .put(updateContact)
    .delete(deleteContact);

module.exports = router;
```

### Router Integration in Server:
```javascript
app.use("/api/contacts", require("./routes/contactRoutes"));
```

### Benefits of Router System:
- **Modularity**: Separate route logic from main server file
- **Reusability**: Routes can be mounted in multiple places
- **Organization**: Clean and maintainable code structure
- **Scalability**: Easy to add new routes

---

## Middleware Architecture

### What is Middleware?
Middleware functions are functions that have access to the request object (req), response object (res), and the next middleware function in the application's request-response cycle.

### Types of Middleware in Your Project:

#### 1. Built-in Middleware
```javascript
app.use(express.json()); // Parses incoming JSON
```

#### 2. Router Middleware
```javascript
app.use("/api/contacts", contactRoutes);
```

#### 3. Error Handling Middleware
```javascript
app.use(errorHandler);
```

#### 4. Custom Middleware (Error Handler)
```javascript
const errorHandler = (err, req, res, next) => {
    const statusCode = res.statusCode ? res.statusCode : 500;
    res.status(statusCode).json({
        title: getErrorTitle(statusCode),
        message: err.message,
        stackTrace: err.stack
    });
};
```

### Middleware Execution Flow:
```mermaid
graph TD
    A[Request] --> B[express.json]
    B --> C[Router Middleware]
    C --> D[Route Handler]
    D --> E[Controller]
    E --> F[Response]
    F --> G[Error Handler if error]
```

---

## Error Handling

### Custom Error Handler (errorHandler.js)

```javascript
const { constants } = require('../constants');

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
        case constants.NOT_FOUND:
            res.json({
                title: "Not Found",
                message: err.message,
                stackTrace: err.stack
            });
            break;
        // ... other cases
        default:
            console.log("No Error, ALL good");
            break;
    }
};
```

### Error Constants (constants.js)
```javascript
constants = {
    VALIDATION_ERROR: 400,
    UNAUTHORIZED: 401,
    FORBIDDEN: 403,
    NOT_FOUND: 404,
    SERVER_ERROR: 500
}
```

### Async Error Handling with express-async-handler
```javascript
const asyncHandler = require("express-async-handler");

const getContacts = asyncHandler(async (req, res) => {
    const contacts = await Contact.find();
    res.status(200).json(contacts);
});
```

**Benefits:**
- Automatic error propagation
- No need for try-catch blocks
- Cleaner code structure

---

## MongoDB Integration with Mongoose

### Database Connection (dbConnection.js)

```javascript
const mongoose = require("mongoose");

const connectDb = async () => {
    try {
        const connect = await mongoose.connect(process.env.MONGODB_URI);
        console.log("Connection Established: ", 
            connect.connection.host,
            connect.connection.name
        );
    } catch(err) {
        console.log(err);
        process.exit(1);
    }
};

module.exports = connectDb;
```

### Data Model (contactModel.js)

```javascript
const mongoose = require("mongoose");

const contactSchema = mongoose.Schema({
    name: {
        type: String,
        required: [true, "Please add the contact name"],
    },
    email: {
        type: String,
        required: [true, "Please add the contact email address"],
    },
    phone: {
        type: String,
        required: [true, "Please add the contact Phone number"],
    }
}, {
    timestamps: true,
});

module.exports = mongoose.model("Contact", contactSchema);
```

### Mongoose Features Used:
- **Schema Definition**: Structured data validation
- **Model Creation**: Database interaction interface
- **Timestamps**: Automatic createdAt and updatedAt fields
- **Validation**: Built-in required field validation

---

## CRUD Operations

### 1. Create Operation
```javascript
const createContacts = asyncHandler(async (req, res) => {
    const { name, email, phone } = req.body;
    
    if (!name || !email || !phone) {
        res.status(400);
        throw new Error("All Fields are Mandatory!");
    }
    
    const contact = await Contact.create({ name, email, phone });
    res.status(201).json(contact);
});
```

### 2. Read Operations
#### Get All Contacts:
```javascript
const getContacts = asyncHandler(async (req, res) => {
    const contacts = await Contact.find();
    res.status(200).json(contacts);
});
```

#### Get Single Contact:
```javascript
const getContact = asyncHandler(async (req, res) => {
    const contact = await Contact.findById(req.params.id);
    if (!contact) {
        res.status(404);
        throw new Error("Contact Not Found");
    }
    res.status(200).json(contact);
});
```

### 3. Update Operation
```javascript
const updateContact = asyncHandler(async (req, res) => {
    const contact = await Contact.findById(req.params.id);
    if (!contact) {
        res.status(404);
        throw new Error("Contact Does not exist");
    }

    const updatedContact = await Contact.findByIdAndUpdate(
        req.params.id,
        req.body,
        { new: true }
    );
    res.status(200).json(updatedContact);
});
```

### 4. Delete Operation
```javascript
const deleteContact = asyncHandler(async (req, res) => {
    const contact = await Contact.findById(req.params.id);
    if (!contact) {
        res.status(404);
        throw new Error("Contact Not found!");
    }
    await Contact.deleteOne({ _id: req.params.id });
    res.status(200).json(contact);
});
```

---

## Project Structure & File Interaction

### File Architecture:
```mermaid
graph TD
    A[server.js] --> B[routes/contactRoutes.js]
    B --> C[controllers/contactController.js]
    C --> D[models/contactModel.js]
    C --> E[middleware/errorHandler.js]
    A --> F[config/dbConnection.js]
    F --> G[MongoDB Database]
    
    style A fill:#e1f5fe
    style B fill:#f3e5f5
    style C fill:#e8f5e8
    style D fill:#fff3e0
    style F fill:#ffebee
```

### File Responsibilities:

#### server.js - Application Entry Point
- Server initialization
- Middleware configuration
- Route mounting
- Database connection
- Server startup

#### contactRoutes.js - Route Definitions
- HTTP method routing
- Parameter handling
- Route consolidation

#### contactController.js - Business Logic
- Data processing
- Database operations
- Request validation
- Response formatting

#### contactModel.js - Data Schema
- Data structure definition
- Validation rules
- Database interface

#### errorHandler.js - Error Management
- Error categorization
- Response formatting
- Stack trace handling

#### dbConnection.js - Database Configuration
- Connection establishment
- Error handling
- Connection monitoring

---

## API Documentation

### Contacts API Endpoints

#### Base URL: `http://localhost:5000/api/contacts`

| Method | Endpoint | Description | Parameters | Body |
|--------|----------|-------------|------------|------|
| **GET** | `/` | Get all contacts | None | None |
| **POST** | `/` | Create new contact | None | `{name, email, phone}` |
| **GET** | `/:id` | Get single contact | `id` | None |
| **PUT** | `/:id` | Update contact | `id` | `{name, email, phone}` |
| **DELETE** | `/:id` | Delete contact | `id` | None |

### Request/Response Examples

#### Create Contact Request:
```json
{
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "123-456-7890"
}
```

#### Success Response:
```json
{
    "_id": "507f1f77bcf86cd799439011",
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "123-456-7890",
    "createdAt": "2024-01-15T10:30:00.000Z",
    "updatedAt": "2024-01-15T10:30:00.000Z"
}
```

#### Error Response:
```json
{
    "title": "Validation Error",
    "message": "All Fields are Mandatory!",
    "stackTrace": "..."
}
```

---

## Testing & Outputs

### API Testing with Thunder Client

#### 1. Get All Contacts
**Request:** `GET http://localhost:5000/api/contacts`
<img width="1137" height="634" alt="Query Parameters" src="https://github.com/user-attachments/assets/3fef16e1-daa2-4724-98c4-961d6626dd82" />

#### 2. Get an Individual Contacts
**Request:** `GET http://localhost:5000/api/contacts/:id`
<img width="1142" height="634" alt="image" src="https://github.com/user-attachments/assets/91f16de1-9d1a-452b-80df-006102d81b7e" />


#### 3. Create Contact
**Request:** `POST http://localhost:5000/api/contacts`
**Body:** 
```json
{
    "name": "Alice Johnson",
    "email": "alice@example.com", 
    "phone": "555-0123"
}
```

<img width="1142" height="653" alt="Pasted Graphic 2" src="https://github.com/user-attachments/assets/64070fcd-ba15-4c1c-8531-ed9b4f050e57" />


#### 4. Update Contact
**Request:** `PUT http://localhost:5000/api/contacts/:id`
**Body:**
```json
{
    "name": "Alice Smith",
    "email": "alice.smith@example.com"
}
```
<img width="1135" height="635" alt="Pasted Graphic 3" src="https://github.com/user-attachments/assets/1e02a7af-f9fb-4546-9301-2b238e0c45d2" />


#### 5. Delete Contact
**Request:** `DELETE http://localhost:5000/api/contacts/:id`
<img width="1138" height="644" alt="Pasted Graphic 4" src="https://github.com/user-attachments/assets/a835cb35-ce2e-41cc-8dd2-8bf71ea85a8c" />



---

## Key Learning Outcomes

### Express.js Concepts Mastered:
1. **Middleware Architecture**: Understanding request-response cycle
2. **Routing System**: Modular and scalable route management
3. **Error Handling**: Centralized error management
4. **API Design**: RESTful principles implementation

### MongoDB/Mongoose Skills:
1. **Schema Design**: Data structure and validation
2. **CRUD Operations**: Complete data manipulation
3. **Connection Management**: Database connectivity
4. **Async Operations**: Proper async/await usage



*This documentation provides a comprehensive overview of your Express.js learning journey and serves as a reference for future projects. The modular architecture and best practices implemented will scale well for larger applications.*
