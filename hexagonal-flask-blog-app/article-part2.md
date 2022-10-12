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

There is a `schema.sql` file which is located in `src/adapters/app/schema.sql` which is going to be used in `init_db()`,
to create database tables. As the `init-db` command is part of the flask app command, the `schema.sql` is also located,
nearby.

> We are not using ORM here as the original Flask blog tutorial misses this topic.

Another interesting part is `get_db()` function, if you noticed we are returning the callable from this function, 
and the caller should call the returned value to get the actual db connection.

This is a workaround on [Dependency Injector](https://github.com/ets-labs/python-dependency-injector)
, as the connection object itself can not be pickled.

## Flask blueprints

Now we are getting closer to actual web app. 

Let's define our blueprints, in `src/adapters/app/blueprints`.
Basically we have two files `auth.py` for authentication endpoints, and `blog.py` for blogging:

```sh
adapters/app/blueprints
├── auth.py
├── blog.py
└── __init__.py

0 directories, 3 files
```

`auth.py`:

```py
import functools

from flask import Blueprint, request, redirect, url_for, flash, render_template, session, g
from dependency_injector.wiring import inject, Provide
from werkzeug.security import check_password_hash

from src.domain.ports import register_user_factory
from src.domain.ports.user_service import UserService, UserDBOperationError
from src.main.containers import Container

blueprint = Blueprint('auth', __name__, url_prefix='/auth')


def login_required(view):
    """View decorator that redirects anonymous users to the login page."""

    @functools.wraps(view)
    def wrapped_view(**kwargs):
        if g.user is None:
            return redirect(url_for("auth.login"))

        return view(**kwargs)

    return wrapped_view


@blueprint.before_app_request
@inject
def load_logged_in_user(user_service: UserService = Provide[Container.user_package.user_service]):
    """If a user id is stored in the session, load the user object from
    the database into ``g.user``."""
    user_id = session.get("user_id")

    if user_id is None:
        g.user = None
    else:
        g.user = (
            user_service.get_user_by_id(user_id)
        )


@blueprint.route('/register', methods=('GET', 'POST'))
@inject
def register(user_service: UserService = Provide[Container.user_package.user_service]):
    if request.method == 'POST':
        user_name = request.form['username']
        password = request.form['password']
        error = None
        if not user_name:
            error = 'Username is required.'
        elif not password:
            error = 'Password is required.'
        user_ = register_user_factory(user_name=user_name, password=password)
        if not error:
            try:
                user_service.create(user_)
            except UserDBOperationError as err:
                print(err)
                error = f"Something went wrong with database operation {err}"
            else:
                return redirect(url_for("auth.login"))
        flash(error)

    return render_template('auth/register.html')


@blueprint.route('/login', methods=('GET', 'POST'))
@inject
def login(user_service: UserService = Provide[Container.user_package.user_service]):
    if request.method == 'POST':
        user_name = request.form['username']
        password = request.form['password']

        error = None
        user = user_service.get_user_by_user_name(user_name=user_name)
        if not user:
            error = 'Incorrect username.'
        elif not check_password_hash(user['password'], password):
            error = 'Incorrect password.'

        if not error:
            session.clear()
            session['user_id'] = user['id']
            return redirect(url_for('index'))

        flash(error)

    return render_template('auth/login.html')


@blueprint.route("/logout")
def logout():
    """Clear the current session, including the stored user id."""
    session.clear()
    return redirect(url_for("index"))
```

The logout function and login_required decorator is straightforward and requires no attention.

The rest is quite interesting. The `register` function signature is:

```py
@blueprint.route('/register', methods=('GET', 'POST'))
@inject
def register(user_service: UserService = Provide[Container.user_package.user_service]):
```

Notice that, the `user_service` is a dependency for this function and the actual `user_service` object is provided by 
the Dependency Injector Container(`Provide[Container.user_package.user_service]`). 
The `@inject` decorator indicates that the `user_service` is going to be injected, provided.

> We will explore the Dependency Injection topic in third part, so please do not panic.

Let's explain a bit the code portion below:

```py
user_ = register_user_factory(user_name=user_name, password=password)
if not error:
    try:
        user_service.create(user_)
```

`register_user_factory` accepts user_name and password and returns the `RegisterUserInputDto` as we have explained,
in the first part of these article series.

```py
def register_user_factory(user_name: str, password: str) -> RegisterUserInputDto:
    return RegisterUserInputDto(user_name=user_name, password=generate_password_hash(password))
```

Notice that password hashing, and all other validation stuff is going to happen inside the factory.
We have delegate this responsibility from endpoints and services to Dto side.

After getting back the `RegisterUserInputDto` object we send it directly to the `user_service.create()` method, 
which has the following signature:

```py
def create(self, user: RegisterUserInputDto) -> User:
    user = user_factory(user_name=user.user_name, password=user.password)
```

It accepts the `RegisterUserInputDto` then creates actual User model using user_factory.

```py
def user_factory(user_name: str, password: str) -> User:
    # data validation should happen here
    return User(id_=str(uuid4()), user_name=user_name, password=password)
```

Notice that how the `uuid4` generation is also hidden inside the user factory, 
because it is not the responsibility of service create.

The rest is easy, insert the User into the User database table and return it back.

The same principles and ideas is valid for blog blueprint.

Let's explore it in detail, `blog.py`:

```py
from flask import Blueprint, render_template, request, flash, g, redirect, url_for
from dependency_injector.wiring import inject, Provide

from src.adapters.app.blueprints.auth import login_required
from src.domain.ports import create_post_factory, update_post_factory, delete_post_factory
from src.domain.ports.post_service import PostService, BlogDBOperationError
from src.main.containers import Container

blueprint = Blueprint('post', __name__)


@blueprint.route('/')
@inject
def index(post_service: PostService = Provide[Container.blog_package.post_service]):
    posts = post_service.get_all_blogs()
    return render_template('post/index.html', posts=posts)


@blueprint.route('/create', methods=('GET', 'POST'))
@login_required
@inject
def create(post_service: PostService = Provide[Container.blog_package.post_service]):
    if request.method == 'POST':
        title = request.form['title']
        body = request.form['body']
        error = None

        if not title:
            error = 'Title is required.'

        post = create_post_factory(author_id=g.user["id"], title=title, body=body)
        if not error:
            try:
                post_service.create(post)
            except BlogDBOperationError as err:
                error = f"Something went wrong with database operation {err}"
            else:
                return redirect(url_for('post.index'))
        flash(error)

    return render_template('post/create.html')


@blueprint.route('/<int:id>/update', methods=('GET', 'POST'))
@login_required
@inject
def update(id, post_service: PostService = Provide[Container.blog_package.post_service]):
    post = post_service.get_post_by_id(id)

    if request.method == 'POST':
        title = request.form['title']
        body = request.form['body']
        error = None

        if not title:
            error = 'Title is required.'

        _post = update_post_factory(id=id, title=title, body=body)

        if not error:
            try:
                post_service.update(_post)
            except BlogDBOperationError as err:
                error = f"Something went wrong with database operation {err}"
            else:
                return redirect(url_for('post.index'))
        flash(error)

    return render_template('post/update.html', post=post)


@blueprint.route('/<int:id>/delete', methods=('POST',))
@login_required
@inject
def delete(id, post_service: PostService = Provide[Container.blog_package.post_service]):
    post_service.get_post_by_id(id)
    _post = delete_post_factory(id=id)
    try:
        post_service.delete(_post)
    except BlogDBOperationError as err:
        error = f"Something went wrong with database operation {err}"
    else:
        return redirect(url_for('post.index'))
    flash(error)
```

Create, Update and Delete endpoints accepts `PostService` as a dependency from Dependency Injection Container.
The again notable things we have separated `create_post_factory`, `update_post_factory` and `delete_post_factory`,
for creating Dtos: `CreatePostInputDto`, `UpdatePostInputDto` and `DeletePostInputDto`.

Those Dtos then passed to actual respective service actions.

In this second part we have explored:

* Repository pattern implementations
* Flask app starter code, database initialization
* Briefly introduced the Dependency Injection Container ideas
* Implemented the web endpoints, with Flask blueprints