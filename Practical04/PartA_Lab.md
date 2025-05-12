## Practical 04: Part A - Lab Activity

### Lab Activity

In this part, you will be refactoring code based on your existing Books API code to adopt the **MVC architecture** and implement key robustness and security features.

### Task 1: Project Setup & Review

1. **Create a New Project Directory:**

   - Navigate to your BED Practicals repository
   - Create a new folder: `Practical04`
   - Inside this folder, create another folder: `books-api-mvc-db`

2. **Initialize the Project:**

   - Open the terminal in the `books-api-mvc-db` directory
   - Run `npm init -y` to initialize a new Node.js project
   - Install the required packages:
     ```bash
     npm install express mssql joi dotenv
     ```

3. **Copy Configuration Files:**

   - Copy your `dbConfig.js` file from the previous practical
   - Create a new `.env` file to store environment variables:
     ```
     PORT=3000
     DB_USER=your_username
     DB_PASSWORD=your_password
     DB_SERVER=localhost
     DB_DATABASE=bed_db
     DB_PORT=1433
     ```
   - Create a `.gitignore` file and add `node_modules` and `.env` to it

   #### Explanation: Environment Variables and Security

   - **`dotenv` Package:**

     - Loads variables from `.env` into `process.env`.
     - Keeps configuration separate from code (security, flexibility).
     - Required/configured early in your app (e.g., `require('dotenv').config();`).

   - **`.env` File:**

     - Root-level file (`.env`).
     - Stores `KEY=VALUE` config (ports, credentials, etc.).
     - Values accessible via `process.env.KEY` in your code.

   - **`.gitignore` File:**
     - Tells Git which files/folders to ignore.
     - Include `node_modules/` (large dependencies).
     - **Crucially, include `.env`**: Prevents accidental commit of sensitive credentials to version control.

4. **Update dbConfig.js file:**

   - Update your `dbConfig.js` file with the environment variables:

   ```
   module.exports = {
     user: process.env.DB_USER,
     password: process.env.DB_PASSWORD,
     server: process.env.DB_SERVER,
     database: process.env.DB_DATABASE,
     trustServerCertificate: true,
     options: {
       port: parseInt(process.env.DB_PORT), // Default SQL Server port
       connectionTimeout: 60000, // Connection timeout in milliseconds
     },
   };
   ```

5. **Review the Previous Implementation:**
   - Examine your previous `app.js` file
   - Notice how all logic is contained within route handlers
   - Identify areas that would benefit from separation of concerns

### Task 2: Implement MVC Architecture

1.  **Create Folder Structure:**

    - Create the following folders within your project:
      - `models` (for database interaction logic)
      - `controllers` (for request handling logic)
      - `middlewares` (for validation middleware)

