# tauorm

A SQLAlchemy-style SQL toolkit + ORM for [Tauraro](https://github.com/tauraro/tauraro)
— a dialect-agnostic Core (engine, expression language, schema) plus a
declarative ORM (sessions, unit of work, relationships), over pluggable
`taupkg` backend packages.

**Status: proposal stage.** See [PROPOSAL.md](PROPOSAL.md) for the full
architecture, the concrete divergence between the two backend drivers that
drives the dialect design, and the phased build plan. Nothing here is
implemented yet.

```python
from orm import DeclarativeBase, Column, Integer, String, Session, create_engine

class User(DeclarativeBase):
    __tablename__ = "users"
    id   = Column(Integer, primary_key=True)
    name = Column(String(100))

def main():
    mut engine = create_engine("sqlite:///app.db")   # or "postgres://user@localhost/app"
    User.metadata.create_all(engine)

    mut session = Session(engine)
    session.add(User(name="Ada"))
    session.commit()

    for user in session.query(User).all():
        print(user.id, user.name)
```

## Choosing a backend

Both database backends are optional `taupkg` dependencies — pick what your
project actually needs at build time:

```
taupkg build --features postgres          # Postgres only (needs ../taupostgres)
taupkg build --features sqlite            # SQLite only (needs ../tausqlite3)
taupkg build --features postgres,sqlite   # both
```

## License

MIT.