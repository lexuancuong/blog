# Django-Ninja: Is DRF over?
A new framework for building APIs with Python and Django could beat DRF.

## 1. Overview

Creating Rest API applications has become a frequent task in the software industry. Thankfully, there are many frameworks and libraries available to support their implementation.

[Django](https://github.com/django/django) is the oldest web framework in the Python world, having been around since 2005. It offers rapid development and a clean and pragmatic design.

In development, to avoid writing boilerplate code and speed up the process of building Rest API applications with Django, the most popular tool is Django Rest Framework.

While it is a well-established tool and has been utilized by many companies and individuals, there are some drawbacks to using DRF. One alternative option is Django Ninja, which was created to address these pain points.

## 2. Django Ninja Introduction

### Motivation

[Django-Ninja](https://django-ninja.rest-framework.com/) works almost like FastAPI. FastAPI is loved because of its modern(far from boilerplate code), explicit API function interface, and excellent documentation.

FastAPI doesn't support ORM by default; developers need to coordinate FastAPI with query tools like Sqlalchemy or Django ORM.

But Django is built with a lot of awesome features, so Django-Ninja is tried to bring the most loved features from FastAPI to Django.

### Philosophy

Like FastAPI, Django Ninja has been designated forward functional programming.

Django-Ninja's API functions are expected to be declared with pure function with the specified interface and its core logic.

### CRUD API application

Import needed modules from Django-ninja and the declared Book models

```python
from datetime import date
from typing import List

from django.shortcuts import get_object_or_404
from ninja import NinjaAPI, Schema

from books.models import Book
```

Initialize the Ninja API Application and request/response schemas.

```python
api = NinjaAPI()

class BookIn(Schema):
    name: str
    author: str

class BookOut(Schema):
    id: int
    name: str
    author: str
    created_date: datetime
```

Define CRUD API function in functional programming styles

```python
@api.post("/books/", response=BookOut)
def create_book(request, payload: BookIn):
    book = Book.objects.create(**payload.dict())
    return {"id": book.id}

@api.get("/books/{book_id}", response=BookOut)
def get_book(request, book_id: int):
    book = get_object_or_404(book, id=book_id)
    return book

@api.get("/books", response=List[bookOut])
def list_books(request):
    return book.objects.all()

@api.put("/books/{book_id}", response=BookOut)
def update_book(request, book_id: int, payload: bookIn):
    book = get_object_or_404(book, id=book_id)
    for attr, value in payload.dict().items():
        setattr(book, attr, value)
    book.save()
    return book

@api.delete("/books/{book_id}")
def delete_book(request, book_id: int):
    book = get_object_or_404(book, id=book_id)
    book.delete()
    return {"success": True}
```

From the sample code, we could recognize the simplicity without the magic implementation of Django Ninja.

All request and response class was defined concretely and specified on every endpoint.

They are automatically visualized on the Open Swagger UI of this web application.

We will dive into all the outstanding points of Django Ninja in the next part.

## 3. Django-Ninja vs. DRF
### Performance
Django-ninja's performance is significantly higher than DRF (parsing, validating, exporting,...) based on Pydantic powerful.

There is a benchmark for Django, DRF, and Flask+Marshmallow. With this API endpoint:
For DRF:
```python
@api_view(['POST'])
def create(request):
    data = Model(data=json.loads(request.body))
    assert data.is_valid()
    return Response({'success': True})
```

For Django Ninja:
```python
@api.post("/create")
def create(request, model: Model):
    return {"success": True}
```

For Flask + Marshmallow:
```python
@app.route("/api/create", methods=["POST"])
def create():
    json_data = request.get_json()
    data = model_schema.load(json_data)
    return {"success": True}, 201
```

(Environment: The same WSGI with one currency and Hardware)

Eventually, Django-ninja is faster 2 times than DRF at serializing tasks.
![benchmark](/blog/images/django_ninja/benchmarks.png)

### Async support

If you're utilizing ASGI, you should be aware that Django 3.0 and above provides support for the development of asynchronous ("async") views and an async-capable request stack.

Nonetheless, it is not currently feasible to use `async` for I/O bound tasks in DRF 3.14-. This topic has been debated (https://github.com/encode/django-rest-framework/issues/7260), and for the time being, the application of async to DRF 3.14- is not possible.

Django-ninja supports async features from first releases, and it is a well-known feature of this library now.

### Syntax

Django-ninja defines schemas as similar to Pydantic or FastAPI.

So, like Pydantic, schema classes follow the [PEP 484](https://peps.python.org/pep-0484/): `variable_name: type`.

It is straightforward and could be ensured by almost linter/code diagnostic (i.e., mypy, copyright,...) without installing an extra library for type checking.
For example, define schema classes in Django-ninja is:

```python
class UserRequest(pydantic.BaseModel):
  name: str
```

For DRF, we need to install the [drf-stubs](https://github.com/typeddjango/djangorestframework-stubs) package to support linter tools to verify type hints.

We need to learn about the DRF convention to define DRF's classes.

For example:
```python
class UserRequest(drf_serializers.Serializer):
  name = serializers.CharField()
```

This explicit feature is the most attractive one for me.

### Explicity

It has mandatory type hints for function arguments, making the API specification straightforward to write, read and maintain.

```python
@api.post("/books/", response=BookOut)
def create_book(request, payload: BookIn):
    book = Book.objects.create(**payload.dict())
    return {"id": book.id}
```

Relying on this explicit declaration, Django-ninja could generate precisely Swagger API documentation.

### Boilerplate
We don't need to write these extra lines to validate the payloads when using Django-ninja:
```python
    input_serializer = ShipmentIdentifierRequestSerializer(data=request.query_params)
    input_serializer.is_valid(raise_exception=True)
    validated_data = input_serializer.validated_data # A wild dictionary object
```
These DRF's boilerplate code makes the application so complicated.
This will help us have more concise code. In typical cases, the more line of code we have, the more risks we must face. (obey DRY principles)

### API Documentation

Django-ninja supports automatically generating interactive Swagger API documentation from the application Python functions for API endpoints.

![Swagger API documentation generated by django-ninja](/blog/images/django_ninja/swagger-api.png)

With DRF, we need to install an external library [drf-yasg](https://github.com/axnsan12/drf-yasg) to generate the Swagger API docs.

Due to implicit code, `drf-yasg` has a problem in addressing the schema of POST request payload. `drf-yasg` cannot identify the request payload schema because it is hidden from the API interface.

```python
@api_view(['POST'])
def create(request):
    data = Model(data=json.loads(request.body)) # the request schema is specified behind the API interface.
    data.is_valid(raise_exception=True)
    validated_data = data.validated_data
```

### Library Documentation

DRF is older than Django-ninja (Django-ninja is just 3 years old).
DRF has a lot of documentation, blog posts, answers on StackOverflow, etc.

But Django-ninja is minimalist, and the documentation is easy to learn.

(It took me around 40 minutes to read all of the Django-ninja documentation. For DRF, it took me a lot of days to go through all of its features (these things that I haven't used at this moment))

## 4. Usage

### Project Structure

You could follow my fine-tuned project structure to build up an API application. It is pretty explicit and follows common architectural principles.

```text
  > migrations          # Contain migration files to migrate ORM classes to DB
  > models              # Define ORM classes
    > book.py           # Define ORM classes for `book` table
  > views               # Contain API functions with associated endpoint and interface
    > v1                # Everything are related to API version `v1`
      > endpoints       # Contain all python files with utilizing v1's requests and responses
        > book.py       # Define endpoints related to `book` table
        > mock.py       # Mock endpoints for testing purpose - for example
      > requests        # Define the expected requests payload structure of endpoints
        > book.py       # Request payload structure of `book` table
      > responses       # Explicitly define the response schema that will be returned
        > book.py       # Response schemas of endpoints related to `book` table
    > v2                # Everything are related to API version `v1`
      > endpoints
      > requests
      > responses
    urls.py
```

### Tips

1. The order of installed apps decides the correctness of the application.
The main app should be standing before other Django default installed apps.

```py
# settings.py
INSTALLED_APPS = [
    'api',
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.messages',
    'django.contrib.sessions',
    'django.contrib.staticfiles',
]
```

### Issues
There are 2 existing issues of Django-ninja when compared with DRF:

1. The latest stable version doesn't support some of Django decorators.

```python
from django.views.decorators.cache import cache_page
from ninja.decorators import decorate_view

@api.get("/some")
@decorate_view(cache_page(60 * 15))  # <---- Cannot caching this API.
def some_operation(request):
     return {"hello": "World"}
```

2. Lack of the `context` feature.

This issue is mainly from Pydantic. With version under 2.0, Pydantic doesn't support passing context into the class model.
Pydantic 2.0+ supported these features and Django-ninja is moving forward this Pydantic version with Django-ninja v1.0.

## 5. Conclusion
Django-ninja offers numerous superior features when compared to DRF.

However, migrating from DRF to Django-ninja requires time, effort, and a willingness to embrace this innovative change.
