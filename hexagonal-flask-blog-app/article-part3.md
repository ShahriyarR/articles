# Flask Blog tutorial rewritten with Hexagonal Architecture - part 2

Photo from:
http://thinkmicroservices.com/blog/2019/hexagonal-architecture.html

[GitHub Repo of this project](https://github.com/ShahriyarR/hexagonal-flask-blog-tutorial)

In the [first part](https://rzayev-sehriyar.medium.com/flask-blog-tutorial-with-hexagonal-architecture-part-1-6446e7e9aaaa) 
of this series, we have launched the idea and initial code base for rewriting the original 
[Flask Blog tutorial](https://flask.palletsprojects.com/en/2.2.x/tutorial/) using Hexagonal Architecture.
The ports and domain model were constructed

In the [second part](https://blog.devgenius.io/flask-blog-tutorial-with-hexagonal-architecture-part-2-8930ca009c27)
we have implemented the adapters and the Flask app endpoints are ready to go.

In this third and final part we are going to wire every parts together using Dependency Injection Container and fire up our application.

## UserContainer

We are going to have post and user containers separately as `src/main/user_containers.py` and `src/main/post_containers.py`.

Let's see what we have in `src/main/user_containers.py`:

```py
from dependency_injector import containers, providers
from src.adapters.db.user_repository import UserRepository
from src.domain.ports.user_service import UserService


class UserContainer(containers.DeclarativeContainer):

    db_conn = providers.Dependency()

    user_repository = providers.Singleton(
        UserRepository,
        db_conn=db_conn
    )

    user_service = providers.Factory(
        UserService,
        user_repo=user_repository
    )
```

We have introduced `db_conn` as a "virtual" dependency which is going to be replaced by actual implementation in the main container.

> `src/main/containers.py` this is the main container

As we recall the `UserRepostory` accepts the actual database connection as dependency:

```py
class UserRepository(RepositoryInterface):

    def __init__(self, db_conn: Callable[[], Connection]) -> None:
        self.db_conn = db_conn()
```

And in `UserContainer` we are initializing the `UserRepository` with `db_conn` injected as "virtual" dependency:

```py
user_repository = providers.Singleton(
        UserRepository,
        db_conn=db_conn
    )
```

The next is we need to create `UserService` instance with user repository, as this is a dependency for our service:

```py
class UserService:

    def __init__(self, user_repo: RepositoryInterface) -> None:
        self.user_repo = user_repo
```

Again, notice that the UserService actually does not depend on concrete `UserRepository` but the `RepositoryInterface`,
which is a clear Dependency Inversion Principle in action.

In our `UserContainer` as we have created the `user_repository` as a provider of `UserRepository`(and it is a `RepositoryInterface`) that means,
we can inject it to the `UserService`:

```py
user_service = providers.Factory(
        UserService,
        user_repo=user_repository
    )
```

## PostContainer

And now let's look at `src/main/post_containers.py`:

```py
from dependency_injector import containers, providers
from src.adapters.db.post_repository import PostRepository
from src.domain.ports.post_service import PostService


class PostContainer(containers.DeclarativeContainer):

    db_conn = providers.Dependency()

    post_repository = providers.Singleton(
        PostRepository,
        db_conn=db_conn
    )

    post_service = providers.Factory(
        PostService,
        post_repo=post_repository
    )

```

The idea is the same above, we first create the `db_conn` "virtual" dependency, then initialize the `PostRepository` with this `db_conn`.

Finally, create `post_service` provider from the `PostService` and pass the `post_repository` as a dependency to `PostService`.

## Main Container

The main Dependency Injector Container is located in `src/main/containers.py`:

```py
from dependency_injector import containers, providers

from src.main.config import get_db
from src.main.user_containers import UserContainer
from src.main.post_containers import PostContainer


class Container(containers.DeclarativeContainer):
    wiring_config = containers.WiringConfiguration(packages=["src.adapters.app.blueprints"])
    db_connection = get_db()

    user_package = providers.Container(
        UserContainer,
        db_conn=db_connection
    )

    blog_package = providers.Container(
        PostContainer,
        db_conn=db_connection
    )

```

The first line `containers.WiringConfiguration` please read about it here: [Wiring configuration](https://python-dependency-injector.ets-labs.org/wiring.html#wiring-configuration)

> We will explore it in a moment, for now read the doc.

Please notice about `db_connection = get_db()`, if you recall in `UserContainer` and `PostContainer` we defined the `db_conn` as a "virtual" dependency.
But now, we have the concrete, real db connection, which we have injected to the containers.

As a result we have `user_package` and `blog_package` which stores the respective real dependencies initialized.

## PoC in the Flask endpoints:

Go back to the `register` endpoint(`src/adapters/app/blueprints/auth.py`):

```py
@blueprint.route('/register', methods=('GET', 'POST'))
@inject
def register(user_service: UserService = Provide[Container.user_package.user_service]):
```

Notice that, how it is injected from the main container: `Container.user_package.user_service`.
This actual injection happens.

## PoC in the Flask create_app factory

In the `src/adapters/app/application.py` where we have initialized the Flask app there is a code portion:

```py
from src.main.containers import Container

def create_app() -> Flask:
    container = Container()
    ...
```

This is a main Container initialized, which will automatically wire our Flask blueprints together with DI Containers.

`containers.WiringConfiguration(packages=["src.adapters.app.blueprints"])` - we have provided the path to the blueprints.

## Running the application

Remaining steps:

* Init the database

`flask --app src.adapters.app.application init-db`
Initialized the database.


* Start development service:

`flask --app src.adapters.app.application --debug run`
```
* Serving Flask app 'src.adapters.app.application'
 * Debug mode: on
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on http://127.0.0.1:5000
Press CTRL+C to quit
```

That was a real journey for us, let's wrap up what we have achieved so far:

* We have implemented the simple Blog web app with Hexagonal Architecture.
* We have decoupled nearly every part of our application.
* From now, you are free to change Flask to Bottle or Pyramid: just change the DI Container wiring and create endpoints.
* From now, it is easy to switch from SQLite to PG or MySQL: just implement MySQL or PG Repositories and inject them in the relevant places.



