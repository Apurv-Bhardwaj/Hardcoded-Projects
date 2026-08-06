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

## Bugs found & fixed (running log)

Chronological log of real bugs hit while building this app — symptom, root
cause, fix. Kept short on purpose; the "why" is the part worth remembering.

1. **`config.py` — `os.ath.abspath` / `os.environ.het`**
   Typos for `os.path` / `os.environ.get`. Crashed on the very first import,
   before Flask even started — `AttributeError: module 'os' has no attribute 'ath'`.

2. **`config.py` — `SQLALCHEMY_DATABSE_URI` (missing “A”)**
   Flask-SQLAlchemy never saw the real config key, so it had no database URI
   at all: `RuntimeError: Either 'SQLALCHEMY_DATABASE_URI' or 'SQLALCHEMY_BINDS' must be set`.
   This is what made `flask db init`/`migrate` fail.

3. **`models.py` — `so.mapped[str]` (lowercase) instead of `so.Mapped[str]`**
   Python is case-sensitive; `sqlalchemy.orm` only exports `Mapped`.
   `AttributeError: module 'sqlalchemy.orm' has no attribute 'mapped'`.

4. **`models.py` — `author = so.Mapped[User] = so.relationship(...)`**
   Chained assignment instead of a type annotation (missing `:`). Tried to
   call `so.Mapped.__setitem__`, which doesn't exist — broken at class-body
   execution time, not just a lint warning.

5. **`models.py` — `__repr__` dedented out of the `User` class**
   Ended up as a dangling module-level function using `self` with nothing to
   bind it to — dead code, and it's why `User`/`Post` printed with default
   reprs instead of `<User john>`.

6. **`werkzeug.security` works, but `User.set_password`/`check_password` didn't exist yet**
   `generate_password_hash`/`check_password_hash` are just functions — they
   don't attach themselves to the model. Had to write the two wrapper methods
   on `User` by hand.

7. **`app/__init__.py` — a second `app = Flask(__name__)`**
   The big one. A stray duplicate `Flask()` call overwrote the configured
   `app` instance with a fresh, unconfigured one *after* `db`/`Config` were
   already bound to the first instance. Everything after that line
   (`LoginManager`, all of `routes.py`) attached to the empty instance —
   config and db extension effectively vanished. Symptoms varied depending
   on what ran first: `AttributeError: 'Flask' object has no attribute 'login_manager'`,
   then `RuntimeError: no secret key was set`, then `SQLALCHEMY_DATABASE_URI` reading `None`.
   One duplicate line, three different-looking errors — a good reminder to
   check for double-initialization before chasing each symptom separately.

8. **`routes.py` — `login()` used `db.session.scalars()` (plural) and checked the wrong field**
   `.scalars()` returns an iterator that's never `None`, so the "user not
   found" branch could never trigger correctly; needed `.scalar()` (singular)
   for a single result-or-`None`. Also compared the password against
   `form.username.data` instead of `form.password.data` — login would have
   accepted the username as if it were the password.

9. **`login.html` — `url_for('register) }}` missing closing quote**
   Broke Jinja compilation entirely: `TemplateSyntaxError: unexpected char "'"`.

10. **`register.html` — `for.password2.label` / `for.submit()`**
    Typo'd `form` as `for` (a reserved word in Jinja's `{% for %}` tag, but
    fine as a plain variable name — just the wrong one here). Jinja treated
    it as an undefined variable: `UndefinedError: 'for' is undefined`.

11. **`routes.py` — `index()` built a `user` dict but never passed it to `render_template`**
    `index.html` does `{{ user.username }}`; since `user` wasn't in the
    template context, Jinja raised `UndefinedError: 'user' is undefined`.
    Classic case of the local variable existing but never making it into the
    call that actually needs it.

**Pattern across most of these:** almost every bug was a *name* mismatch —
a typo, a wrong case, a variable built but not passed through — not a logic
or design problem. Worth a slow read-through of variable names before
assuming something structural is broken.
