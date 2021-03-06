==============
Initialization
==============

Sanic JWT operates under the hood by creating a `Blueprint <http://sanic.readthedocs.io/en/latest/sanic/blueprints.html>`_, and attaching a few routes to your application. This is accomplished by instanciating the ``Initialize`` class.

.. code-block:: python

    from sanic_jwt import Initialize
    from sanic import Sanic

    async def authenticate(request):
        return dict(user_id='some_id')

    app = Sanic()
    Initialize(app, authenticate=authenticate)

------------

+++++++
Concept
+++++++

Sanic JWT is a user authentication system that does not require the developer to settle on any single user management system. This part is left up to the developer. Therefore, you (as the developer) are left with the responsibility of telling Sanic JWT how to tie into your user management system.

------------

++++++++++++++++++++++++
The ``Initialize`` class
++++++++++++++++++++++++

This is the gateway worker into Sanic JWT. When initialized, it allows you to pass run time configurations to it, and gives you a window into customizing how the module will work for you. There are **five main parts** to it when initializing:

- the **instance** of your Sanic app or a Sanic blueprint | **REQUIRED**
- handler methods | the ``authenticate`` handler **REQUIRED**
- any runtime configurations you want to make
- custom view classes
- component overrides

--------
Instance
--------

In most cases, the developer wants to add authentication to their web application. Simply instantiate Sanic, and then tell Sanic JWT.

.. code-block:: python

    from sanic_jwt import Initialize
    from sanic import Sanic

    app = Sanic()
    Initialize(app, authenticate=lambda: True)

You can now go ahead and :doc:`protect<protected>` any route (whether on a blueprint or not).

.. code-block:: python

    from sanic_jwt import protected
    from sanic.response import json

    ...

    @app.route("/")
    @protected
    async def test(request):
        return json({ "protected": True })

What if we ONLY want the authentication on some subset of our web application? Say, a `Blueprint <http://sanic.readthedocs.io/en/latest/sanic/blueprints.html>`_. Not a problem. Just initialize on the blueprint instance and continue as normal.

.. code-block:: python

    from sanic_jwt import Initialize
    from sanic import Sanic, Blueprint

    app = Sanic()
    bp = Blueprint('my_blueprint')
    Initialize(app, authenticate=lambda: True)
    app.blueprint(bp)

.. warning::

    If you are initializing on a blueprint, be careful of the ordering of ``app.blueprint()`` and ``Initialize``. Putting them in the wrong order will cause the authentication endpoints to not properly attach.

.. note::

    If you decide to initialize more than one instance of Sanic JWT (on multiple blueprints, for example), than an access token generated by one will be acceptable on **ALL** your instances unless they have different a ``secret``.

Under the hood, Sanic JWT creates its own ``Blueprint`` for holding all of the :doc:`endpoints`. If you decide to use your own blueprint, just know that Sanic JWT will not create its own, and instead attach to the one that you passed to it.

This is a very powerful tool that allows you to really gain some granularity in your applications authentication system.

.. code-block:: python

    async def authenticate(request, *args, **kwargs):
        return get_my_user()

    app = Sanic()
    bp1 = Blueprint('my_blueprint_1')
    bp2 = Blueprint('my_blueprint_2')

    Initialize(app, authenticate=authenticate)
    Initialize(bp1, authenticate=authenticate, access_token_name='mytoken')
    Initialize(bp2, authenticate=authenticate, access_token_name='yourtoken')

In the above example, I now have three independent instances of Sanic JWT running side by side. Each is isolated to its own environment.

--------
Handlers
--------

There is a set of methods that Sanic JWT uses to hook into your application code. Each of them can be either a method or an awaitable. You decide.

.. code-block:: python

    # This works
    async def authenticate(request, *args, **kwargs):
        ...

    # And so does this
    def authenticate(request, *args, **kwargs):
        ...

~~~~~~~~~~~~~~~~~~~~~~~~~~~
``authenticate`` - Required
~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Purpose**: Just like Django's ``authenticate`` `method <https://docs.djangoproject.com/en/2.0/ref/contrib/auth/#django.contrib.auth.backends.ModelBackend.authenticate>`_, this is responsible for taking a given ``request`` and deciding whether or not there is a valid user to be authenticated. If yes, it **MUST** return:

- a ``dict`` with a ``user_id`` key, **or**
- an instance with an id and ``to_dict`` property.

By default, it looks for the id on the ``user_id`` property of a user instance. However, you can :doc:`change that to another property<configuration>`.

If your user should **not** be authenticated, then you should :doc:`raise an exception<exceptions>`, preferably ``AuthenticationFailed``.

**Example**:

.. code-block:: python

    async def authenticate(request, *args, **kwargs):
        username = request.json.get('username', None)
        password = request.json.get('password', None)

        if not username or not password:
            raise exceptions.AuthenticationFailed("Missing username or password.")

        user = await User.get(username=username)
        if user is None:
            raise exceptions.AuthenticationFailed("User not found.")

        if password != user.password:
            raise exceptions.AuthenticationFailed("Password is incorrect.")

        return user

    Initialize(app, authenticate)


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
``store_refresh_token`` - Optional \*
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Default**: ``None``

**Purpose**: It is a handler to persist a refresh token to disk. See `refresh tokens <refreshtokens>`_ for more information.

**Example**:

.. code-block:: python

    async def store_refresh_token(user_id, refresh_token, *args, **kwargs):
        key = 'refresh_token_{user_id}'.format(user_id=user_id)
        await aredis.set(key, refresh_token)

    Initialize(
        app,
        authenticate=lambda: True,
        store_refresh_token=store_refresh_token)