2.  **Set Up the Model:**

    - Create a file `models/bookModel.js` with the following content:

           ```javascript
           const sql = require("mssql");
           const dbConfig = require("../dbConfig");

           // Get all books
           async function getAllBooks() {
             let connection;
             try {
               connection = await sql.connect(dbConfig);
               const query = "SELECT id, title, author FROM Books";
               const result = await connection.request().query(query);
               return result.recordset;
             } catch (error) {
               console.error("Database error:", error);
               throw error;
             } finally {
               if (connection) {
                 try {
                   await connection.close();
                 } catch (err) {
                   console.error("Error closing connection:", err);
                 }
               }
             }
           }

           // Get book by ID
           async function getBookById(id) {
             let connection;
             try {
               connection = await sql.connect(dbConfig);
               const query = "SELECT id, title, author FROM Books WHERE id = @id";
               const request = connection.request();
               request.input("id", id);
               const result = await request.query(query);

               if (result.recordset.length === 0) {
                 return null; // Book not found
               }

               return result.recordset[0];
             } catch (error) {
               console.error("Database error:", error);
               throw error;
             } finally {
               if (connection) {
                 try {
                   await connection.close();
                 } catch (err) {
                   console.error("Error closing connection:", err);
                 }
               }
             }
           }

           // Create new book
           async function createBook(bookData) {
             let connection;
             try {
               connection = await sql.connect(dbConfig);
               const query =
                 "INSERT INTO Books (title, author) VALUES (@title, @author); SELECT SCOPE_IDENTITY() AS id;";
               const request = connection.request();
               request.input("title", bookData.title);
               request.input("author", bookData.author);
               const result = await request.query(query);

               const newBookId = result.recordset[0].id;
               return await getBookById(newBookId);
             } catch (error) {
               console.error("Database error:", error);
               throw error;
             } finally {
               if (connection) {
                 try {
                   await connection.close();
                 } catch (err) {
                   console.error("Error closing connection:", err);
                 }
               }
             }
           }

           module.exports = {
             getAllBooks,
             getBookById,
             createBook,
           };
           ```

      #### 🧠 Explanation

      This model file defines **three core functions** for interacting with the `Books` table in your SQL Server database:

      ***

      #### - `getAllBooks()`

      - Executes a simple `SELECT` query to retrieve all books.
      - Uses `connection.request().query(...)` to execute the SQL command.
      - Returns an array of book records.

      ***

      #### - `getBookById(id)`

      - Retrieves a single book by its ID using a **parameterized query**:
        ```sql
        SELECT id, title, author FROM Books WHERE id = @id
        ```
      - The `@id` is safely injected using `.input("id", id)`.
      - Returns the book object if found, or `null` if no record exists.
      - Parameterized queries help prevent SQL injection.

      ***

      #### - `createBook(bookData)`

      - Inserts a new book using parameters `@title` and `@author`.
      - Uses `SCOPE_IDENTITY()` to get the ID of the newly inserted record:
        ```sql
        SELECT SCOPE_IDENTITY() AS id;
        ```
      - Calls `getBookById()` to return the full inserted book record.

      ***

      ### 🔐 Security Note

      All database interactions use **parameterized queries**, which:

      - Prevent SQL injection by separating data from the SQL command.
      - Safely handle user-provided input values.
      - Improve security and maintainability of the application.

      ***

      ### 🔁 Connection Management

      Each function:

      - Establishes a new SQL connection using `sql.connect(dbConfig)`.
      - Performs the query logic inside a `try` block.
      - Closes the connection inside a `finally` block to ensure cleanup even if errors occur.

      This practice prevents connection leaks and ensures stable performance.

3.  **Set Up the Controller:**

    - Create a file `controllers/bookController.js` with the following content:

      ```javascript
      const bookModel = require("../models/bookModel");

      // Get all books
      async function getAllBooks(req, res) {
        try {
          const books = await bookModel.getAllBooks();
          res.json(books);
        } catch (error) {
          console.error("Controller error:", error);
          res.status(500).json({ error: "Error retrieving books" });
        }
      }

      // Get book by ID
      async function getBookById(req, res) {
        try {
          const id = parseInt(req.params.id);
          if (isNaN(id)) {
            return res.status(400).json({ error: "Invalid book ID" });
          }

          const book = await bookModel.getBookById(id);
          if (!book) {
            return res.status(404).json({ error: "Book not found" });
          }

          res.json(book);
        } catch (error) {
          console.error("Controller error:", error);
          res.status(500).json({ error: "Error retrieving book" });
        }
      }

      // Create new book
      async function createBook(req, res) {
        try {
          const newBook = await bookModel.createBook(req.body);
          res.status(201).json(newBook);
        } catch (error) {
          console.error("Controller error:", error);
          res.status(500).json({ error: "Error creating book" });
        }
      }

      module.exports = {
        getAllBooks,
        getBookById,
        createBook,
      };
      ```

      ### 🧠 Explanation

      This controller file defines **three main endpoint handlers** that coordinate between incoming HTTP requests and the model layer:

      ***

      #### 1. `getAllBooks(req, res)`

      - Calls `bookModel.getAllBooks()` to fetch all book records.
      - Sends the result array as a JSON response (`res.json(books)`).
      - Catches and logs any errors, returning a `500 Internal Server Error` with an appropriate message.

      ***

      #### 2. `getBookById(req, res)`

      - Parses `req.params.id` into an integer and validates it, returning `400 Bad Request` if invalid.
      - Uses `bookModel.getBookById(id)` to retrieve the specific book.
      - If no book is found, returns `404 Not Found`.
      - Otherwise, responds with the book data as JSON.
      - Logs and handles unexpected errors with a `500` response.

      ***

      #### 3. `createBook(req, res)`

      - Passes `req.body` directly into `bookModel.createBook()` to insert a new book.
      - On success, returns `201 Created` with the new book object.
      - On failure, logs the error and returns `500 Internal Server Error`.

      ***

      ### 🔗 Key Points

      - **Separation of Concerns:** Controllers handle request parsing, response formatting, and error status codes while delegating data operations to the model.
      - **Error Handling:** Each method uses `try...catch` to manage exceptions, log errors, and send consistent error responses.
      - **Input Validation:** Basic parameter checks (e.g., `isNaN(id)`) ensure controllers guard against invalid inputs before invoking the model.

