# Practical Lab 3: Server-Side Templating with Node.js and EJS

## Objective
In this lab, you will learn about server-side templating by creating a simple Node.js application using the EJS templating engine. You'll set up a basic web server, create templates, and render dynamic content.

## Prerequisites
- Node.js installed on your computer
- Basic understanding of JavaScript and HTML
- Familiarity with command-line operations

## Estimated Time
1 hour

## Use Case
You're a web developer tasked with creating a simple dynamic website for a local bookstore. The bookstore wants to display their inventory with the ability to easily update the content without changing the HTML structure. Server-side templating will allow you to achieve this efficiently.

## Steps

### 1. Set up the project

1. Create a new directory for your project:
   ```
   mkdir bookstore-app
   cd bookstore-app
   ```

2. Initialize a new Node.js project:
   ```
   npm init -y
   ```

3. Install the required dependencies:
   ```
   npm install express ejs
   ```

### 2. Create the server file

Create a new file named `server.js` with the following content:

```javascript
const express = require('express');
const app = express();
const port = 3000;

// Set EJS as the view engine
app.set('view engine', 'ejs');

// Serve static files from the 'public' directory
app.use(express.static('public'));

// Sample data
const books = [
  { title: 'The Great Gatsby', author: 'F. Scott Fitzgerald', year: 1925 },
  { title: 'To Kill a Mockingbird', author: 'Harper Lee', year: 1960 },
  { title: '1984', author: 'George Orwell', year: 1949 }
];

// Route for the home page
app.get('/', (req, res) => {
  res.render('index', { books: books });
});

// Start the server
app.listen(port, () => {
  console.log(`Server running at http://localhost:${port}`);
});
```

### 3. Create the EJS template

1. Create a new directory named `views`:
   ```
   mkdir views
   ```

2. Create a new file named `index.ejs` in the `views` directory with the following content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Bookstore Inventory</title>
    <link rel="stylesheet" href="/styles.css">
</head>
<body>
    <h1>Bookstore Inventory</h1>
    <ul>
        <% books.forEach(function(book) { %>
            <li>
                <strong><%= book.title %></strong> by <%= book.author %> (<%= book.year %>)
            </li>
        <% }); %>
    </ul>
</body>
</html>
```

### 4. Add some basic styling

1. Create a new directory named `public`:
   ```
   mkdir public
   ```

2. Create a new file named `styles.css` in the `public` directory with the following content:

```css
body {
    font-family: Arial, sans-serif;
    max-width: 800px;
    margin: 0 auto;
    padding: 20px;
}

h1 {
    color: #333;
}

ul {
    list-style-type: none;
    padding: 0;
}

li {
    margin-bottom: 10px;
    padding: 10px;
    background-color: #f0f0f0;
    border-radius: 5px;
}
```

### 5. Run the application

Start the server by running:
```
node server.js
```

Open a web browser and navigate to `http://localhost:3000`. You should see the list of books rendered using the EJS template.

### 6. Modify the template

1. Add a new section to display the total number of books. Update the `index.ejs` file by adding the following code just before the closing `</body>` tag:

```html
<p>Total number of books: <%= books.length %></p>
```

2. Refresh the browser to see the changes.

### 7. Add a new route and template

1. Create a new file named `book.ejs` in the `views` directory with the following content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><%= book.title %></title>
    <link rel="stylesheet" href="/styles.css">
</head>
<body>
    <h1><%= book.title %></h1>
    <p>Author: <%= book.author %></p>
    <p>Year: <%= book.year %></p>
    <a href="/">Back to inventory</a>
</body>
</html>
```

2. Add a new route to `server.js` just before the `app.listen()` call:

```javascript
app.get('/book/:id', (req, res) => {
  const bookId = parseInt(req.params.id);
  const book = books[bookId];
  if (book) {
    res.render('book', { book: book });
  } else {
    res.status(404).send('Book not found');
  }
});
```

3. Update the `index.ejs` file to include links to individual book pages:

```html
<ul>
    <% books.forEach(function(book, index) { %>
        <li>
            <strong><a href="/book/<%= index %>"><%= book.title %></a></strong> by <%= book.author %> (<%= book.year %>)
        </li>
    <% }); %>
</ul>
```

4. Restart the server and refresh the browser to see the changes.

## Conclusion
In this lab, you've learned how to implement server-side templating using Node.js and EJS. You've created a simple dynamic website that renders content from a data source, demonstrating the power and flexibility of server-side templating.

## Additional Exercises
1. Add a form to allow adding new books to the inventory.
2. Implement a search functionality to filter books by title or author.
3. Create a partial template for the header and footer, and include it in both `index.ejs` and `book.ejs`.
4. Experiment with conditional rendering in the templates (e.g., display a special message for books published before 1950).

Remember to explore the EJS documentation for more advanced features and best practices in server-side templating.
