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


