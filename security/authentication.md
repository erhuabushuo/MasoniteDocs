# Authentication

## Authentication

## Introduction

Masonite comes with some authentication out of the box. Nothing is too opinionated to be restrictive so Masonite leaves it really customizable but gives you a great starting point.

The concept of authentication is a "Guard" concept. Guards are simply authentication logic for different scenarios. For example if you are building an API you might have an `api` guard but if you are building for a normal web app you might use a `web` guard. You can put different routes behind different guards to give you an ultimate level of control.

## Configuration

There is a single `config/auth.py` configuration file which you can use to set the authentication behavior of your Masonite project. The default configuration looks something like:

```python
AUTH = {
    'defaults': {
        'guard': env('AUTH_GUARD', 'web')
    },
    'guards': {
        'web': {
            'driver': 'cookie',
            'model': User,
            'drivers': { # 'cookie', 'jwt'
                'jwt': {
                    'reauthentication': True,
                    'lifetime': '5 minutes'
                }
            }
        },
    }
}
```

You can set the default guard to use in the `defaults` key. From there you can set various different settings based on which guard you need inside the `guards` key. Each guard takes a few different settings like the `driver` the guard will use, the `model`, and then `driver` settings here.

If you wish to change the authentication model, to a `Company` model for example, feel free to do in this configuration file.

This would look something like:

```python
from app.Company import Company

AUTH = {
    'defaults': {
        'guard': env('AUTH_GUARD', 'web')
    },
    'guards': {
        'web': {
            'driver': 'cookie',
            'model': Company,
            'drivers': { # 'cookie', 'jwt'
                'jwt': {
                    'reauthentication': True,
                    'lifetime': '5 minutes'
                }
            }
        },
    }
}
```

## Cookie Driver

The cookie driver will set a token as a cookie and then fetch the user from the database on every request. For most applications this is fine although you are making an additional query per request just to fetch the user.

This is the most basic authentication driver.

## JWT Driver

The JWT driver will store an encrypted JWT token inside a cookie with all the authenticated user information. Then when the authenticated user goes to the page, the JWT token is decrypted and fills in the data on the user model without having to recall the database. Use this option for high traffic sites that you don't want making that extra database call on every request.

There are also 2 options you can set as well. The first option is how long until the jwt token expires. By default this is 5 minutes but you can extend it out longer:

```python
    'jwt': {
        'reauthentication': True,
        'lifetime': '5 minutes'
    }
```

The second option is whether or not the user should reauthenticate with the database after their token has expired. If set to `False`, the token will simply continue to refill the user model and set a new token all without touching the database. If set to `True` it will cancel out the token and reauthenticate with the database to create a new token.

## Authentication Model

Again the default authentication model is the `app/User` model which out of the box comes with a `__auth__` class attribute. This attribute should be set to the column that you want to authenticate with when a user logs in.

### Authentication Column

By default your `app/User` model will default to the `email` column but if you wish to change this to another column such as `name`, you can do so here. This will lead your model to look like:

```python
class User(Model):

    __fillable__ = ['name', 'email', 'password']

    __auth__ = 'name'
```

### Multiple Authentication Columns

Sometimes your application will be able to either login by email OR by username. You can do this by specifying the `__auth__` attribute as a list of columns:

```python
class User(Model):

    __fillable__ = ['name', 'email', 'password']

    __auth__ = ['name', 'email']
```

Now you can do something like "Please enter Username or Email".

### Password Column

By default, Masonite will use the `password` column to authenticate as the password. Some applications may have this changed. Your specific application may be authenticating with a `token` column for example.

You can change the password column by speciyfing the `__password__` attribute to the column name:

```python
class User(Model):

    __fillable__ = ['name', 'email', 'password']

    __auth__ = ['name']
    __password__ = 'token'
```

## Authenticating a Model

If you want to authenticate a model, you can use the `Auth` class that ships with Masonite. This is simply a class that is used to authenticate models with a `.login()` method.

In order to authenticate a model this will look like:

```python
from masonite.auth import Auth

def show(self, request: Request, auth: Auth):
    auth.login('user@email.com', 'password')
```

This will find a model with the supplied username, check if the password matches using `bcrypt` and return the model. If it is not found or the password does not match, it will return `False`.

{% hint style="warning" %}
Again all authenticating models need to have a `password` column. The column being used to authenticate, such as a username or email field can be specified in the model using the `__auth__` class attribute.
{% endhint %}

### Changing The Authentication Column

You may change the column to be authenticated by simply changing the column value of the `__auth__` class attribute. This will look something like:

```python
class User(Model):

    __fillable__ = ['name', 'email', 'password']

    __auth__ = 'email'
```

This will look inside the `email` column now and check that column and password. The authentication column is `email` by default.

### Changing The Guard

The above examples will use the default guard. You can change the guard on the fly by doing:

