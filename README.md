# Django Migrations
How to create, name, run, test and manage Django migrations

## Creating Migrations
Migration files can be created automatically via  `./manage.py makemigrations` or manually by 
`./manage.py makemigrations app_name --empty`

## Naming Migration Files
Just like every other piece of code we write migration files need to be properly documented and named intuitively.

Django will sometime name your migrations for you, if the change is simple enough, but when it doesn't, the resulting title is very unhelpful.
what does migration `0006_auto_20220609_2145` contain? who knows. Is it important? Perhaps. Life altering? possibly.

That's where naming migrations comes in. A better name for a migration file is `0006_book_fiction`. Without opening the file you might be able to guess what this migration file is doing.

You can provide the `-n/--name` argument to makemigrations, but we often forget this.

The correct way to make migrations:  `python manage.py makemigrations --name mymodel_myfield`

## Migration Contents
Now that we have our migration file we need to make sure we have some things checked off of our list:

1. Does this migration need a sperate `RunPython` operation?
2. If so do we have our forward and reverse functions written?

### Example Scenario
Say we are adding a non-nullable, unique field named slug to an existing model called Post.
Generally speaking, always add new fields either as null=True or with a default value. If we can’t solve the problem with the default parameter, first create the field as null=True then create a data migration for it. After that we can then create a new migration to set the field as null=False.

Here is how we can do it:

```python
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=255)
    date = models.DateTimeField(auto_now_add=True)
    content = models.TextField()
    slug = models.SlugField(null=True)

    def __str__(self):
        return self.title
```

Create the migration:

```
python manage.py makemigrations blog

Migrations for 'blog':
  blog/migrations/0002_post_slug.py
    - Add field slug to post
```

Apply it:

```
python manage.py migrate blog

Operations to perform:
  Apply all migrations: blog
Running migrations:
  Applying blog.0002_post_slug... OK
```

At this point, the database already have the slug column.

Create an empty migration with the following command:
   
```   
python manage.py makemigrations blog --empty

Migrations for 'blog':
  blog/migrations/0003_auto_20170926_1105.py
```

## The Django Migration Table
Django keeps a record of all the migrations applied in a table called `django_migrations`

You can view all the applied migrations with:

    from django.db.migrations.recorder import MigrationRecorder
    latest_migration = MigrationRecorder.Migration.objects.order_by('-applied')
