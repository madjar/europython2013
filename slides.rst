:title: Pyramid advanced configuration tactics for nice apps and libs
:author: Georges Dubus
:css: europython.css

----

Pyramid advanced configuration tactics for nice apps and libs
=============================================================

Europython 2013
---------------

.. Default x increment : 1600

----

Hi, I'm Georges
===============

----

Quick question
==============

----

Once upon a time ...

Persona is the awesome web authentication system by Mozilla that will
save us all.

.. image:: persona_sign_in_blue.png

Using it requires to add some javascript to your page and
to implement a few stuff on server side.

I included it in a few apps

Then, I made a library to do that in Pyramid : pyramid_persona.

.. note::
   
   Holger talked about it.

----

What I'm going to talk about
----------------------------

- The configuration system of Pyramid
- How to use it to make libraries, and nicely organized applications

.. note::

   The result is a nice, easy to use library, because pyramid is
   well-designed for extensions

   There was a few non-trivial stuff, and a some of research involved.

   2 parts

   - In the first, I'll introduce you to the configuration system so
     you get an idea of what we are dealing with.
   - In the second part, I'll show how to use it for extensions and
     organizing apps.

----

What's pyramid again ?
----------------------

Web framework

- Simplicity
- Minimalism
- Documentation
- Speed
- Reliability
- Openness

.. note::

   Here are the keywords from pyramid introduction

..
   ----

   Why I like pyramid
   ------------------

   - Let me use SQLAlchemy
   - Provide two ways to map url to views : dispatch and traversal, and traversal is neat.

----

Hello World
-----------


.. code:: python

   def hello_world(request):
       return Response('Whoa there!')

   if __name__ == '__main__':
       config = Configurator()
       config.add_route('home', '/')
       config.add_view(hello_world, route_name='home')
       app = config.make_wsgi_app()
       server = make_server('0.0.0.0', 8080, app)
       server.serve_forever()

.. note::

   Here is a basic application. The part that's going to interest us
   is in the bottom.

----

Meet the configurator
---------------------

``pyramid.config.Configurator``

.. code:: python

   config = Configurator()
   config.add_route('home', '/')
   config.add_view(hello_world, route_name='home')
   app = config.make_wsgi_app()

That where you configure everything: views, authentication, renderers, session, …

Its methods are called directives.

.. note::

   The configurator is used in every pyramid application, the put
   together all the bits

----

The order of the directives is not significant. They're just added to
a pending list that is treated on commit.

.. code:: python
   
   config = Configurator()
   config.add_view(hello_world, route_name='home') # moved this up
   config.add_route('home', '/')
   app = config.make_wsgi_app()

.. note::

   You can move directives around, the order doesn't matter.

   Everything is resolved when the config is committed. `make_wsgi_app`
   does a commit.

----

Decorators with no import-time effects

.. code:: python

   @view_config(route_name='home')
   def view(request):
       return Response('Halt!  Who goes there?')

   config = Configurator()
   config.add_route('home', '/')
   config.scan()

----

:data-x: 0
:data-y: r2000

Sanity checks
-------------

The configurator checks that you didn't mess up.

.. code:: python

    config = Configurator()
    config.add_view(hello_world, route_name='home')
    # config.add_route('home', '/')
    app = config.make_wsgi_app() 

::

   pyramid.exceptions.ConfigurationExecutionError:
   No route named home found for view registration
     in:
       Line 10 of file app.py:
         config.add_view(hello_world, route_name='home')

.. note::

   Here, we try to add a view to a route that does not exists. When we
   commit, the configurator will tell us "nope, you can't".

   This is done for a lot of others directives, like authorization
   that require authentication.

----

:data-x: r1600
:data-y: r0

The configurator checks for conflicts. It doesn't let you overwrite by
accident.

.. code:: python

    config = Configurator()
    config.add_route('home', '/')
    config.add_view(hello_world, route_name='home')
    config.add_view(hi_world, route_name='home') # added
    app = config.make_wsgi_app()

::

   pyramid.exceptions.ConfigurationConflictError:
   Conflicting configuration actions
   For: ('view', None, '', 'home', 'd41d8cd98f00b204e9800998ecf8427e')
     Line 14 of file app.py:
         config.add_view(hello_world, route_name='home')
     Line 15 of file app.py:
         config.add_view(hi_world, route_name='home')

.. note::

   We try to define two conflicting views, the configurator detects
   it, and doesn't silently discard one.

..
   ----

   Solving conflicts:

   .. code:: python

      config = Configurator()
      config.add_route('home', '/')
      config.add_view(hello_world, route_name='home')
      config.commit() # added

      config.add_view(hi_world, route_name='home') # added
      app = config.make_wsgi_app()

   .. note::

      A commit checks conflicts among pending directives. So, two
      directives are in conflict only if they are in the same commit.

      What's the use of this ? I'll get to it.

----

:data-x: 0
:data-y: r800

Modular configuration: `include`
--------------------------------

