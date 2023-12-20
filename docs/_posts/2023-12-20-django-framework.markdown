---
layout: default
title: "Django development/testing/quality setup"
---

Here are some lessons I've learned about managing a Django code base.

## Makefiles

I love Makefiles. I like to have these targets:

  - `skip`: first target; does nothing, so `make` does nothing.
  - `pushup`: runs a bunch of git commands to checkout, push, and merge
  - `superuser`: creates a superuser user in development
  - `linecounts`: shows # of lines of code
  - `mypy`: runs mypy
  - `tests`: runs tests
  - `coverage`: runs coverage
  - `pylint`: runs pylint

## Containerization for a development environment

If you use a containerization tool such as podman, you can have a separate development image/container. For me, the development container `Dockerfile` includes stuff like:

    FROM FIXME-IMAGE-USED-FOR-PROD
	COPY requirements-dev.txt /FIXME/
	
	# whatever command-line utilities help you in debugging:
	RUN apt-get install -y vim netcat-traditional
	RUN pip install -r /FIXME/requirements-dev.txt
	
	ENV DEBUG 1

This ensures that, for the development container, `DEBUG` is always set and I have tools like `vi` available.

I then like to be able to run the development image in two ways:

  - Web app
  - Shell
  
The "shell" version is as easy as using the same `podman run` command but with `-ti` as additional options and then `/bin/bash` as the command.

I mount the entire Django app as a volume, so it's easy to have the development shell container then update the app code as needed.

## modules to install

Here are some useful modules to install in `requirements-dev.txt`:

	coverage
	django-debug-toolbar
	django-stubs
	factory-boy
	mypy
	responses
    pylint
	pylint-django

After you've run `mypy` once, it can help you identify other stub modules to install.

## Development-specific settings

You can have a section in your Django settings that only gets loaded for the development environment. These settings can do things like allow different authentication, set the `EMAIL_HOST` and `EMAIL_PORT` to something that works with a tool like [MailCatcher](https://mailcatcher.me), and add apps such as `django-debug-toolbar`.

## scripts for building development environment data

I like to have a folder in my project that contains any scripts needed for the development environment. This might include scripts that can purge/sanitize production data, in case I clone from production.

For example, I might have a script `setup_db.sh` with stuff in it like:

    python manage.py migrate
	python manage.py loaddata FIXME/important-fixture.json

## managing database migrations

I have ended up using SQL migrations with a special folder in the project for them. Whenever I run `makemigrations` and `migrate` in the development environment, if it looks good then I run `sqlmigrate` and save the output to this folder.

It's possible to then have part of your release process be running these SQL migrations.

This is mainly helpful to be double-super-sure that we know exactly what's going to be changing on the database server.

The unfortunate drawback of this approach is that it doesn't populate the Django migration tables. After cloning you have to be very careful about your code repository state, by going to the current production branch and running `python manage.py migrate --fake` so the database gets populated with the correct set of current Django migrations.

## testing

I like to run testing with a different Django settings file. I call it something like `exampleproject.test_settings` and include this:

	# get the regular settings:
	from .settings import *
	
	TEST_RUNNER = 'my_example_test_runner'
	
	MIGRATION_MODULES = {
		# this makes Django not run through the migrations:
	    'auth': None,
		# add other modules as desired
	}

For the test runner, if you have any unmanaged models, it can be helpful to have a custom test runner. See  https://dev.to/vergeev/testing-against-unmanaged-models-in-django .

All the above said, I can then run the tests with something like:

    DEBUG= DJANGO_SETTINGS_MODULE=exampleproject.test_settings python manage.py test --keepdb

(The `DEBUG=` there resets the `DEBUG` environment variable, which for me is desirable because I configured Django logging to be VERY VERBOSE in the development environment based if that environment variable is set.)

### factories

Ideally we're building tests as we develop. I have found `factory` to be very helpful for dynamic object generation. It lets you do things like:

    class ExampleFactory(factory.django.DjangoModelFactory):
        class Meta:
            model = OriginalModel
    
        name = factory.Faker('name')
        first_name = factory.Faker('first_name')
        last_name = factory.Faker('last_name')
		other_model = factory.SubFactory(OtherModel)

        @factory.post_generation
		def manytomany_field(self, create, extracted, **kwargs):
		    if not create:
			    return
				
			if extracted:
			    for item in extracted:
				    self.manytomany_field.add(item)

You can use this in your tests like so:

    class TestModel(TestCase):
	    def setUp(self):
		    self.example1 = ExampleFactory()
			self.long_last_name = ExampleFactory(last_name='X'*60)

### testing external APIs

`responses` is very helpful for testing external APIs. It lets you control what happens when your test case attempts to contact an external resource. You can do things like:

    responses.add(**{
	    'method': responses.GET,
		'url': 'http://fixme/url/to/override',
		'body': b'This is the body content that will be returned',
		'status': 200,
	})

Then for each test method you want to use, you add a decorator:

    @responses.activate
	def setUp(self):
	    # stuff that uses http://fixme/url/to/override goes here


## coverage

Use `coverage` to check what code is/isn't covered by unit tests. First, build a `.coveragerc` file, e.g.:

    [coverage:run]
    source = .
    omit =
      manage.py
      */migrations/*
      *_flymake.py
    
    [coverage:report]
    fail_under = 100
    show_missing = True
    skip_covered = True

Use as:

    coverage erase
	DEBUG="" coverage run manage.py test --keepdb
	coverage report

## mypy

Use `dmypy run`, which starts a mypy daemon if it's not already installed. Use as:

    dmypy run --exclude migrations .
	
## pylint

I'm still working on a useful pylint structure. I've got this really basic command:

    find exampleproject -name '*.py' -not -path "*/migrations/*" -print0 | xargs -0 pylint --load-plugins pylint_django

which generates a whole bunch of pylint output.

## Sphinx/documentation

If you want to use Sphinx to develop documentation, and _especially_ if you want to create a PDF, it can help to use a pre-built Sphinx image. I found the easiest way to do this is to create a Sphinx `Dockerfile` that looks something like this:

    FROM docker.io/sphinxdoc/sphinx-latexpdf
	# may need tools for pip to use, e.g. `git` or `gcc`
	RUN apt-get update -y && apt-get -y install FIXME
	RUN pip install -U pip commonmark
	COPY . /FIXME/
	ENV PYTHONPATH /FIXME

I have found it helpful to build Sphinx "stubs" for each application that look like this:

    MODULENAME
	====
	
	General description
	
	Gotchas
	----
	
	- a
	- b
	
	Links to other apps/code
	----
	FIXME
	
	URLs
	----
	``/path/`` is the root URL.
	
	Models
	----
	
	.. automodule: example.models
	               :members:
				   
	Settings used
	-------------
	
	:SETTING_1: Description.
	:SETTING_2: Description.
	
That said, I'm not very knowledgeable about Sphinx/reStructuredText formatting.

If you use a template like the above, you will need to run Sphinx to generate the `automodule` content, which I leave as an exercise to the reader.

I can then build/run this image against the codebase to build whatever's in `docs/`:

    podman run --rm -v "/path/to/docs/:/docs" my-sphinx-image /bin/bash -c "make latexpdf; make latexpdf"
