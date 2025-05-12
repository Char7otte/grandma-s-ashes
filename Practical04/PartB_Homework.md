## Practical 04: Part B - Homework

### Task 1: Reflection on Refactoring

Reflect on the process of refactoring the API code into the MVC structure by answering the following questions:

1. What were the main changes you made to refactor the code into MVC architecture?
2. What challenges did you face during the refactoring process?
3. How does the MVC structure change the way you think about adding new features?
4. In what ways is the MVC version more organized or easier to understand than the previous version?
5. How does separating concerns (Model vs. Controller) make the code better?

### Task 2: Reflecting on Robustness & Security

Reflect on the impact of validation, error handling, and parameterized queries by answering the following questions:

1. How does input validation make the API more reliable?
2. Explain in your own words how parameterized queries prevent SQL injection. Why is this more secure than building query strings with concatenation?
3. Consider a potential security risk (other than SQL injection). How might robust error handling help mitigate it?

### Task 3: Applying Concepts to Students API

Apply what you've learned to the Students API that you created in the homework for the last practical:

1. Refactor your Students API to follow the MVC architecture.
2. Implement validation middleware for student data.
3. Ensure all database interactions use parameterized queries.
4. Add appropriate error handling.

Your refactored Students API should have:

- A model (`studentModel.js`) for database operations
- A controller (`studentController.js`) for request handling
- Validation middleware for student data
- Main application file (`app.js`) tying everything together

### Submission:

Submit your answers to the reflection and exploration tasks. Ensure your answers are clear and directly address the questions asked.

- Create a text file `Homework_reflection.txt` answering the reflection questions.
- Commit and push your refactored code for both Books API and Students API to your BED Practicals **Github repository**.

### Pro-tip:

Always consult your tutor or classmates when in doubt. Pair programming or discussing challenges can be very helpful!

#### End of Homework