4.  **Create the Main Application File:**

    - Create a file `app.js` with the following content:

      ```javascript
      const express = require("express");
      const sql = require("mssql");
      const dotenv = require("dotenv");
      // Load environment variables
      dotenv.config();

      const bookController = require("./controllers/bookController");

      // Create Express app
      const app = express();
      const port = process.env.PORT || 3000;

      // Middleware
      app.use(express.json());
      app.use(express.urlencoded({ extended: true }));

      // Routes for books
      app.get("/books", bookController.getAllBooks);
      app.get("/books/:id", bookController.getBookById);
      app.post("/books", bookController.createBook);

      // Start server
      app.listen(port, () => {
        console.log(`Server running on port ${port}`);
      });

      // Graceful shutdown
      process.on("SIGINT", async () => {
        console.log("Server is gracefully shutting down");
        // Close any open connections
        await sql.close();
        console.log("Database connections closed");
        process.exit(0);
      });
      ```

      ### 🧠 Explanation

      This `app.js` file serves as the **entry point** for your Express application:

      ***

      #### 1. Environment Configuration

      - Uses `dotenv.config()` to load database credentials and other sensitive data from a `.env` file.
      - Keeps secrets out of source code.

      ***

      #### 2. Middleware Setup

      - `express.json()` parses incoming JSON payloads.
      - `express.urlencoded()` handles URL-encoded form data.
      - Ensures request bodies are available under `req.body`.

      ***

      #### 3. Routing

      - Mounts three routes for book operations:
        - `GET /books` → fetch all books
        - `GET /books/:id` → fetch a single book by ID
        - `POST /books` → create a new book
      - Routes directly reference controller methods for separation of concerns.

      ***

      #### 4. Server Launch

      - `app.listen(port, ...)` starts the HTTP server on the specified port.
      - Logs a confirmation message once running.

      ***

      #### 5. Graceful Shutdown

      - Listens for `SIGINT` (Ctrl+C) to cleanly shut down.
      - Closes any open SQL connections via `sql.close()`.
      - Prevents resource leaks and ensures a tidy exit.

      ***

5.  **Test the refactored code using Postman:**

- Run your Node.js application (`node app.js`).
- Test all the implemented **CRUD endpoints** (`GET /books`, `GET /books/:id`, `POST /books`) to ensure they work correctly after refactoring.
- Test missing data: Send `POST /books` requests with invalid or missing data to find out what happens to your application.

### Task 3: Implement Input Validation Middleware

