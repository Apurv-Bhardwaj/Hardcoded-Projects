# Flask-SQLAlchemy — Query Practice Notes

> Note: `python`/`flask shell` don't save command history on this machine (no
> readline history file), so this is reconstructed from your git commit
> messages ("added 2 new test users", "tested db with python shell commands")
> and the one line captured in PowerShell history (`db.session.get`). Matches
> the `flask shell` context in [microblog.py](microblog.py), which auto-injects
> `sa`, `so`, `db`, `User`, `Post` — no imports needed inside the shell.

## Starting the shell

```python
flask shell
# Drops into a Python REPL with app context already pushed, and
# sa, so, db, User, Post already bound (see make_shell_context() in microblog.py).
# Without `flask shell` you'd need to do this by hand:
from app import app, db
from app.models import User, Post
import sqlalchemy as sa
```

## Creating and saving users

```python
u = User(username='john', email='john@example.com')
# Builds a User object in memory only — not in the database yet.

db.session.add(u)
# Stages the object in the current session (like `git add`) — still not saved.

db.session.commit()
# Flushes the session and commits the transaction — row is now in the DB.

u = User(username='susan', email='susan@example.com')
db.session.add(u)
db.session.commit()
# Same pattern for a second test user.
```

## Reading users back

```python
query = sa.select(User)
# Builds a SELECT * FROM user statement (doesn't run it yet — lazy).

users = db.session.scalars(query).all()
# Executes the query and returns a list of User objects (.scalars() unwraps
# each row to a single column/entity instead of a Row tuple).

for u in users:
    print(u.id, u.username)
# Iterate and print — relies on User having id/username columns.

u = db.session.get(User, 1)
# Primary-key lookup shortcut — equivalent to
# SELECT * FROM user WHERE id = 1, but skips building a select() manually.

u
# Bare expression in the REPL — prints repr(u), i.e. '<User john>'
# (from the __repr__ method we added on the User model).
```

## Adding a post linked to a user

```python
u = db.session.get(User, 1)
# Re-fetch john so we have a User instance to attach the post to.

p = Post(body='my first post!', author=u)
# Creates a Post row and sets the relationship directly — SQLAlchemy fills
# in user_id from `u.id` automatically via the author/posts relationship.

db.session.add(p)
db.session.commit()
# Stage + commit, same as with users above.
```

## Querying posts through the relationship

```python
query = sa.select(Post)
posts = db.session.scalars(query)
# Note: no .all() here — `posts` is an iterator, not a list. It can only
# be consumed once.

for p in posts:
    print(p.id, p.author.username, p.body)
# p.author triggers a lazy-load back to the User row via the foreign key —
# this is the ORM following user_id -> user.id for you.
```

```python
u = db.session.get(User, 1)
posts = u.posts.select()
# u.posts is a WriteOnlyMapped collection (see models.py) — it doesn't load
# all posts into memory, it hands back a select() you still have to run.

db.session.scalars(posts).all()
# Actually executes it and materializes the list of that user's posts.
```

## Resetting the schema (terminal, not shell)

```powershell
flask db downgrade base
# Runs every migration's downgrade() in reverse, back to an empty schema —
# useful when you want to replay migrations from scratch.

flask db upgrade
# Re-runs upgrade() for every migration in order, rebuilding the schema
# (tables + indexes) from nothing.
```