.. code:: python

   def moreconfiguration(config):
       config.add_route('goodbye', '/goodbye')
       config.add_view(goodbye, route_name='goodbye')


   config = Configurator()
   config.add_route('home', '/')
   config.add_view(hello_world, route_name='home')
   config.include(moreconfiguration)
   app = config.make_wsgi_app()

.. note::

   It is possible to include a callable. It looks like a simple
   function call, but there is a few differences.

----

:data-x: r1600
:data-y: r0

Not a simple function call

.. code:: python

   config.include(moreconfiguration, route_prefix='/other')

.. note:: All the routes defined in moreconfiguration will have the prefix.

----

Really not a simple function call: solving conflicts

.. code:: python

   def moreconfiguration(config):
       config.add_route('hello', '/hello')
       config.add_view(hello_world, route_name='hello')


   config = Configurator()
   config.add_view(hi_world, route_name='hello')  # This directives wins
   config.include(moreconfiguration)
   app = config.make_wsgi_app()

.. note::

   Top level is more important than what's included. Top level
   overrides the rest.

----

It means you have a way to solve conflicts between libraries

.. code:: python

   def some_config(config):
       config.add_view(some_view, route_name='hello')

   def more_config(config):
       config.add_view(some_other_view, route_name='hello')


   config = Configurator()
   config.add_route('hello', '/hello')
   config.include(some_config)
   config.include(more_config)

   config.add_view(some_view, route_name='hello')
   app = config.make_wsgi_app()

----

:data-x: 0
:data-y: r800


The `includeme` convention
--------------------------


.. code:: python

   import pyramid_awesomeness
   config.include(pyramid_awesomeness.includeme)

Is equivalent to

.. code:: python

   config.include('pyramid_awesomeness.includeme')

Also equivalent to

.. code:: python

   config.include('pyramid_awesomeness')

- `include` can be used on dotted names.
- If we include a module, pyramid looks for the `includeme` function in it.

.. note::

   This means we can include a package without worrying on what's inside.

----

:data-x: r1600
:data-y: r0

python package + includeme function = pyramid extension
-------------------------------------------------------

- Libraries can do everything that is possible in pyramid
- Libraries can set default that can be overriden

----

Example: `pyramid_persona` sets some default
authentication/authorization policy

.. code:: python

   # in pyramid_persona
   def includeme(config):
       authz_policy = ACLAuthorizationPolicy()
       config.set_authorization_policy(authz_policy)
       secret = settings.get('persona.secret', None)
       authn_policy = AuthTktAuthenticationPolicy(secret, hashalg='sha512')
       config.set_authentication_policy(authn_policy)


Easily overriden

.. code:: python

   config.include('pyramid_persona')
   authn_policy = AuthTktAuthenticationPolicy(settings['persona.secret'],
                                              hashalg='sha512',
                                              max_age=60*60*24*30)
   config.set_authentication_policy(authn_policy)

.. note::

   For example, pyramid_persona defines some default authentication
   and authorization policy, for the convenience.

   I might want to use another one, or the same one with different
   parameters. I just have to include it, and call the directives I
   want.

   There's nothing the library writer can do that would reduce the
   possibilities of the user.

----

Higher order stuff: a directive to add directives

(with conflicts detection!)

.. code:: python

   # in pyramid_awesomeness.includeme
   def set_awesomeness_level(config, level):
       def callback():
           config.registry.awesomeness_level = level
       discriminator = ('set_awesomeness_level',)
       config.action(discriminator, callback=callback)

   config.add_directive('set_awesomeness_level', set_awesomeness_level)

In my application:

.. code:: python

   config.include('pyramid_awesomeness')
   config.set_awesomeness_level(42)

.. note::

   A directive consist of a directive that is discriminator that is
   used to detect conflicts (two directive calls with the same
   discriminator are in conflict), and a callback : the actual stuff
   that is done.

   Here, we have a directive to set the awesomeness level. It has
   conflict detection.

----

:data-scale: 3
:data-x: -1800
:data-y: r-800

Summary
-------

We have a system to delegate configuration and to check that
everything is sound.

A great way to organize the configuration of an application.

A great way to make libraries.

.. note::

   You can put different parts of your app in different places and
   include them all.

   In the end, there is no difference between a modularized
   application and a library : the exacts same tools are used.

   Except that you have to find a name for the library (and write
   documentation).

----

:data-scale: 1
:data-y: r1600

Meanwhile, using django
-----------------------

When you import something, you have to configure every hook by hand.

.. code:: python

   INSTALLED_APPS
   AUTHENTICATION_BACKENDS
   TEMPLATE_CONTEXT_PROCESSORS
   urls.py
   ...

----

:data-x: r1600
:data-y: r0

So, what can I do with all this ?
---------------------------------

Tips, examples, recipes, ...

For applications and libraries

Examples of ways to use config in an application or a library.