1. **Create the Validation Middleware:**

   - Create a file `middlewares/bookValidation.js` with the following content:

   ```javascript
   const Joi = require("joi");

   // Validation schema for books
   const bookSchema = Joi.object({
     title: Joi.string().min(1).max(50).required().messages({
       "string.base": "Title must be a string",
       "string.empty": "Title cannot be empty",
       "string.min": "Title must be at least 1 character long",
       "string.max": "Title cannot exceed 50 characters",
       "any.required": "Title is required",
     }),
     author: Joi.string().min(1).max(50).required().messages({
       "string.base": "Author must be a string",
       "string.empty": "Author cannot be empty",
       "string.min": "Author must be at least 1 character long",
       "string.max": "Author cannot exceed 50 characters",
       "any.required": "Author is required",
     }),
   });

   // Middleware to validate book data
   function validateBook(req, res, next) {
     const { error } = bookSchema.validate(req.body, { abortEarly: false });

     if (error) {
       const errorMessage = error.details
         .map((detail) => detail.message)
         .join(", ");
       return res.status(400).json({ error: errorMessage });
     }

     next();
   }

   // Middleware to validate book ID
   function validateBookId(req, res, next) {
     const id = parseInt(req.params.id);

     if (isNaN(id) || id <= 0) {
       return res
         .status(400)
         .json({ error: "Invalid book ID. ID must be a positive number" });
     }

     next();
   }

   module.exports = {
     validateBook,
     validateBookId,
   };
   ```

   ### 🧠 Explanation

   This file implements **two middleware functions** using Joi for robust input validation:

   ***

   #### 1. `validateBook(req, res, next)`

   - Defines a Joi `bookSchema` requiring:
     - `title`: a non-empty string (1–50 chars)
     - `author`: a non-empty string (1–50 chars)
   - Custom error messages provide clear feedback on failure.
   - Validates `req.body` against the schema with `abortEarly: false` to collect all errors.
   - On validation error, concatenates all messages and returns `400 Bad Request` with the error details.
   - Calls `next()` if validation passes.

   ***

   #### 2. `validateBookId(req, res, next)`

   - Parses `req.params.id` into an integer.
   - Ensures the ID is a positive number.
   - Returns `400 Bad Request` if the ID is missing, non-numeric, or <= 0.
   - Calls `next()` to proceed when the ID is valid.

   ***

   ### 🔗 Key Points

   - **Centralized Validation**: keeps route handlers clean by offloading checks to middleware.
   - **Comprehensive Feedback**: `abortEarly: false` ensures users see all validation issues at once.
   - **Consistent Error Responses**: standard `400` status with JSON error messages.

2. **Update the routes in app.js file:**

   - Ensure your routes in `app.js` file is updated to include the validation middleware as shown:

   ```javascript
   const express = require("express");
   const sql = require("mssql");
   const dotenv = require("dotenv");
   // Load environment variables
   dotenv.config();

   const bookController = require("./controllers/bookController");
   const {
     validateBook,
     validateBookId,
   } = require("./middlewares/bookValidation"); // import Book Validation Middleware

   // ... other lines of code before routes for books

   // Routes for books
   app.get("/books", bookController.getAllBooks);
   app.get("/books/:id", validateBookId, bookController.getBookById); // add Validate Book Id Middleware
   app.post("/books", validateBook, bookController.createBook); // add Validate Book Middleware

   // ... rest of the code in app.js
   ```

### Task 4: Ensure Parameterized Queries

Verify that all your database interactions use **Parameterized Queries** to prevent **SQL injection**.

- **Review Model Functions:** Go back to your `bookModel.js` file.
- **Check All Queries with Variables:** Examine every SQL query that includes data from variables (especially data coming from user input via the controller, like `bookId`, `bookData.title`, `bookData.author`).
- **Verify Parameterization:** Ensure that for every such variable, you are using the `request.input('parameterName', variable)` method and referencing the parameter in the query string with `@parameterName`.

### Task 5: Testing with Postman

Thoroughly test your refactored and enhanced API using **Postman**.

- Run your Node.js application (`node app.js`).
- Test all implemented **CRUD endpoints** (`GET /books`, `GET /books/:id`, `POST /books`, `PUT /books/:id`, `DELETE /books/:id` - if applicable) to ensure they still work correctly after refactoring.
- **Test Validation:** Send `POST /books` requests with invalid or missing data to verify that your validation middleware intercepts them and returns a **400 Bad Request** response with the expected error message.
- Verify **graceful shutdown** (`Ctrl+C`) still works.

### Task 6: Implement Update and Delete Books

Do it yourself! Implement Update and Delete Books endpoints including the validation middleware based on the refactored code.

- Test the implemented endpoints (`PUT /books/:id`, `DELETE /books/:id`) to ensure they work correctly.
- **Test Validation:** Send `PUT /books` requests with invalid or missing data to verify that your validation middleware intercepts them and returns a **400 Bad Request** response with the expected error message.

#### End of Lab Activity