```python
from masonite.auth import Auth

def show(self, request: Request, auth: Auth):
    auth.guard('api').login('user@email.com', 'password')
```

This will now use the `api` guard \(if one existed\).

### Changing the Guard in Routes

Masonite ships with a guard middleware which allows you to switch the guard based on the route. For example if you had an API route you might want to use an `api` guard you have created:

```python
Get('/api/users').middleware('guard:api')
```

Or you may want to use route groups:

```python
RouteGroup([
    Get('/api/users'),
    Get('/api/settings'),
    Get('/api/posts'),
], middleware=('guard:api',))
```

### Creating an Authentication System

You may of course feel free to roll your own authentication system if you so choose but Masonite comes with one out of the box but left out by default. In order to scaffold this authentication system you can of course use a `craft` command:

```text
$ python craft auth
```

This will create some controllers, views and routes for you. This command should be used primarily on fresh installs of Masonite but as long as the controllers do not have the same names as the controllers being scaffolded, you will not have any issues.

The views scaffolded will be located under `resources/templates/auth`.

After you have ran the `craft auth` command, just run the server and navigate to `http://localhost:8000/login` and you will now have a login, registration and dashboard. Pretty cool, huh?

### Retrieving the Authenticated User

Masonite ships with a `LoadUserMiddleware` middleware that will load the user into the request if they are authenticated. Masonite uses the `token` cookie in order to retrieve the user using the `remember_token`column in the table.

Using this `LoadUserMiddleware` middleware you can retrieve the current user using:

```python
def show(self, request: Request):
    request.user()
```

If you wish not to use middleware to load the user into the request you can get the request by again using the `Auth` class

```python
from masonite.auth import Auth

def show(self, request: Request, auth: Auth):
    auth.user()
```

#### Checking if the User is Authenticated

If you would like to simply check if the user is authenticated, `request.user()` or `auth.user()` will return `False` if the user is not authenticated. This will look like:

```python
def show(self, request: Request):
    if request.user():
        user_email = request.user().email
```

### Logging In

You can easily log users into your application using the Auth class:

```python
from masonite.auth import Auth

def show(self, request: Request, auth: Auth):
    auth.login(
        request.input('username'),
        request.input('password')
    )
```

{% hint style="info" %}
Note that the username you supply needs to be in whatever format the `__auth__` attribute is on your model. If the email address is the "username", then the user will need to supply their email address.
{% endhint %}

### Login By ID

If you need more direct control internally, you can login by the models ID:

```python
from masonite.auth import Auth

def show(self):
    auth.login_by_id(1)
```

You are now logged in as the user with the ID of 1.

### Login Once

If you only want to login "once", maybe for just authenticating an action or verifying the user can supply the correct credentials, you can login without saving any cookies to the browser:

```python
from masonite.auth import Auth

def show(self):
    auth.once().login_by_id(1)
```

You can do the same for the normal login method as well:

```python
from masonite.auth import Auth

def show(self, request: Request, auth: Auth):
    auth.once().login(
        request.input('username'),
        request.input('password')
    )
```

### Registering a User

You can also easily register a user using the `register()` method on the `Auth` class:

```python
from masonite.auth import Auth

def show(self, auth: Auth):
    auth.register({
        'name': 'Joe',
        'email': 'joe@email.com',
        'password': 'secret'
    })
```

### Protecting Routes

Masonite ships with an authentication middleware. You can use this middleware as a route middleware to protect certain routes from non authenticated users. This is great for redirecting users to a login page if they attempt to go to their dashboard.

You can use this middleware in your routes file like so:

```python
Get().route('/dashboard', 'DashboardController@show').middleware('auth')
```

By default this will redirect to the route named `login`. If you wish to redirect the user to another route or to a different URI, you can edit the middleware in `app/http/middleware/AuthenticationMiddleware.py`

### Logging Out a User

If you wish to end the session for the user and log them out, you can do so by using the `Auth` class. This looks like:

```python
auth.logout()
```

This will delete the cookie that was set when logging in. This will not redirect the user to where they need to go. A complete logout view might look like:

```python
def logout(self, request: Request, auth: Auth):
    auth.logout()
    return request.redirect('/login')
```

### Verifying A User's Email

If you wish to require a user to verify their email address and automatically send them an email, you can extend the `User` model.

```python
from masonite.auth import MustVerifyEmail

class User(Model, MustVerifyEmail):

    __fillable__ = ['name', 'email', 'password']

    __auth__ = 'name'
```

When a user registers this will automatically send them an email asking them to confirm their email address.

#### Redirecting Unverified User's

You can use the `VerifyEmailMiddleware` class to redirect an unverified user.

You can use this middleware in your routes file like so:

```python
Get().route('/dashboard', 'DashboardController@show').middleware('verified')
```

Great! You’ve mastered how Masonite uses authentication. Remember that this is just out of the box functionality and you can create a completely different authentication system but this will suffice for most applications.

