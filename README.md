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
python manage.py makemigrations blog --name post_slug

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
python manage.py makemigrations blog --empty --name post_slug_not_nullable

Migrations for 'blog':
  blog/migrations/0003_post_slug_not_nullable.py

```

Now open the file 0003_post_slug_not_nullable.py, and it should have the following contents:

```python
from __future__ import unicode_literals
from django.db import migrations


class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0002_post_slug'),
    ]

    operations = [
    ]
    
```

Then here in this file, we can create a function that can be executed by the `RunPython` command:

```python
from __future__ import unicode_literals

from django.db import migrations
from django.utils.text import slugify


def slugify_title(apps, schema_editor):
    '''
    We can't import the Post model directly as it may be a newer
    version than this migration expects. We use the historical version.
    '''
    Post = apps.get_model('blog', 'Post')
    for post in Post.objects.all():
        post.slug = slugify(post.title)
        post.save()
        
def reverse_slugify_title(apps, schema_editor):
    '''
    Remove all the values in the slugify field
    '''
    Post = apps.get_model('blog', 'Post')
    for post in Post.objects.all():
        post.slug = None
        post.save()


class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0002_post_slug'),
    ]

    operations = [
        migrations.RunPython(slugify_title, reverse_slugify_title),
    ]
```

Save the file and execute the migration as you would do with a regular model migration:

```
python manage.py migrate blog
Operations to perform:
  Apply all migrations: blog
Running migrations:
  Applying blog.0003_post_slug_not_nullable... OK
```

Every Post entry have a value, so we can safely change the switch from null=True to null=False. And since all the values are unique, we can also add the unique=True flag.

```python
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=255)
    date = models.DateTimeField(auto_now_add=True)
    content = models.TextField()
    slug = models.SlugField(null=False, unique=True)

    def __str__(self):
        return self.title
```

Create a new migration:
```
python manage.py makemigrations blog --name post_slug_unique
```

This time you will see the following prompt:

```
You are trying to change the nullable field 'slug' on post to non-nullable without a default; we can't do that
(the database needs something to populate existing rows).
Please select a fix:
 1) Provide a one-off default now (will be set on all existing rows with a null value for this column)
 2) Ignore for now, and let me handle existing rows with NULL myself (e.g. because you added a RunPython or RunSQL
 operation to handle NULL values in a previous data migration)
 3) Quit, and let me add a default in models.py
Select an option:
```

Select option 2 by typing “2” in the terminal.

```
Migrations for 'blog':
  blog/migrations/0004_post_slug_unique.py
    - Alter field slug on post
```

Creating a reverse migration method is not needed for this migration as Django will simply remove the unique constraint on the field.

Now we can safely apply the migration:
```
python manage.py migrate blog
Operations to perform:
  Apply all migrations: blog
Running migrations:
  Applying blog.0004_0004_post_slug_unique... OK
```

## The Django Migration Table
Django keeps a record of all the migrations applied in a table called `django_migrations`

You can view all the applied migrations with:

    from django.db.migrations.recorder import MigrationRecorder
    latest_migration = MigrationRecorder.Migration.objects.order_by('-applied')
