# Flask Blog tutorial rewritten with Hexagonal Architecture

[GitHub Repo of this project](https://github.com/ShahriyarR/hexagonal-flask-blog-tutorial)

The idea is to rewrite the official [Flask blog tutorial](https://flask.palletsprojects.com/en/2.2.x/tutorial/) with the help of Dependency Injection and following the Hexagonal Architecture.

You can diff and look at original code base from here: [Flask blog tutorial code base](https://github.com/pallets/flask/tree/main/examples/tutorial/flaskr)

All statics, templates were grabbed from original blog post.

This project should give an idea how we can actually implement the Ports and Adapters(Hexagonal) architecture using Python, Flask and DI.

The final project structure looks like:

```sh
src
├── adapters
│   ├── app
│   │   ├── application.py
│   │   ├── blueprints
│   │   │   ├── auth.py
│   │   │   ├── blog.py
│   │   │   └── __init__.py
│   │   ├── __init__.py
│   │   ├── schema.sql
│   │   ├── static
│   │   │   └── style.css
│   │   └── templates
│   │       ├── auth
│   │       │   ├── login.html
│   │       │   └── register.html
│   │       ├── base.html
│   │       └── post
│   │           ├── create.html
│   │           ├── index.html
│   │           └── update.html
│   └── db
│       ├── __init__.py
│       ├── post_repository.py
│       └── user_repository.py
├── domain
│   ├── model
│   │   ├── __init__.py
│   │   ├── post.py
│   │   └── user.py
│   └── ports
│       ├── __init__.py
│       ├── post_service.py
│       ├── repository.py
│       └── user_service.py
└── main
    ├── config.py
    ├── containers.py
    ├── post_containers.py
    └── user_containers.py

12 directories, 27 files
```

### About Hexagonal Architecture

You can read it from original author:

[The Pattern: Ports and Adapters](https://alistair.cockburn.us/hexagonal-architecture/)


### About project dependencies

The project has 2 main dependencies:

[Dependency Injector](https://github.com/ets-labs/python-dependency-injector)

[Flask](https://github.com/pallets/flask)


### Getting started

Now let's dig into what we have and how we can convert this original flask blog tutorial code into the clean, maintainable hexagonal architecture.

Conceptually we need to separate our domain model from outer world by centering it in the heart of our application.
That means we need to first think about our domain models. Again not about database, not about flask, not about docker etc, think about what is going to be the heart of your application.

The best is to ask questions:

* Q: What a heck is this app?

  - As for the official Flask tutorial this is a blog app, and it needs 2 things to work: user to login and blog to be written.

* Q: Who is the heck this user? What makes "something" a user?

  - "something" with unique ID, username and password can be considered as a User for us.

* Q: What is a blog post then?

  - Blog post is going to have author_id, title, body and maybe the created time. "Something" with that information would be considered as blog post.

Cool we have defined our models, it is time to implement.

> Please keep in mind, the domain models are not the database models, we do not even think about database. Database is a detail.

### Implement domain models

Our domain models is going to rest at: `/src/domain/model` path. So please create those folders.

Then create `user.py` and `post.py` which is going to represent User and Post respectively.

```py
from uuid import uuid4
from dataclasses import dataclass


@dataclass
class User:
    id_: str
    user_name: str
    password: str


def user_factory(user_name: str, password: str) -> User:
    # data validation should happen here
    return User(id_=str(uuid4()), user_name=user_name, password=password)
```

Notice the factory method for creating User model and returning it back. Ideally, there should be a great validation step in this factory method, but we have omitted it as the original Flask tutorial also blinds it silently :)

Our Post model:

```py
from datetime import datetime
from uuid import uuid4
from dataclasses import dataclass


@dataclass
class Post:
    id_: str
    author_id: int
    title: str
    body: str
    created: datetime


def post_factory(author_id: int, title: str, body: str, created: datetime = None) -> Post:
    # data validation should happen here
    _created = created or datetime.now()
    return Post(id_=str(uuid4()), author_id=author_id, title=title, body=body, created=_created)
```

Quite simple, quite neat.

### Reasoning about ports.

As we are going to follow Ports and Adapters architecture, we need to think about Ports first a this is the place is getting to be implemented and used say by REST API.

I put ports code inside the domain ports folder `/src/domain/ports`.

> Ports are interfaces to be implemented by adapters, which you will see in a moment.

Questions:

* Q: How we are going to persist our user and post data?

  - Okay, we can have simple Repository pattern, which is an interface or abstract class accepting database connection.

Create `repository.py` file inside the ports and put the following code:

```py
from sqlite3 import Connection
from abc import ABC, abstractmethod
from typing import Any, Callable


class RepositoryInterface(ABC):

    @abstractmethod
    def __init__(self, db_conn: Callable[[], Connection]) -> None:
        self.db = db_conn

    @abstractmethod
    def execute(self, query: str, data: tuple[Any, ...], commit: bool = False) -> Any:
        ...

    @abstractmethod
    def commit(self) -> None:
        ...
```

Basically, our Repo is going to execute the SQL statements and then call the db commit.
We are not using any kind of ORM here, as the original Flask Blog tutorial also only uses raw SQLs with SQlite.

* Q: How the outside world(CLI, REST, anything) is going to register and login the User?

  - We can have the simplest ever Command or Service pattern, let's just call it `user_service.py`:

```py
from typing import Optional, Any
from . import RegisterUserInputDto
from ..model import user_factory, User
from .repository import RepositoryInterface


class UserDBOperationError(Exception):
    ...


class UserService:

    def __init__(self, user_repo: RepositoryInterface) -> None:
        self.user_repo = user_repo

    def create(self, user: RegisterUserInputDto) -> User:
        user = user_factory(user_name=user.user_name, password=user.password)
        data = (user.id_, user.user_name, user.password)
        query = "INSERT INTO user (id_, username, password) VALUES (?, ?, ?)"
        try:
            self.user_repo.execute(query, data, commit=True)
        except Exception as err:
            raise UserDBOperationError(err) from err
        return user

    def get_user_by_user_name(self, user_name: str) -> Optional[tuple[Any, ...]]:
        data = (user_name, )
        query = "SELECT * FROM user WHERE username = ?"
        try:
            return self.user_repo.execute(query, data).fetchone()
        except Exception as err:
            raise UserDBOperationError() from err

    def get_user_by_id(self, id_: int) -> Optional[tuple[Any, ...]]:
        data = (id_,)
        query = "SELECT * FROM user WHERE id = ?"
        try:
            return self.user_repo.execute(query, data).fetchone()
        except Exception as err:
            raise UserDBOperationError() from err

```

Now please, notice that the `UserService` accepts the `RepositoryInterface` as an argument, as a dependency.

Using the `user_repo` it is going to persist the actual User data. 

Also notice how the user model was created using user_factory inside the create method and only after that the user data get constructed to be inserted into the database.

This is how we invert the dependency of storage mechanism as well, our User model do not depend on the database to survive, but the database action requires our User to do its work.

* How about managing Posts? Here you go with, `post_service.py`:

```py
from typing import Optional, Any
from flask import abort, g
from . import CreatePostInputDto, UpdatePostInputDto, DeletePostInputDto
from src.domain.ports.repository import RepositoryInterface
from src.domain.model.post import Post, post_factory


class BlogDBOperationError(Exception):
    ...


class PostService:

    def __init__(self, post_repo: RepositoryInterface) -> None:
        self.post_repo = post_repo

    def create(self, post: CreatePostInputDto) -> Optional[Post]:
        _post = post_factory(author_id=post.author_id, title=post.title, body=post.body)
        data = (_post.id_, _post.title, _post.body, _post.author_id)
        query = "INSERT INTO post (id_, title, body, author_id) VALUES (?, ?, ?, ?)"
        try:
            return self.post_repo.execute(query, data, commit=True)
        except Exception as err:
            raise BlogDBOperationError(err) from err

    def update(self, post: UpdatePostInputDto):
        data = (post.title, post.body, post.id)
        query = """UPDATE post SET title = ?, body = ? WHERE id = ?"""
        try:
            return self.post_repo.execute(query, data, commit=True)
        except Exception as err:
            raise BlogDBOperationError(err) from err

    def delete(self, post: DeletePostInputDto):
        data = (post.id, )
        query = "DELETE FROM post WHERE id = ?"
        try:
            return self.post_repo.execute(query, data, commit=True)
        except Exception as err:
            raise BlogDBOperationError(err) from err

    def get_all_blogs(self) -> Optional[list[Any]]:
        data = ()
        query = """SELECT p.id, title, body, created, author_id, username
         FROM post p JOIN user u ON p.author_id = u.id
         ORDER BY created DESC"""
        try:
            return self.post_repo.execute(query, data).fetchall()
        except Exception as err:
            raise BlogDBOperationError() from err

    def get_post_by_id(self, _id: int, check_author: bool = True) -> Post:
        data = (_id,)
        query = """SELECT p.id, title, body, created, author_id, username
             FROM post p JOIN user u ON p.author_id = u.id
            WHERE p.id = ?"""

        post = self.post_repo.execute(query, data).fetchone()

        if post is None:
            abort(404, f"Post id {id} doesn't exist.")

        if check_author and post['author_id'] != g.user['id']:
            abort(403)

        return post
```

PostService is accepting again the abstract repository for persisting the post data.
But we have more interesting things here.

Let me explain more about the idea behind `CreatePostInputDto`, `UpdatePostInputDto`, `DeletePostInputDto`.

Again the questions:

* What kind of information we need to create Post domain model?

  - Basically, post_factory() requires author_id, body and title, created time will be automatically instantiated.
`CreatePostInputDto`(data transfer object) simply encapsulates and validates this information, 
using its data the post_factory creates Post, and then it is inserted into the database.

* What kind of information we need to update the saved post in the database?

  - `UpdatePostInputDto` encapsulates and again validates post id, title and body

* What kind of information we need to delete the saved post from the database?

  - We can find the post by id and then remove. `DeletePostInputDto` simply encapsulates the id for us.

But why, we need those DTOs? 
If you accept something from outside as an Input, and if you return something back as Output, 
it is always better to restrict, validate and also give some shape to the accepted and returned data.
Think about it as Request/Response model in web frameworks. You accept some data as part of request object, then you construct some Response model and return it to the user.

I am going to put these DTOs in the `/src/domain/ports/__init__.py`:

```py
from dataclasses import dataclass, asdict
from werkzeug.security import generate_password_hash

@dataclass
class RegisterUserInputDto:
    user_name: str
    password: str

    def to_dict(self):
        return asdict(self)

    @classmethod
    def from_dict(cls, dict_):
        return cls(**dict_)


def register_user_factory(user_name: str, password: str) -> RegisterUserInputDto:
    return RegisterUserInputDto(user_name=user_name, password=generate_password_hash(password))


@dataclass
class CreatePostInputDto:
    title: str
    body: str
    author_id: int

    def to_dict(self):
        return asdict(self)

    @classmethod
    def from_dict(cls, dict_):
        return cls(**dict_)


def create_post_factory(title: str, body: str, author_id: int) -> CreatePostInputDto:
    return CreatePostInputDto(title=title, body=body, author_id=author_id)


@dataclass
class UpdatePostInputDto:
    id: int
    title: str
    body: str

    def to_dict(self):
        return asdict(self)

    @classmethod
    def from_dict(cls, dict_):
        return cls(**dict_)


def update_post_factory(id: int, title: str, body: str) -> UpdatePostInputDto:
    return UpdatePostInputDto(id=id, title=title, body=body)


@dataclass
class DeletePostInputDto:
    id: int

    def to_dict(self):
        return asdict(self)

    @classmethod
    def from_dict(cls, dict_):
        return cls(**dict_)


def delete_post_factory(id: int) -> DeletePostInputDto:
    return DeletePostInputDto(id=id)
```


In conclusion of the first part:

* We have implemented the simple domain model, without any domain events or other fancy things.
* We have implemented all ports of the Hexagonal Architecture.

> We have no tests as well, because original Flask Blog tutorial brutally kicked the tests out :D



