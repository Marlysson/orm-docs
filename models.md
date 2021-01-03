# Models

Models are the easiest way to interact with your tables. A model is a way for you to interact with a Python class in a simple and elegant way and have all the hard overhead stuff handled for you under the hood. A model can be used to query the data in the table or even create new records, fetch related records between tables and many other features.

## Creating A Model

The first step in using models is actually creating them. You can scaffold out a model by using the command:

```text
$ python craft model Post
```

This will create a post model like so:

```python
from masoniteorm.models import Model

class Post(Model):
    """Post Model"""
    pass
```

From here you can do as basic or advanced queries as you want. You may need to configure your model based on your needs, though.

From here you can start querying your records:

```python
user = User.first()
users = User.all()
active_users = User.where('active', 1).first()
```

We'll talk more about setting up your model below

## Conventions And Configuration

Masonite ORM makes a few assumptions in order to have the easiest interface for your models.

The first is table names. Table names are assumed to be the plural of your model name. If you have a User model then the `users` table is assumed and if you have a model like `Company` then the `companies` table is assumed. You can realize that Masonite ORM is smart enough to know that the plural of `Company` is not `Companys` so don't worry about Masonite not being able to pick up your table name.

### Table Name

If your table name is something other than the plural of your models you can change it using the `__table__` attribute:

```python
class Clients:
  __table__ = "users"
```

### Primary Keys

The next thing Masonite assumes is the primary key. Masonite ORM assumes that the primary key name is `id`. You can change the primary key name easily:

```python
class Clients:
  __primary_key__ = "user_id"
```

### Connections

The next thing Masonite assumes is that you are using the `default` connection you setup in your configuration settings. You can also change thing on the model:

```python
class Clients:
  __connection__ = "staging"
```

### Mass Assignment

By default, Masonite ORM protects against mass assignment to help prevent users from changing values on your tables you didn't want.

This is used in the create and update methods. You can set the columns you want to be mass assignable easily:

```python
class Clients:
  __fillable__ = ['email', "active", "password"]
```

### Timestamps

Masonite also assumed you have `created_at` and `updated_at` columns on your table. You can easily disable this behavior:

```python
class Clients:
  __timestamps__ = False
```

### Timezones

Models use `UTC` as the default timezone. You can change the timezones on your models using the `__timezone__` attribute:

```python
class User(Model):
    __timezone__ = "Europe/Paris"
```

## Querying

Almost all of a models querying methods are passed off to the query builder. If you would like to see all the methods available for the query builder, see the [QueryBuilder](models.md) documentation here.

* sub queries

### Single results

A query result will either have 1 or more records. If your model result has a single record then the result will be the model instance. You can then access attributes on that model instance. Here's an example:

```python
from app.models import User

user = User.first()
user.name #== 'Joe'
user.email #== 'joe@masoniteproject.com'
```

You can also get a record by its primary key:

```python
from app.models import User

user = User.find(1)
user.name #== 'Joe'
user.email #== 'joe@masoniteproject.com'
```

### Collections

If your model result returns several results then it will be wrapped in a collection instance which you can use to iterate over:

```python
from app.models import User

users = User.where('active', 1).get()
for users in user:
  user.name #== 'Joe'
  user.active #== '1'
  user.email #== 'joe@masoniteproject.com'
```

If you want to find a collection of records based on the models primary key you can pass a list to the `find` method:

```python
users = User.find([1,2,3])
for users in user:
  user.name #== 'Joe'
  user.active #== '1'
  user.email #== 'joe@masoniteproject.com'
```

The collection class also has some handy methods you can use to interact with your data:

```python
user_emails = User.where('active', 1).get().pluck('email') #== Collection of email addresses
```

If you would like to see more methods available like `pluck` be sure to read the [Collections](models.md) documentation.

### Deleting

You may also quickly delete records:

```python
from app.models import User

users = User.delete(1)
```

This will delete the record based on the primary key value of 1.

You can also delete based on a query:

```python
from app.models import User

users = User.where('active', 0).delete()
```

### Sub Queries

You may also use subqueries to do more advanced queries using lambda expressions:

```python
from app.models import User

users = User.where(lambda q: q.where('active', 1).where_null('deleted_at'))
# == SELECT * FROM `users` WHERE (`active` = '1' AND `deleted_at` IS NULL)
```

## Relationships

Another great feature when using models is to be able to relate several models together \(like how tables can relate to eachother\).

### Belongs To

A belongs to relationship is a one-to-one relationship between 2 table records.