.. ::


   TODO : ou alors séparer appli et lib, et mettre un résumé global à la
   fin de appli (un exemple d'appli avec des petits bouts de config partout)


----

For your application
--------------------

----

Using add_directive to simplify the config
------------------------------------------

.. code:: python

   # A very simple application, with only one view per route
   config.add_route('route1', '/')
   config.add_view(view1, route_name='route1')
   config.add_route('route2', '/stuff')
   config.add_view(view2, route_name='route2')
   config.add_route('route3', '/otherstuff')
   config.add_view(view3, route_name='route3')
   # And so on

----

Transformed to:

.. code:: python
   
   def add_simple_view(config, view, path):
       def callback():
           route_name = view.__qualname__
           config.add_route(route_name, path)
           config.add_view(view, route_name=route_name)
       discriminator = ('add_simple_view', path)
       config.action(discriminator, callback)

   config.add_directive('add_simple_view', add_simple_view)
   config.add_simple_view(view1, '/')
   config.add_simple_view(view2, '/stuff')
   config.add_simple_view(view3, '/otherstuff')

----

What if I want a decorator ?
----------------------------

Use venusian decorators, they are detected by config.scan()

.. code:: python

   class simple_view(object):
       def __init__(self, path):
           self.path = path

       def register(self, scanner, name, wrapped):
	   scanner.config.add_simple_view(wrapped, self.path)

       def __call__(self, wrapped):
           venusian.attach(wrapped, self.register)
	   return wrapped

   @simple_view('/')
   def view(request):
       return Response('It is I, Arthur, son of Uther Pendragon, ...')

----

Adding methods to request
-------------------------

It'd be nice if request.user was the user object for the current user.

.. code:: python

   def get_user(request):
       id = authenticated_userid(request)
       if not id:
           return None
       return DBSession.query(User).get(id)


   config.add_request_method(get_user, 'user', reify=True)

Reified means the method is replaced by the object after the first call.


----

Events
------

Pyramid has an event system.

.. code:: python

   @subscriber(EventClass)
   def do_stuff(event):
       pass

- BeforeRender to add globals to the templates
- NewRequest, NewResponse when a request/response is created
- ContextFound when traversal is done
- ApplicationCreated

----

Adding globals to the templates
-------------------------------

.. code:: python

   @subscriber(BeforeRender)
   def add_helper(event):
       from . import helpers
       event['h'] = helpers

In my template:

.. code:: mako

   ${h.format_date(date)}


----

A lot of other things can be changed
------------------------------------

- How url are generated
- How view responses are handled
- How requests are mapped to views
- Sessions, authentication, renderers

----

Libraries
---------

.. note::

   All of the above is still usable

----

More globals to templates
-------------------------

.. code:: python

   @subscriber(BeforeRender)
   def add_renderer_global(event):
       event['persona_js'] = get_persona_js(event.request)
       # persona_js is available in the template

----

Using add_directive to add an entry point
-----------------------------------------

pyramid_layout extends pyramid with layouts and panels.

To the user, it is seamless.

.. code:: python

   config = Configurator(...)
   config.include('pyramid_layout')
   config.add_layout(...)
   config.add_panel(...)

.. note::

   add_directive is used in library to add new configuration
   directives for the user.

   Once added, they are no different from pyramid's directives.

----

add_view_predicate
------------------

.. code:: python

   config.add_view(view, route_name='hello',
                         request_method='POST')
   # route_name and request_method are view predicates

Used to decide whether a request matches a view.

Can also do extra work before the view is executed.

----

Example in pyramid_layout

.. code:: python

   class LayoutPredicate(object):
       def __init__(self, val, config):
           self.val = val

       def text(self):
           return 'layout = %s' % self.val

       phash = text

       def __call__(self, context, request):
           request.layout_manager.use_layout(self.val)
	   return True

   # In the includeme
   config.add_view_predicate('layout', LayoutPredicate)
   
   # In user's code
   config.add_view(view, route_name='hello',
                         layout='some_layout')

----

Tweens
------

Some code around pyramid's request handling.

Used by pyramid_debugtoolbar and pyramid_exclog to catch exceptions in
the application.

----

How to handle global stuff
--------------------------

Global stuff like database connections, ...

- registry (config.registry and request.registry)
- in the settings dict

.. note::

   Two clans

.. ::

   Bonus points : ZCA
   TODO, après avoir testé la durée de la présentation.


----

Building upon pyramid
---------------------

Example : cornice

.. code::

   user_info = Service(name='users', path='/{username}/info',
		       description=info_desc)

   @user_info.get()
   def get_info(request):
       username = request.matchdict['username']
       return _USERS[username]

   @user_info.post()
   def set_info(request):
       ...

----

- config.add_cornice_service

- venusian callback attached to Service, so service is caught by config.scan

----

Conclusion
----------

.. note::

   That's it.

   Pyramid is really customizatible. If one day, you need something
   that doesn't fit the django way, think about this, and come back to
   find the pointers that are here.

----

References
----------

- http://docs.pylonsproject.org/projects/pyramid_cookbook/en/latest/configuration/whirlwind_tour.html
- http://docs.pylonsproject.org/projects/pyramid/en/1.4-branch/narr/advconfig.html
- http://docs.pylonsproject.org/projects/pyramid/en/1.4-branch/narr/hooks.html

.. note::

   Parts of this presentation were heavily inspired by the
   documentation and the cookbook

----

Thanks
------

@georgesdubus

madjar.github.io/europython2013
