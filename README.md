# Networks Lab 2 - REST API

In this lab, you will get to put into practice the semantics of REST (Representational State Transfer) by implementing a REST over HTTP API.

If you done backend programming before and want to challenge yourself, you may skip to the Checkoff Requirements. We will be using the FastAPI framework, which is very well documented, very easy to learn and has many tutorials online. You are free to search tutorials online instead of following this guide.

## Setup

In the guided lab, we will create a REST-over-HTTP API that imitates a student registry service: it can create, list (read), update and delete students. In REST terminology, _students_ are our resource that we manipulate over HTTP.

Make sure you have [docker](https://docs.docker.com/get-docker/) and [docker-compose](https://docs.docker.com/compose/install/) installed. Pycharm is the suggested editor, but you can use any editor you're comfortable with.

Run `docker-compose up` and make sure you get "Hello world" by going to [http://127.0.0.1:8000](http://127.0.0.1:8000).

## Walkthrough

### Part 1 - Simple GET

Look at `app/main.py`. In FastAPI, routes in an API are defined by a function annotated with `@app.verb('route')`. `app`
refers to the special FastAPI object that we instantiated. When the docker container runs, it will import the `main.py`
file and look for a variable named `app` by default.

Let's try to create another route:

```python3
@app.get('/students')
def get_all_students():
    return "Something"
```

Now let's try to make a GET request in your browser. Open [http://127.0.0.1:8000/students](http://127.0.0.1:8000/students) and you should see "Something" in your browser.

As you can tell, inside a route function, whatever you `return` in the Python function will be wrapped into plaintext HTTP response to the client (your web browser). Can we try to return something that is not a string?

```python3
@app.get('/students')
def get_all_students():
    return ["Alice", "Bob", "Charlie"]
```

There is no restriction on the data you send in a HTTP Response in a REST API. For example, you may choose to format your data as an XML string:

```python3
@app.get('/students')
def get_all_students():
    return """
    <students>
        <student>Alice</student>
        <student>Bob</student>
        <student>Charlie</student>
    </students>
    """
```

(You can probably do it better using XML libraries in Python instead of hardcoding it like this)

In a similar vein, you can return text that can be parsed as HTML, or even binary files for multimedia like images and videos.

[JSON (Javascript Object Notation)](https://en.wikipedia.org/wiki/JSON) is essentially the most common format that most REST APIs accept and return data. When you return a dictionary in the route function, FastAPI will try its best to convert it to a JSON string.

Let's try to return data that is JSON-compliant:

```python3
@app.get("/students")
def get_all_students():
    return {
        "students": [
            {
                "name": "Alice",
                "id": "1004803",
                "gpa": 4.0
            },
            {
                "name": "Bob",
                "id": "1004529",
                "gpa": 3.6
            },
            {
                "name": "Charlie",
                "id": "1004910",
                "gpa": 5.0
            }
        ]
    }
```

### Part 2 - Path Parameters

[Official FastAPI Docs on Path Parameters](https://fastapi.tiangolo.com/tutorial/path-params/)

Ok, we have a route that returns all the students. What if we want to retrieve data on a single student?

We first need to decide how to uniquely identify the resource (student), or the so called "primary key" of the resource. This primary key has a strict definition that no two resources can have the same value for its primary key. In our case, lets use the 7-digit student ID as the primary key.

In REST conventions, we should be able to read data on a single student using the following HTTP request:

```
GET http://127.0.0.1:8000/students/1004910 HTTP/1.1


```

In FastAPI, we can specify _route parameters_, that can automatically be replaced with whatever that is inside that segment of the path.

But before that, lets move our "students database" to a global variable so that it can be read by our old function and the new function that we're about to add.

```python3
students = [
    {
        "name": "Alice",
        "id": "1004803",
    },
    {
        "name": "Bob",
        "id": "1004529"
    },
    {
        "name": "Charlie",
        "id": "1004910",
        "gpa": 5.0
    }
]
```

```python3
@app.get("/students/{student_id}")
def find_student(student_id: str):
    global students
    for student in students:
        if student["id"] == student_id:
            return student
    return None
```

You'll notice that we return `None` if we cannot find a student with a matching ID. We can use HTTP status codes to indicate tell the client that some errors have occurred. For example, 404 is usually meant for not found.

To use response codes in FastAPI, you need to first declare the `response` parameter, then edit its `status_code` attribute.

```python3
from fastapi import FastAPI, Response


@app.get("/students/{student_id}")
def find_student(student_id: str, response: Response):
    global students
    for student in students:
        if student["id"] == student_id:
            return student
    response.status_code = 404
    return None
```

### Part 3 - GET with query parameters

[Official FastAPI Docs on Query Parameters](https://fastapi.tiangolo.com/tutorial/path-params/)

In REST convention, the body of GET requests is usually empty. To customise GET queries for your REST API such as sorting and filtering, we usually encode query parameters in the URL, like so:

```
GET http://127.0.0.1:8000/students&sortBy=id&limit5 HTTP/1.1
```

By REST convention, the above request means to retrieve the first 5 students sorted by id. In FastAPI, any extra parameters inside your route function that do not appear as route parameters will automatically be assumed to be query parameters with a matching name.

```python3
from typing import Optional


@app.get('/students')
def get_students(sortBy: Optional[str] = None, limit: Optional[int] = None):
    # TODO: if parameters are not None, sort & limit...
    ...
```

You can then check the value of the parameters to see if any query parameters are set and modify your response accordingly.

### Part 4 - POST w/ Request Models

[Official FastAPI docs on request bodies](https://fastapi.tiangolo.com/tutorial/body/)

Next, lets try giving our API the ability to modify the list of students. According to REST conventions, the HTTP request to create a new student would be:

```
POST http://127.0.0.1/students HTTP/1.1
Content-Type: application/json

{
    "name": "Daniel",
    "id": "1004231"
}
```

We will send the full details of the student to be created in a single request, encoded in the body as a JSON string. For FastAPI to read the body, we must first create a class that describes the expected fields and types in the body. We inherit from pydantic's `BaseModel` class:

```python3
from pydantic import BaseModel


class Student(BaseModel):
    name: str
    id: str
```

Then, to simply have FastAPI convert the request body into the instance of the class, we need to add a parameter to our request route and typehint it accordingly:

```python3
@app.post("/students")
def create_student(student: Student):
    ...  # do something with the student param
```

Note that you will need an external tool such as Postman to fire the request, web browsers fetch pages with GET only (without writing your own Javascript). You can also use command line tools like wget. Alternatively, you can write `.http` files that are recognised by both Jetbrains and Visual Studio Code as HTTP requests and you can "run" the files to fire a HTTP request.

### Part 5 - Persistence and Dependency Injection

[Official FastAPI docs on dependency injection](https://fastapi.tiangolo.com/tutorial/dependencies/?h=depen#what-is-dependency-injection)

In the earlier part, you might have added the student to the global student variable like this:

```python3
@app.post("/students")
def create_student(student: Student):
    global students
    students.append(student)
```

This introduces some problems. Web servers are multi-threaded: a web server will run a few threads in parallel so that it can answer multiple requests at once. As you have learnt in CSE and ESC, mutating global variables in multithreaded environments can cause visibility issues. What means in this context is that when one client has POST-ed a new student, when another client fires `GET /students`, he may be retrieving the old list of students. In addition, when the Python program shuts down, all data is lost.

To solve this issue pretty much every web service out there decouples their HTTP application layer and the database persistence layer. The data resources are not stored in Python variables, but live in database software that are built to store data in a structured or unstructured manner. There are lots of database software available, and for this lab we will use Redis, a very straightforward key-value store.

This part of the lab, and the rest of the lab, is intentionally left open ended. You are free to use any other database of your choice, modifying the `docker-compose.yaml` file to add new containers and `requirements.txt` file to add new Python libraries to interface with the database of your choice.

Even if you use redis, there are many ways you can store the data: do you want to chuck everything in a single key? Do you want to save each student into their own keys by student id, then have a separate key called `student_ids`? This is left as an implementation detail and a thinking exercise for you, you will not be penalised if you choose one way or another, as long as the HTTP clients can see their changes properly.

In any case, you will need to know how to use [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) to give your routing functions access to your database client.

> DI/IoC, as well as multithreading and visibility issues, are very important Software Engineering concepts you must be aware of and comfortable with as a Term 6 ISTD student!

#### Creating the redis (or whatever database client)

First you need to write the function to produce a new instance of the database client. For the case of redis, using the [`redis-py` package](https://github.com/andymccurdy/redis-py):

```python3
import redis


def get_redis_client():
    return redis.Redis(host="redis")
```

Note that the host of the redis container is just `redis`, we have defined it inside `docker-compose.yaml`.

#### Declare the dependency in the dependent

In dependency injection, the "dependant" declares that they need a certain "dependency". In our case, the "dependant" is our routing function, and the "dependency" is the redis/database client.

In FastAPI, declaring dependencies is really simple:

```python3
from fastapi import FastAPI, Depends


@app.post("/students")
def create_student(student: Student, redis_client: redis.Redis = Depends(get_redis_client)):
    # do something with your redis_client, like saving data to a key
    ...
```

Interfacing with redis is a implementation detail and left for you to figure out on your own. Remember to update the `GET` methods to read from redis as well!

## Checkoff Requirements

- A REST-over-HTTP API written in any programming language, in any framework, to imitate any real life service (e.g. fake myportal, fake edimension), backed with any other supporting services (redis, mysql, etc):
    - Can be deployed on any docker host using `docker compose` - you fail if I need to install any other dependencies on my computer!
- With accompanying `.http` unit test files, showcasing your API's ability to respond to:
    - a GET request ...
        - with no query parameters
        - with a `sortBy` query parameter, to transform the order of the items returned
            - up to you to decide what attributes can be sorted
        - with a `count` query parameter, to limit the number of items returned
        - with a `offset` query parameter, to "skip" ahead by a number of items
        - with any combination of the above query parameters
    - a POST request ...
        - that creates a new resource with the given attributes in the body
        - show that the resource has indeed been created through another HTTP request
        - has validation and returns an appropriate HTTP response code if the input data is invalid (e.g. missing name)
    - either a DELETE or PUT request...
        - that deletes or updates a _single_ resource respectively
        - show that the resource has indeed been modified through another HTTP request = has validation and returns an appropiate HTTP response code if the input data is invalid (e.g. trying to delete a nonexistent user)
- Identify which routes in your application are _idempotent_, and provide proof to support your answer.
- Implement at least two of the following challenges:
    - File upload in a POST request, using multipart/form-data
    - Have a route in your application that returns a content type that is not _plaintext_
    - Some form of authorization through inspecting the request headers
    - A special route that can perform a batch delete or update of resources matching a certain condition

> You must provide ample documentation on how to build & run your code and how to make the HTTP requests to your API, as well as what are the expected responses for each request. You will not be deducted if your program is slow or unoptimised, but bonus points may be given if you show meaningful thought in analysing how performance was / can be improved in your application.