You can add a one-to-one relationship easily:

```python
from masoniteorm.relationships import belongs_to
class User:

  @belongs_to
  def company(self):
    from app.models import Company
    return Company
```

It will be assumed here that the primary key of the relationship here between users and companies is `id -> id`. You can change the relating columns if that is not the case:

```python
from masoniteorm.relationships import belongs_to
class User:

  @belongs_to('company_id', 'id')
  def company(self):
    from app.models import Company
    return Company
```

The first argument is always the column name on the current models table and the second argument is the related field on the other table.

### Has Many

Another relationship is a one-to-many relationship where a record relates to many records in another table:

```python
from masoniteorm.relationships import has_many
class User:

  @has_many('company_id', 'id')
  def posts(self):
    from app.models import Post
    return Post
```

The first argument is always the column name on the current models table and the second argument is the related field on the other table.

## Using Relationships

You can easily use relationships to get those related records. Here is an example on how to get the company record:

```python
user = User.first()
user.company #== <app.models.Company>
user.company.name #== Masonite X Inc.

for post in user.posts:
    post.title
```

## Eager Loading

You can eager load any related records. Eager loading is when you preload model results instead of calling the database each time.

Let's take the example of fetching a users phone:

```python
users = User.all()
for user in users:
    user.phone
```

This will result in the query:

```text
SELECT * FROM users
SELECT * FROM phones where user_id = 1
SELECT * FROM phones where user_id = 2
SELECT * FROM phones where user_id = 3
SELECT * FROM phones where user_id = 4
...
```

This will result in a lot of database calls. Now let's take a look at the same example but with eager loading:

```python
users = User.with_('phone').get()
for user in users:
    user.phone
```

This would now result in this query:

```text
SELECT * FROM users
SELECT * FROM phones where user_id IN (1, 2, 3, 4)
```

This resulted in only 2 queries. Any subsquent calls will pull in the result from the eager loaded result set.

### Nested Eager Loading

You may also eager load multiple relationships. Let's take another more advanced example:

Let's say you would like to get a users phone as well as the contacts. The code would look like this:

```python
users = User.all()
for user in users:
    for contacts in user.phone:
        contact.name
```

This would result in the query:

```text
SELECT * FROM users
SELECT * FROM phones where user_id = 1
SELECT * from contacts where phone_id = 30
SELECT * FROM phones where user_id = 2
SELECT * from contacts where phone_id = 31
SELECT * FROM phones where user_id = 3
SELECT * from contacts where phone_id = 32
SELECT * FROM phones where user_id = 4
SELECT * from contacts where phone_id = 33
...
```

You can see how this can get pretty large as we are looping through hundreds of users.

We can use nested eager loading to solve this by specifying the chain of relationships using `.` notation:

```python
users = User.with_('phone.contacts').all()
for user in users:
    for contacts in user.phone:
        contact.name
```

This would now result in the query:

```text
SELECT * FROM users
SELECT * FROM phones where user_id IN (1,2,3,4)
SELECT * from contacts where phone_id IN (30, 31, 32, 33)
```

You can see how this would result in 3 queries no matter how many users you had.

## Scopes

Scopes are a way to take common queries you may be doing and be able to condense them into a method where you can then chain onto them. Let's say you are doing a query like getting the active user a lot:

```python
user = User.where('active', 1).get()
```

We can take this query and add it as a scope:

```python
from masoniteorm.scopes import scope
class User(Model):

  @scope
  def active(self, query):
    return query.where('active', 1)
```

Now we can simply call the active method:

```python
user = User.active().get()
```

You may also pass in arguments:

```python
from masoniteorm.scopes import scope
class User(Model):

  @scope
  def active(self, query, active_or_inactive):
    return query.where('active', active_or_inactive)
```

then pass an argument to it:

```python
user = User.active(1).get()
user = User.active(0).get()
```

## Soft Deleting

Masonite ORM also comes with a global scope to enable soft deleting for your models.

Simply inherit the `SoftDeletes` scope:

```python
from masoniteorm.scopes import SoftDeletesMixin

class User(Model, SoftDeletesMixin):
  # ..
```

Now whenever you delete a record, instead of deleting it it will update the `deleted_at` record from the table to the current timestamp:

```python
User.delete(1)
# == UPDATE `users` SET `deleted_at` = '2020-01-01 10:00:00' WHERE `id` = 1
```

When you fetch records it will also only fetch undeleted records:

