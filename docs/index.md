# Level 0: Complete Beginner - Python & Web Fundamentals

## Goal

Before diving into Django REST Framework, you need to understand the fundamentals. This level covers Python basics for web development, web concepts, and environment setup. By the end, you'll be ready to start building APIs with Django.

## Table of Contents

1. [Why Learn This?](#why-learn-this)
2. [Python Basics for Web Development](#python-basics-for-web-development)
3. [Understanding the Web](#understanding-the-web)
4. [What is an API?](#what-is-an-api)
5. [Environment Setup Explained](#environment-setup-explained)
6. [Virtual Environments - Why and How](#virtual-environments---why-and-how)
7. [Understanding JSON](#understanding-json)
8. [HTTP Basics](#http-basics)
9. [Command Line Essentials](#command-line-essentials)
10. [Practice Problems](#practice-problems)
11. [Trivia](#trivia)

## Why Learn This?

### What is Django REST Framework?

**Django REST Framework (DRF)** is a powerful toolkit for building Web APIs in Django. Think of it as a set of tools that makes it easy to create APIs that can be used by web applications, mobile apps, or any other client.

### Why Use DRF?

- **Fast Development**: Build APIs quickly with less code
- **Powerful Features**: Authentication, permissions, serialization built-in
- **Well Documented**: Excellent documentation and community
- **Industry Standard**: Used by many companies worldwide
- **Flexible**: Works with any frontend (React, Vue, mobile apps)

### What You'll Build

By the end of this guide, you'll be able to build:
- REST APIs for web applications
- Backends for mobile apps
- Microservices
- Data APIs for analytics
- Integration APIs for third-party services

## Python Basics for Web Development

### Variables and Data Types

**Why this matters**: APIs work with data. Understanding Python data types helps you understand how APIs handle data.

```python
# Strings - Text data (like names, descriptions)
name = "John Doe"
description = "A book about Python"

# Integers - Whole numbers (like IDs, counts)
book_id = 1
page_count = 250

# Floats - Decimal numbers (like prices)
price = 29.99

# Booleans - True/False (like completion status)
is_published = True
is_available = False

# Lists - Ordered collections (like arrays of items)
books = ["Book 1", "Book 2", "Book 3"]
numbers = [1, 2, 3, 4, 5]

# Dictionaries - Key-value pairs (like JSON objects)
book = {
    "id": 1,
    "title": "Python Guide",
    "author": "John Doe",
    "price": 29.99
}
```

**Explanation**: 
- **Strings** are used for text data in APIs (titles, descriptions, names)
- **Integers/Floats** are used for numeric data (IDs, prices, counts)
- **Booleans** are used for yes/no states (published, completed, active)
- **Lists** are used for collections of items (list of books, list of users)
- **Dictionaries** are the most important - they represent JSON objects in APIs

### Functions

**Why this matters**: Views in Django are functions (or methods). Understanding functions helps you understand how APIs process requests.

```python
# Basic function
def greet(name):
    return f"Hello, {name}!"

# Function with multiple parameters
def create_book(title, author, price):
    book = {
        "title": title,
        "author": author,
        "price": price
    }
    return book

# Function that processes data (like an API endpoint)
def get_book_info(book_id):
    # In a real API, this would fetch from database
    if book_id == 1:
        return {"id": 1, "title": "Python Guide", "author": "John"}
    else:
        return None

# Using functions
result = greet("Alice")
book = create_book("Django Guide", "Jane", 39.99)
info = get_book_info(1)
```

**Explanation**: 
- Functions are reusable blocks of code
- In APIs, each endpoint is like a function that processes a request
- Functions take input (parameters) and return output (response)
- This is exactly how API endpoints work!

### Classes and Objects

**Why this matters**: Django uses classes extensively. Models, Views, Serializers are all classes.

```python
# Define a class (like a blueprint)
class Book:
    def __init__(self, title, author, price):
        # __init__ is called when creating an object
        self.title = title
        self.author = author
        self.price = price
    
    def get_info(self):
        # Method (function inside a class)
        return f"{self.title} by {self.author} - ${self.price}"

# Create objects (instances) from the class
book1 = Book("Python Guide", "John", 29.99)
book2 = Book("Django Guide", "Jane", 39.99)

# Use the objects
print(book1.get_info())  # "Python Guide by John - $29.99"
print(book2.title)       # "Django Guide"
```

**Explanation**:
- **Class** = Blueprint (like a cookie cutter)
- **Object** = Instance created from class (like a cookie)
- **Methods** = Functions inside a class
- In Django, `Book` model is a class, and each book in database is an object

### Importing Modules

**Why this matters**: Django and DRF are modules you import. Understanding imports is essential.

```python
# Import entire module
import json
import datetime

# Import specific functions/classes
from datetime import date
from json import loads, dumps

# Import with alias
import django.db.models as models

# Using imports
today = date.today()
data = loads('{"key": "value"}')  # Parse JSON string
```

**Explanation**:
- Python code is organized in **modules** (files) and **packages** (folders)
- `import` brings code from other files into your current file
- Django uses imports extensively: `from django.db import models`
- DRF uses imports: `from rest_framework import serializers`

## Understanding the Web

### How the Web Works

**Client-Server Model**:

```
┌─────────┐         HTTP Request          ┌─────────┐
│ Client  │ ────────────────────────────> │ Server │
│(Browser)│                                │(Django)│
└─────────┘ <────────────────────────────  └─────────┘
            HTTP Response (JSON/HTML)
```

**Explanation**:
- **Client** = Your browser, mobile app, or any application making requests
- **Server** = Django application running on a computer
- **Request** = Client asks for data ("Give me all books")
- **Response** = Server sends back data (JSON with book list)
- **HTTP** = Protocol (language) they use to communicate

### What Happens When You Visit a Website?

1. **You type URL** → `http://localhost:8000/api/books/`
2. **Browser sends HTTP GET request** → "Hey server, give me the books list"
3. **Server processes request** → Django runs code, queries database
4. **Server sends response** → JSON data with books
5. **Browser displays data** → You see the books list

**In API terms**:
- URL = **Endpoint** (where to send request)
- GET = **HTTP Method** (what action to perform)
- JSON = **Response Format** (how data is structured)

## What is an API?

### API = Application Programming Interface

**Simple Explanation**: An API is like a waiter in a restaurant.

```
You (Client)          Waiter (API)          Kitchen (Database)
   │                       │                      │
   │  "I want pizza"       │                      │
   ├──────────────────────>│                      │
   │                       │  "Get pizza"         │
   │                       ├─────────────────────>│
   │                       │                      │
   │                       │  "Here's pizza"      │
   │                       │<─────────────────────┤
   │  "Here's your pizza"  │                      │
   │<──────────────────────┤                      │
```

**Explanation**:
- **You** = Client application (browser, mobile app)
- **Waiter** = API (Django REST Framework)
- **Kitchen** = Database (where data is stored)
- **Menu** = API Documentation (what you can order)
- **Order** = API Request (what you want)
- **Food** = API Response (data you get back)

### REST API Example

**Real-world analogy**: Think of a library system

- **GET /books/** → "Show me all books" (like browsing catalog)
- **GET /books/1/** → "Show me book with ID 1" (like asking for specific book)
- **POST /books/** → "Add a new book" (like donating a book)
- **PUT /books/1/** → "Update book 1 completely" (like replacing a book)
- **PATCH /books/1/** → "Update only title of book 1" (like updating catalog entry)
- **DELETE /books/1/** → "Remove book 1" (like removing a book)

**Why REST?**
- **Standardized**: Everyone uses the same pattern
- **Predictable**: Easy to understand what each endpoint does
- **Stateless**: Each request is independent
- **Scalable**: Can handle many clients

## Environment Setup Explained

### Why Do We Need This?

**Problem**: Different projects need different Python packages and versions.

**Solution**: Virtual environments isolate each project's dependencies.

**Real-world analogy**: 
- Think of virtual environments like separate rooms in a house
- Each room (project) has its own furniture (packages)
- Room 1 might have Django 4.0, Room 2 might have Django 5.0
- They don't interfere with each other

### Step-by-Step Setup

#### Step 1: Check Python Installation

```bash
# Check if Python is installed
python --version
# Should show: Python 3.8.x or higher

# If not installed, download from python.org
```

**Why**: Django requires Python 3.8+. This command verifies you have the right version.

#### Step 2: Create Project Folder

```bash
# Create a folder for your project
mkdir drf_learning
cd drf_learning
```

**Why**: Organize your code in a dedicated folder. This keeps everything together.

#### Step 3: Create Virtual Environment

```bash
# Create virtual environment
python -m venv venv
```

**What this does**:
- `python -m venv` = Run the venv module
- `venv` = Name of the virtual environment folder
- Creates an isolated Python environment

**Why**: Isolates your project's packages from other projects.

#### Step 4: Activate Virtual Environment

**Windows**:
```bash
venv\Scripts\activate
```

**Linux/Mac**:
```bash
source venv/bin/activate
```

**What happens**:
- Your terminal prompt changes to show `(venv)`
- Python now uses packages from this virtual environment
- `pip install` will install packages here, not globally

**Why**: Tells your system to use this project's Python environment.

**How to know it worked**: Your prompt should look like:
```
(venv) C:\Users\YourName\drf_learning>
```

#### Step 5: Install Django and DRF

```bash
# Install packages
pip install django djangorestframework

# Verify installation
pip list
# Should show django and djangorestframework
```

**What this does**:
- `pip` = Python package installer
- `install` = Download and install packages
- `django` = Web framework
- `djangorestframework` = API toolkit for Django

**Why**: These are the tools we need to build APIs.

#### Step 6: Save Requirements

```bash
# Save installed packages to file
pip freeze > requirements.txt
```

**What this does**:
- `pip freeze` = List all installed packages with versions
- `>` = Save output to file
- `requirements.txt` = File containing package list

**Why**: 
- Others can install exact same packages: `pip install -r requirements.txt`
- Ensures everyone uses same versions
- Essential for deployment

## Virtual Environments - Why and How

### The Problem Virtual Environments Solve

**Without virtual environments**:
```
Project A needs Django 4.0
Project B needs Django 5.0
System Python: Which version? ❌ Conflict!
```

**With virtual environments**:
```
Project A (venv): Django 4.0 ✅
Project B (venv): Django 5.0 ✅
No conflict!
```

### Virtual Environment Commands

```bash
# Create virtual environment
python -m venv venv

# Activate (Windows)
venv\Scripts\activate

# Activate (Linux/Mac)
source venv/bin/activate

# Deactivate (all platforms)
deactivate

# Check if activated
# Prompt shows (venv) when active
```

**Important**: Always activate virtual environment before working on project!

### Common Mistakes

1. **Forgetting to activate**: Packages install globally instead of in venv
   - **Fix**: Always check for `(venv)` in prompt

2. **Activating wrong venv**: Working in different project's environment
   - **Fix**: Make sure you're in correct project folder

3. **Not using venv**: Installing packages globally
   - **Fix**: Always create and use virtual environment

## Understanding JSON

### What is JSON?

**JSON** = JavaScript Object Notation (but works with any language!)

**Why it matters**: APIs use JSON to send and receive data.

### JSON Structure

```json
{
  "id": 1,
  "title": "Python Guide",
  "author": "John Doe",
  "published": true,
  "tags": ["python", "programming", "tutorial"],
  "price": 29.99
}
```

**Explanation**:
- **Keys** (left side): Field names in quotes: `"title"`
- **Values** (right side): Data - can be string, number, boolean, array, object
- **Strings**: Always in quotes: `"Python Guide"`
- **Numbers**: No quotes: `1`, `29.99`
- **Booleans**: `true` or `false` (no quotes)
- **Arrays**: `["item1", "item2"]` (like Python lists)
- **Objects**: `{"key": "value"}` (like Python dictionaries)

### JSON vs Python Dictionary

**They're almost the same!**

```python
# Python Dictionary
book = {
    "id": 1,
    "title": "Python Guide",
    "published": True  # Note: True not true
}

# JSON (what APIs use)
{
    "id": 1,
    "title": "Python Guide",
    "published": true  # Note: lowercase true
}
```

**Key Differences**:
- Python: `True`/`False` (capital), `None` (not null)
- JSON: `true`/`false` (lowercase), `null` (not None)
- JSON keys must be strings, Python keys can be other types

### Converting Between Python and JSON

```python
import json

# Python dict to JSON string
book = {"id": 1, "title": "Python Guide"}
json_string = json.dumps(book)
# Result: '{"id": 1, "title": "Python Guide"}'

# JSON string to Python dict
json_string = '{"id": 1, "title": "Python Guide"}'
book = json.loads(json_string)
# Result: {"id": 1, "title": "Python Guide"}
```

**Why this matters**: 
- APIs send JSON strings over HTTP
- Django converts JSON to Python objects
- You work with Python, Django handles conversion

## HTTP Basics

### HTTP Methods (Verbs)

Think of HTTP methods as actions you can perform:

| Method | Action | Real-world Analogy | Example |
|--------|--------|-------------------|---------|
| **GET** | Read | Looking at a book | Get list of books |
| **POST** | Create | Adding a new book | Create a new book |
| **PUT** | Update (full) | Replacing entire book | Update all book fields |
| **PATCH** | Update (partial) | Updating one page | Update only book title |
| **DELETE** | Delete | Removing a book | Delete a book |

**Why different methods?**
- **GET**: Safe, doesn't change data (can call many times)
- **POST**: Creates new resource (not safe to call multiple times)
- **PUT/PATCH**: Updates existing resource
- **DELETE**: Removes resource (not safe to call multiple times)

### HTTP Status Codes

**What they mean**: Tell client if request succeeded or failed.

| Code | Meaning | When Used |
|------|---------|-----------|
| **200** | OK | Request succeeded |
| **201** | Created | New resource created (POST) |
| **204** | No Content | Success but no data to return (DELETE) |
| **400** | Bad Request | Client sent invalid data |
| **401** | Unauthorized | Not logged in / invalid credentials |
| **403** | Forbidden | Logged in but no permission |
| **404** | Not Found | Resource doesn't exist |
| **500** | Server Error | Server had an error |

**Why this matters**: APIs use these codes to communicate results. Your frontend checks these codes to know what happened.

### HTTP Request Structure

```
GET /api/books/ HTTP/1.1
Host: localhost:8000
Authorization: Bearer token123
Content-Type: application/json
```

**Explanation**:
- **GET** = HTTP method
- **/api/books/** = Endpoint (URL path)
- **HTTP/1.1** = Protocol version
- **Headers** = Additional information (authorization, content type)

### HTTP Response Structure

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 1,
  "title": "Python Guide"
}
```

**Explanation**:
- **200 OK** = Status code and message
- **Content-Type** = Format of response data
- **Body** = Actual data (JSON)

## Command Line Essentials

### Why Command Line?

**GUI vs CLI**:
- **GUI** (Graphical User Interface): Clicking buttons, visual
- **CLI** (Command Line Interface): Typing commands, faster for developers

**Why developers use CLI**:
- Faster for repetitive tasks
- Can be automated
- Works on servers (no GUI)
- More powerful

### Essential Commands

#### Navigation

```bash
# See current directory
pwd                    # Linux/Mac
cd                     # Windows

# List files
ls                     # Linux/Mac
dir                    # Windows

# Change directory
cd folder_name         # Go into folder
cd ..                  # Go up one level
cd ~                   # Go to home directory
```

#### File Operations

```bash
# Create folder
mkdir project_name

# Create file
touch file.txt         # Linux/Mac
type nul > file.txt    # Windows

# View file
cat file.txt           # Linux/Mac
type file.txt          # Windows

# Delete file
rm file.txt            # Linux/Mac
del file.txt           # Windows
```

#### Python-Specific

```bash
# Run Python script
python script.py

# Run Django commands
python manage.py runserver
python manage.py migrate
python manage.py createsuperuser

# Install package
pip install package_name

# List installed packages
pip list
```

**Why these matter**: You'll use these commands constantly when building APIs!

## Practice Problems

### Problem 1: Python Basics

Create a Python script that:
1. Defines a dictionary representing a book with: id, title, author, price
2. Creates a function that takes book data and returns a formatted string
3. Creates a list of 3 books
4. Prints information about each book

**Solution**:
```python
def format_book(book):
    return f"{book['title']} by {book['author']} - ${book['price']}"

book1 = {"id": 1, "title": "Python Guide", "author": "John", "price": 29.99}
book2 = {"id": 2, "title": "Django Guide", "author": "Jane", "price": 39.99}
book3 = {"id": 3, "title": "DRF Guide", "author": "Bob", "price": 49.99}

books = [book1, book2, book3]

for book in books:
    print(format_book(book))
```

### Problem 2: JSON Practice

1. Create a Python dictionary with book data
2. Convert it to JSON string
3. Convert JSON string back to dictionary
4. Print the result

**Solution**:
```python
import json

book = {
    "id": 1,
    "title": "Python Guide",
    "author": "John Doe",
    "price": 29.99,
    "published": True
}

# Convert to JSON
json_string = json.dumps(book)
print("JSON:", json_string)

# Convert back to dict
book_dict = json.loads(json_string)
print("Dictionary:", book_dict)
```

### Problem 3: Virtual Environment

1. Create a new folder called `my_api_project`
2. Create a virtual environment named `venv`
3. Activate it
4. Install Django
5. Verify installation with `pip list`

**Solution**:
```bash
mkdir my_api_project
cd my_api_project
python -m venv venv

# Windows
venv\Scripts\activate

# Linux/Mac
source venv/bin/activate

pip install django
pip list  # Should show django
```

## Trivia

### Question 1
**What does API stand for?**
- [ ] Application Program Interface
- [x] Application Programming Interface
- [ ] Automated Program Interface
- [ ] Advanced Programming Interface

**Explanation**: API = Application Programming Interface. It's the way applications communicate with each other.

### Question 2
**What is the main purpose of a virtual environment?**
- [ ] To make Python run faster
- [x] To isolate project dependencies
- [ ] To secure your code
- [ ] To organize your files

**Explanation**: Virtual environments isolate packages for each project, preventing conflicts between different projects' requirements.

### Question 3
**Which HTTP method is used to retrieve data?**
- [x] GET
- [ ] POST
- [ ] PUT
- [ ] DELETE

**Explanation**: GET is used to retrieve/read data. It's safe and doesn't modify data.

### Question 4
**What does JSON stand for?**
- [ ] Java Script Object Notation
- [x] JavaScript Object Notation
- [ ] Just Simple Object Notation
- [ ] Java Standard Object Notation

**Explanation**: JSON = JavaScript Object Notation, though it's used with many languages, not just JavaScript.

### Question 5
**In JSON, how do you represent a boolean true value?**
- [ ] True
- [x] true
- [ ] TRUE
- [ ] "true"

**Explanation**: JSON uses lowercase `true` and `false` (not Python's `True`/`False`).

### Question 6
**What status code means "Resource created successfully"?**
- [ ] 200
- [x] 201
- [ ] 204
- [ ] 301

**Explanation**: 201 Created is returned when a new resource is successfully created (typically with POST requests).

### Question 7
**What command activates a virtual environment on Windows?**
- [ ] source venv/bin/activate
- [x] venv\Scripts\activate
- [ ] activate venv
- [ ] python venv activate

**Explanation**: On Windows, use `venv\Scripts\activate`. On Linux/Mac, use `source venv/bin/activate`.

### Question 8
**Which Python data type is most similar to JSON objects?**
- [ ] List
- [ ] String
- [x] Dictionary
- [ ] Tuple

**Explanation**: Python dictionaries map directly to JSON objects. They have the same key-value structure.

## Next Steps

Congratulations! You've completed Level 0. You now understand:

- ✅ Python basics for web development
- ✅ How the web works
- ✅ What APIs are and why we use them
- ✅ How to set up your development environment
- ✅ JSON format and HTTP basics
- ✅ Command line essentials

**Ready for Level 1?** Continue to [Level 1: Foundations](LEVEL_1_FOUNDATIONS.md) to start building your first REST API!

---

**Remember**: 
- Don't rush - understand the concepts
- Practice the exercises
- Experiment with the code
- Ask questions when stuck

Good luck! 🚀