.. warning:: \* This parameter is *not* required. However, if you decide to enable refresh tokens (by setting ``SANIC_JWT_REFRESH_TOKEN_ENABLED=True``) then the application will raise a ``RefreshTokenNotImplemented`` exception if you forget to implement this.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
``retrieve_refresh_token`` - Optional \*
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Default**: ``None``

**Purpose**: It is a handler to retrieve refresh token from disk. See `refresh tokens <refreshtokens>`_ for more information.

**Example**:

.. code-block:: python

    async def retrieve_refresh_token(user_id, *args, **kwargs):
        key = 'refresh_token_{user_id}'.format(user_id=user_id)
        return await aredis.get(key)

    Initialize(
        app,
        authenticate=lambda: True,
        retrieve_refresh_token=retrieve_refresh_token)

.. warning:: \* This parameter is *not* required. However, if you decide to enable refresh tokens (by setting ``SANIC_JWT_REFRESH_TOKEN_ENABLED=True``) then the application will raise a ``RefreshTokenNotImplemented`` exception if you forget to implement this.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~
``retrieve_user`` - Optional
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Default**: ``None``

**Purpose**: It is a handler to retrieve a user object from your application. It is used to return the user object in the ``/auth/me`` `endpoint <endpoints>`_. It should return:

- a ``dict``, **or**
- an instance with a ``to_dict`` method.

**Example**:

.. code-block:: python

    class User(object):
        ...

        def to_dict(self):
            properties = ['user_id', 'username', 'email', 'verified']
            return {prop: getattr(self, prop, None) for prop in properties}

    async def retrieve_user(request, payload, *args, **kwargs):
        if payload:
            user_id = payload.get('user_id', None)
            user = await User.get(user_id=user_id)
            return user
        else:
            return None

    Initialize(
        app,
        authenticate=lambda: True,
        retrieve_user=retrieve_user)

You should now have an endpoint at ``/auth/me`` that will return a serialized form of your currently authenticated user. ::

    {
        "me": {
            "user_id": "4",
            "username": "joe",
            "email": "joe@joemail.com",
            "verified": true
        }
    }

.. warning:: \* This parameter is *not* required. However, if you decide to enable refresh tokens (by setting ``SANIC_JWT_REFRESH_TOKEN_ENABLED=True``) then the application will raise a ``RefreshTokenNotImplemented`` exception if you forget to implement this.


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
``add_scopes_to_payload`` - Optional \*
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Default**: ``None``

**Purpose**: It is a handler to add scopes to an access token. See :doc:`scoped` for more information.

**Example**:

.. code-block:: python

    async def add_scopes_to_payload(user):
        return await user.get_scopes()

    Initialize(
        app,
        authenticate=lambda: True,
        add_scopes_to_payload=add_scopes_to_payload)

---------------------
Runtime Configuration
---------------------

There are several ways to :doc:`configure the settings<configuration>` for Sanic JWT. One of the easiest is to simply pass the configurations as keyword objects on Initialize.

.. code-block:: python

    Initialize(
        app,
        access_token_name='mytoken',
        cookie_access_token_name='mytoken',
        cookie_set=True,
        user_id='id',
        claim_iat=True,
        cookie_domain='example.com',)

----------------
Additional Views
----------------

Sometimes you may need to add some endpoints to the authentication system. When this need arises, create a `class based view <http://sanic.readthedocs.io/en/latest/sanic/class_based_views.html#class-based-views>`_, and map it as a tuple with the path and handler.

As an example, perhaps you would like to create a "passwordless" login. You could create a form that sends a POST with a user's email address to a ``MagicLoginHandler``. That handler sends out an email with a link to your ``/auth`` endpoint that makes sure the link came from the email.

.. code-block:: python

    class MagicLoginHandler(HTTPMethodView):
        async def options(self, request):
            return response.text('', status=204)

        async def post(self, request):
            helper = MyCustomUserAuthHelper(app, request)
            token = helper.get_make_me_a_magic_token()
            helper.send_magic_token_to_user_email()

            # Persist the token
            key = f'magic-token-{token}'
            await app.redis.set(key, helper.user.uuid)

            response = {
                'magic-token': token
            }
            return json(response)

    def check_magic_token(request):
        token = request.json.get('magic_token', '')
        key = f'magic-token-{token}'

        retrieval = await request.app.redis.get(key)
        if retrieval is None:
            raise Exception('Token expired or invalid')
        retrieval = str(retrieval)

        user = User.get(uuid=retrieval)

        return user

    Initialize(
        app,
        authenticate=check_magic_token,
        class_views=[
            ('/magic-login', MagicLoginHandler)     # The path will be relative to the url prefix (which defaults to /auth)
        ])

.. note:: Your class based views will probably also need to handle preflight requests, so do not forget to add an options response.

    .. code-block:: python

        async def options(self, request):
            return response.text('', status=204)

-------------------
Component Overrides
-------------------

There are **three** components that are used under the hood that you can subclass and control:

- ``Authentication`` - for more advanced usage, see source code
- ``Configuration`` - see :doc:`configuration` for more information
- ``Responses`` - see :doc:`endpoints` for more information

Simply import, modify, and attach.

.. code-block:: python

    from sanic_jwt import Authentication, Configuration, Responses, Initialize

    class MyAuthentication(Authentication):
        pass

    class MyConfiguration(Configuration):
        pass

    class MyResponses(Responses):
        pass

    Initialize(
        app,
        authentication_class=MyAuthentication,
        configuration_class=MyConfiguration,
        responses_class=MyResponses,)

------------

+++++++++++++++++++++++++
The ``initialize`` method
+++++++++++++++++++++++++

The old method for initializing Sanic JWT was to do so with the ``initialize`` method. It still works, and is in fact now just a wrapper for the ``Initialize`` class. However, it is recommended that you use the class because it is more explicit that you are declaring a new instance.
