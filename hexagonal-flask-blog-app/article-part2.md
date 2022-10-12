# Flask Blog tutorial rewritten with Hexagonal Architecture - part 2

In the [first part](https://rzayev-sehriyar.medium.com/flask-blog-tutorial-with-hexagonal-architecture-part-1-6446e7e9aaaa) 
of this series, we have launched the idea and initial code base for rewriting the original 
[Flask Blog tutorial](https://flask.palletsprojects.com/en/2.2.x/tutorial/) using Hexagonal Architecture.

This version of tutorial is full of Dependency Inversion and Dependency Injection topics, so stay tuned, 
until you learn something new.

As we have concluded with the Ports part of the Ports and Adapters pattern(namely Hexagonal Architecture), it is time to implement the adapters side.
Please refer to the [first part](https://rzayev-sehriyar.medium.com/flask-blog-tutorial-with-hexagonal-architecture-part-1-6446e7e9aaaa) for the project folder structure.

## Repository pattern implementations

We are going to have two separate repos, `PostRepository` and `UserRepository` both of them implements the `RepositoryInterface` from ports part.

Path: `src/adapters/db/post_repository.py`:

```py
from sqlite3 import Connection
from src.domain.ports.repository import RepositoryInterface
from typing import Callable, Any


class PostRepository(RepositoryInterface):

    def __init__(self, db_conn: Callable[[], Connection]) -> None:
        self.db_conn = db_conn()

    def execute(self, query: str, data: tuple[Any, ...], commit: bool = False) -> Any:
        result = self.db_conn.execute(query, data)
        if commit:
            self.commit()
        return result

    def commit(self) -> None:
        self.db_conn.commit()

```

And `src/adapters/db/user_repository.py`:

```py
from sqlite3 import Connection
from src.domain.ports.repository import RepositoryInterface
from typing import Callable, Any


class UserRepository(RepositoryInterface):

    def __init__(self, db_conn: Callable[[], Connection]) -> None:
        self.db_conn = db_conn()

    def execute(self, query: str, data: tuple[Any, ...], commit: bool = False) -> Any:
        result = self.db_conn.execute(query, data)
        if commit:
            self.commit()
        return result

    def commit(self) -> None:
        self.db_conn.commit()
```

Please notice that both repos, accepts database connection as a dependency, which will be injected by Dependency Injection later.

And again recalling from the [first part](https://rzayev-sehriyar.medium.com/flask-blog-tutorial-with-hexagonal-architecture-part-1-6446e7e9aaaa)
our `PostService` and `UserService` ports has a dependency of `RepositoryInterface`, 
and during Dependency Injection the concrete implementations `PostRepository` and `UserRepository` will be injected.

This is so-called Dependency Inversion principle from SOLID in action. 

The actual higher-level code(`PostService` and `UserService`) does not depend on low-level code(`PostRepository` and `UserRepository`),
both depend on abstractions(interfaces).


## Flask application

So far so good, we have a ports part and also the adapters part, now it is time to initialize our Web app.

Thinking about Flask application, just imagine that the Flask is the trigger or initiator of our project.
It is on the outside left of the Hexagon, and it triggers some actions.

Particularly, the endpoints are going to interact with our system via services - `UserService` and `PostService`.

For example, `/auth/register` should use user service create functionality, etc.

And yes, the services are dependencies for our endpoints, therefore they will be injected by Dependency Injector.

### Flask app structure

Our overall web app structure is:

```sh
adapters/app
├── application.py
├── blueprints
│   ├── auth.py
│   ├── blog.py
│   └── __init__.py
├── __init__.py
├── schema.sql
├── static
│   └── style.css
└── templates
    ├── auth
    │   ├── login.html
    │   └── register.html
    ├── base.html
    └── post
        ├── create.html
        ├── index.html
        └── update.html

5 directories, 13 files
```

All static and templates were grabbed from [Flask blog tutorial code base](https://github.com/pallets/flask/tree/main/examples/tutorial/flaskr).

Let's explore `application.py`:

```py
from flask import Flask, url_for

from src.main.containers import Container
from src.main.config import init_app
from .blueprints.blog import blueprint as blog_blueprint
from .blueprints.auth import blueprint as user_blueprint


def create_app() -> Flask:
    container = Container()
    app = Flask(__name__)
    app.secret_key = "Awesome Secret Key which is going to be hacked."
    app.container = container
    with app.app_context():
        init_app(app)
    app.register_blueprint(blog_blueprint)
    app.register_blueprint(user_blueprint)
    app.add_url_rule("/", endpoint="index")
    return app
```

For now, just ignore the `Container()` parts, as it will be explained in the last part.

The registering blueprints are also should be familiar, but what is the `init_app()`?
It is basically the flask CLI command for initializing the database, and it is located in `main.config`.

That leads us to the main configuration section of our project.

## Configuration

In `src/main/config.py` we put all configs, which is quite naive and proper way of centralizing our settings.

```py
import sqlite3
import click
from flask import current_app
from typing import Callable


def get_db() -> Callable[[], sqlite3.Connection]:
    db = sqlite3.connect(
        "hexagonal",
        detect_types=sqlite3.PARSE_DECLTYPES,
        check_same_thread=False
    )

    db.row_factory = sqlite3.Row
    # Solution for -> TypeError: cannot pickle 'sqlite3.Connection' object
    return lambda: db


def init_db():
    db = get_db()

    with current_app.open_resource('schema.sql') as f:
        db().executescript(f.read().decode('utf8'))


@click.command('init-db')
def init_db_command():
    """Clear the existing data and create new tables."""
    init_db()
    click.echo('Initialized the database.')


def close_db(db: Callable[[], sqlite3.Connection], e=None):
    if db is not None:
        db().close()


def init_app(app):
    app.cli.add_command(init_db_command)
```

The `init_app()` registers `init-db` command (see the `@click.command` decorator) as a Flask CLI command.
