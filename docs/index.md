# ORMar

<p>
<a href="https://pypi.org/project/ormar">
    <img src="https://img.shields.io/pypi/v/ormar.svg" alt="Pypi version">
</a>
<a href="https://pypi.org/project/ormar">
    <img src="https://img.shields.io/pypi/pyversions/ormar.svg" alt="Pypi version">
</a>
<a href="https://travis-ci.com/collerek/ormar">
    <img src="https://travis-ci.com/collerek/ormar.svg?branch=master" alt="Build Status">
</a>
<a href="https://codecov.io/gh/collerek/ormar">
    <img src="https://codecov.io/gh/collerek/ormar/branch/master/graph/badge.svg" alt="Coverage">
</a>
<a href="https://www.codefactor.io/repository/github/collerek/ormar">
<img src="https://www.codefactor.io/repository/github/collerek/ormar/badge" alt="CodeFactor" />
</a>
<a href="https://app.codacy.com/manual/collerek/ormar?utm_source=github.com&utm_medium=referral&utm_content=collerek/oramr&utm_campaign=Badge_Grade_Dashboard">
<img src="https://api.codacy.com/project/badge/Grade/62568734f70f49cd8ea7a1a0b2d0c107" alt="Codacy" />
</a>
</p>

The `ormar` package is an async ORM for Python, with support for Postgres,
MySQL, and SQLite. Ormar is built with:

  * [`SQLAlchemy core`][sqlalchemy-core] for query building.
  * [`databases`][databases] for cross-database async support.
  * [`pydantic`][pydantic] for data validation.

Because ormar is built on SQLAlchemy core, you can use [`alembic`][alembic] to provide
database migrations.

The goal was to create a simple ORM that can be used directly with [`fastapi`][fastapi] that bases it's data validation on pydantic.
Initial work was inspired by [`encode/orm`][encode/orm].
The encode package was too simple (i.e. no ability to join two times to the same table) and used typesystem for data checks.

**ormar is still under development:** We recommend pinning any dependencies with `ormar~=0.0.1`

**Note**: Use `ipython` to try this from the console, since it supports `await`.

```python
import databases
import ormar
import sqlalchemy

database = databases.Database("sqlite:///db.sqlite")
metadata = sqlalchemy.MetaData()


class Note(ormar.Model):
    __tablename__ = "notes"
    __database__ = database
    __metadata__ = metadata

    # primary keys of type int by dafault are set to autoincrement    
    id = ormar.Integer(primary_key=True)
    text = ormar.String(length=100)
    completed = ormar.Boolean(default=False)

# Create the database
engine = sqlalchemy.create_engine(str(database.url))
metadata.create_all(engine)

# .create()
await Note.objects.create(text="Buy the groceries.", completed=False)
await Note.objects.create(text="Call Mum.", completed=True)
await Note.objects.create(text="Send invoices.", completed=True)

# .all()
notes = await Note.objects.all()

# .filter()
notes = await Note.objects.filter(completed=True).all()

# exact, iexact, contains, icontains, lt, lte, gt, gte, in
notes = await Note.objects.filter(text__icontains="mum").all()

# .get()
note = await Note.objects.get(id=1)

# .update()
await note.update(completed=True)

# .delete()
await note.delete()

# 'pk' always refers to the primary key
note = await Note.objects.get(pk=2)
note.pk  # 2
```

Ormar supports loading and filtering across foreign keys...

```python
import databases
import ormar
import sqlalchemy

database = databases.Database("sqlite:///db.sqlite")
metadata = sqlalchemy.MetaData()


class Album(ormar.Model):
    __tablename__ = "album"
    __metadata__ = metadata
    __database__ = database

    id = ormar.Integer(primary_key=True)
    name = ormar.String(length=100)


class Track(ormar.Model):
    __tablename__ = "track"
    __metadata__ = metadata
    __database__ = database

    id = ormar.Integer(primary_key=True)
    album = ormar.ForeignKey(Album)
    title = ormar.String(length=100)
    position = ormar.Integer()


# Create some records to work with.
malibu = await Album.objects.create(name="Malibu")
await Track.objects.create(album=malibu, title="The Bird", position=1)
await Track.objects.create(album=malibu, title="Heart don't stand a chance", position=2)
await Track.objects.create(album=malibu, title="The Waters", position=3)

fantasies = await Album.objects.create(name="Fantasies")
await Track.objects.create(album=fantasies, title="Help I'm Alive", position=1)
await Track.objects.create(album=fantasies, title="Sick Muse", position=2)


# Fetch an instance, without loading a foreign key relationship on it.
track = await Track.objects.get(title="The Bird")

# We have an album instance, but it only has the primary key populated
print(track.album)       # Album(id=1) [sparse]
print(track.album.pk)    # 1
print(track.album.name)  # Raises AttributeError

# Load the relationship from the database
await track.album.load()
assert track.album.name == "Malibu"

# This time, fetch an instance, loading the foreign key relationship.
track = await Track.objects.select_related("album").get(title="The Bird")
assert track.album.name == "Malibu"

# By default you also get a second side of the relation 
# constructed as lowercase source model name +'s' (tracks in this case)
# you can also provide custom name with parameter related_name
album = await Album.objects.select_related("tracks").all()
assert len(album.tracks) == 3

# Fetch instances, with a filter across an FK relationship.
tracks = Track.objects.filter(album__name="Fantasies")
assert len(tracks) == 2

# Fetch instances, with a filter and operator across an FK relationship.
tracks = Track.objects.filter(album__name__iexact="fantasies")
assert len(tracks) == 2

# Limit a query
tracks = await Track.objects.limit(1).all()
assert len(tracks) == 1
```

## Data types

The following keyword arguments are supported on all field types.

  * `primary_key`
  * `nullable`
  * `default`
  * `server_default`
  * `index`
  * `unique`

## Model Fields

### Common parameters

All fields are required unless one of the following is set:

  * `nullable` - Creates a nullable column. Sets the default to `None`.
  * `default` - Set a default value for the field.
  * `server_default` - Set a default value for the field on server side (like sqlalchemy's `func.now()`).
  * `primary key` - Set a primary key on a column.
  * `autoincrement` - When a column is set to primary key and autoincrement is set on this column. 
    Autoincrement is set by default on int primary keys. 

### Fields Types

* `String(length)`
* `Text()`
* `Boolean()`
* `Integer()`
* `Float()`
* `Date()`
* `Time()`
* `DateTime()`
* `JSON()`
* `BigInteger()`
* `Decimal(lenght, precision)`

[sqlalchemy-core]: https://docs.sqlalchemy.org/en/latest/core/
[databases]: https://github.com/encode/databases
[pydantic]: https://pydantic-docs.helpmanual.io/
[encode/orm]: https://github.com/encode/orm/
[alembic]: https://alembic.sqlalchemy.org/en/latest/
[fastapi]: https://fastapi.tiangolo.com/