```python
User.all() #== SELECT * FROM `users` WHERE `deleted_at` IS NULL
```

You can disable this behavior as well:

```python
User.with_trashed().all() #== SELECT * FROM `users`
```

{% hint style="warning" %}
**You still need to add the `deleted_at` datetime field to your User table for this feature to work.**
{% endhint %}

Hopefully there is a `soft_deletes()` helper that you can use in migrations to add this field quickly.

```python
# user migrations
with self.schema.create("users") as table:
  # ...
  table.soft_deletes()
```

## Updating

You can also update or create records as well:

```python
User.update_or_create({"username": "Joe"}, {
    'active': 1
})
```

If there is a record with the username or "Joe" it will update that record and else it will create the record. 

Note that when the record is created, the two dictionaries will be merged together. So if this code was to create a record it would create a record with both the username of `Joe` and active of `1`.

## Changing Primary Key to use UUID

Masonite ORM also comes with another global scope to enable using UUID as primary keys for your models.

Simply inherit the `UUIDPrimaryKeyMixin` scope:

```python
from masoniteorm.scopes import UUIDPrimaryKeyMixin

class User(Model, UUIDPrimaryKeyMixin):
  # ..
```

You can also define a UUID column with the correct primary constraint in a migration file

```python
with self.schema.create("users") as table:
    table.uuid('id')
    table.primary('id')
```

Your model is now set to use UUID4 as a primary key. It will be automatically generated at creation.

You can change UUID version standard you want to use:

```python
import uuid
from masoniteorm.scopes import UUIDPrimaryKeyMixin

class User(Model, UUIDPrimaryKeyMixin):
  __uuid_version__ = 3
  # the two following parameters are only needed for UUID 3 and 5
  __uuid_namespace__ = uuid.NAMESPACE_DNS
  __uuid_name__ = "domain.com
```

## Casting

Not all data may be in the format you need it it. If you find yourself casting attributes to different values, like casting active to an `int` then you can set it right on the model:

```python
class User(Model):
  __casts__ = {"active": "int"}
```

Now whenever you get the active attribute on the model it will be an `int`.

Other valid values are:

* `int`
* `bool`
* `json`

## Dates

Masonite uses `pendulum` for dates. Whenever dates are used it will return an instance of pendulum. 

If you would like to change this behavior you can override 2 methods: `get_new_date()` and `get_new_datetime_string()`:

The `get_new_date()` method accepts 1 parameter which is an instance of `datetime.datetime`. You can use this to parse and return whichever dates you would like.

```python
class User(Model):

    def get_new_date(self, datetime=None):
        # return new instance from datetime instance.
```

If the datetime parameter is None then you should return the current date.

The `get_new_datetime_string()` method takes the same datetime parameter but this time should return a string to be used in a table.

```python
class User(Model):

    def get_new_datetime_string(self, datetime=None):
        return self.get_new_date(datetime).to_datetime_string()
```



## Events

Models emit various events in different stages of its life cycle. Available events are:

* booting
* booted
* creating
* created
* deleting
* deleted
* hydrating
* hydrated
* saving
* saved
* updating
* updated

### Observers

You can listen to various events through observers. Observers are simple classes that contain methods equal to the event you would like to listen to.

For example, if you want to listen to when users are created you will create a `UserObserver` class that contains the `created` method.

You can scaffold an obsever by running:

```text
python craft observer User --model User
```

> If you do not specify a model option, it will be assumed the model name is the same as the observer name

Once the observer is created you can add your logic to the event methods:

```python
class UserObserver:
    def created(self, user):
        pass

    def creating(self, user):
        pass

    #..
```

The model object receieved in each event method will be the model at that point in time.

You may then set the observer to a specific model. This could be done in a service provider:

```python
from app.models import User
from app.observers.UserObserver import UserObserver
from masonite.providers import Provider

class ModelProvider(Provider):

    def boot(self):
        User.observe(UserObserver())
        #..
```

## Related Records

There's many times you need to take several related records and assign them all the same attribute based on another record.

For example you may have articles you want to switch the authors of.

For this you can use the `associate` and `save_many` methods. Let's say you had a `User` model that had a `articles` method that related to the `Articles` model.

```python
user = User.find(1)
articles = Articles.where('user_id', 2).get()

user.save_many('articles', articles)
```

This will take all articles where user\_id is 2 and assign them the related record between users and article \(user\_id\).

You may do the same for a one-to-one relationship:

```python
user = User.find(1)
phone = Phone.find(30)

user.associate('phone', phone)
```

