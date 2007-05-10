==============
Django Tagging
==============

A generic tagging application for Django projects, which allows
association of a number of tags with any ``Model`` instance and makes
retrieval of tags simple.


Installation
============

Google Code recommends doing the Subversion checkout like so::

    svn checkout http://django-tagging.googlecode.com/svn/trunk/ django-tagging

But the hyphen in the application name can cause issues installing
into a DB, so it's really better to do this::

    svn checkout http://django-tagging.googlecode.com/svn/trunk/ tagging

If you've already downloaded, rename the directory before installing.

To install django-tagging, do the following:

    1. Put the ``tagging`` folder somewhere on your Python path.
    2. Put ``'tagging'`` in your ``INSTALLED_APPS`` setting.
    3. Run the command ``manage.py syncdb``.

The ``syncdb`` command creates the necessary database tables and
creates permission objects for all installed apps that need them.

That's it!


Tags
====

Tags are represented by the ``Tag`` model, which lives in the
``tagging.models`` module.

API reference
-------------

Fields
~~~~~~

``Tag`` objects have the following fields:

    * ``name`` -- The name of the tag. This is a unique value
      consisting only of letters, numbers, hypens and underscores.

Manager functions
~~~~~~~~~~~~~~~~~

The ``Tag`` model has a custom manager that has the following helper
functions:

    * ``update_tags(obj, tag_names)`` -- Updates tags associated with
      an object.

      ``tag_names`` is a string containing tag names with which
      ``obj`` should be tagged. Tag names must be valid slugs.
      Multiple tag names may be specified, separated by any number of
      commas and spaces.

      If ``tag_names`` is ``None`` or ``''``, the object's tags will
      be cleared.

    * ``get_for_object(obj)`` -- Returns a ``QuerySet`` containing all
      ``Tag`` objects associated with ``obj``.

    * ``usage_for_model(Model, counts=True)`` -- Returns a
      ``QuerySet`` containing the distinct ``Tag`` objects associated
      with all instances of model ``Model``.

      If ``counts`` is ``True``, a ``count`` attribute will be added
      to each tag, indicating how many times it has been associated
      with all instances of ``Model``.

    * ``cloud_for_model(Model, steps=4)`` -- Returns a list of the
      distinct ``Tag`` objects associated with all instances of
      ``Model``, each having along a ``count`` attribute as above and
      an additional ``font_size`` attribute, for use in creation of a
      tag cloud (a weighted list).

      ``steps`` defines the number of font sizes available -
      ``font_size`` may be an integer between ``1`` and ``steps``,
      inclusive.

      The algorithm used to calculate font sizes is from a blog post
      by Chase Davis, `Log-based tag clouds in Python`_.

      .. _`Log-based tag clouds in Python`: http://www.car-chase.net/2007/jan/16/log-based-tag-clouds-python/

Basic usage
-----------

Tagging objects and retrieving an object's tags
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Objects may be tagged the ``update_tags`` helper function::

    >>> from shop.apps.products.models import Widget
    >>> from tagging.models import Tag
    >>> widget = Widget.objects.get(pk=1)
    >>> Tag.objects.update_tags(widget, 'house thing')

Retrieve tags for an object using the ``get_for_object`` helper
function::

    >>> Tag.objects.get_for_object(widget)
    [<Tag: house>, <Tag: thing>]

Tags are created, associated and unassociated accordingly when you use
``update_tags``::

    >>> Tag.objects.update_tags(widget, 'house monkey')
    >>> Tag.objects.get_for_object(widget)
    [<Tag: house>, <Tag: monkey>]

Clear an object's tags by passing ``None`` to ``update_tags``::

    >>> Tag.objects.update_tags(widget, None)
    >>> Tag.objects.get_for_object(widget)
    []

Retrieving tags used by a particular model
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To retrieve all tags used for a particular model, use the
``get_for_model`` helper function::

    >>> widget1 = Widget.objects.get(pk=1)
    >>> Tag.objects.update_tags(widget1, 'house thing')
    >>> widget2 = Widget.objects.get(pk=2)
    >>> Tag.objects.update_tags(widget2, 'cheese toast house')
    >>> Tag.objects.get_for_model(Widget)
    [<Tag: cheese>, <Tag: house>, <Tag: thing>, <Tag: toast>]

To get a count of how many times each tag was used for a particular
model, pass in ``True`` for the ``counts`` argument::

    >>> tags = Tag.objects.get_for_model(Widget, counts=True)
    >>> [(tag.name, tag.count) for tag in tags]
    [('cheese', 1), ('house', 2), ('thing', 1), ('toast', 1)]


Tagged Items
============

The relationship between a ``Tag`` and an object is represented by
the ``TaggedItem`` model, which lives in the ``tagging.models``
module.

API reference
-------------

Fields
~~~~~~

``TaggedItem`` objects have the following fields:

    * ``tag`` -- The ``Tag`` an object is associated with.
    * ``content_type`` -- The ContentType of the associated object.
    * ``object_id`` -- The id of the associated object.
    * ``object`` -- The associated object.

Manager functions
~~~~~~~~~~~~~~~~~

The ``TaggedItem`` model has a custom manager that has the following
helper functions:

    * ``get_by_model(Model, tag)`` -- If ``tag`` is an instance of a
      ``Tag``, returns a ``QuerySet`` containing all instances of
      ``Model`` which are tagged with ``tag``.

      If ``tag`` is a list of tags, returns a ``QuerySet`` containing
      all instances of ``Model`` which are tagged with every tag in
      the list.

    * ``get_intersection_by_model(Model, tags)`` -- Returns a
      ``QuerySet`` containing all instances of ``Model`` which are
      tagged with every tag in the list.

      ``get_by_model`` will call this function behind the scenes when
      you pass it a list, so it's recommended that you use
      ``get_by_model`` instead of calling this function directly.

Basic usage
-----------

Retrieving tagged objects
~~~~~~~~~~~~~~~~~~~~~~~~~

Objects may be retrieve based on their tags using the ``get_by_model``
helper function::

    >>> from shop.apps.products.models import Widget
    >>> from tagging.models import Tag
    >>> house_tag = Tag.objects.get(name='house')
    >>> TaggedItem.objects.get_by_model(Widget, house_tag)
    [<Widget: pk=1>, <Widget: pk=2>]

Passing a list of tags to ``get_by_model`` returns an intersection of
objects which have those tags, i.e. tag1 AND tag2 ... AND tagN::

    >>> thing_tag = Tag.objects.get(name='thing')
    >>> TaggedItem.objects.get_by_model(Widget, [house_tag, thing_tag])
    [<Widget: pk=1>]

You can also pass a ``QuerySet`` to ``get_by_model``::

    >>> tags = Tag.objects.filter(name__in=['house', 'thing'])
    >>> TaggedItem.objects.get_by_model(Widget, tags)
    [<Widget: pk=1>]


Fields
======

The ``tagging.fields`` module contains fields which make it easy to
integrate tagging into your models and into the
``django.contrib.admin`` application.

Field types
-----------

``TagField``
~~~~~~~~~~~~

A ``CharField`` that actually works as a relationship to tags "under
the hood".

Using this example model::

    class Link(models.Model):
        ...
        tags = TagField()

Setting tags::

    >>> l = Link.objects.get(...)
    >>> l.tags = 'tag1 tag2 tag3'

Getting tags for an instance::

    >>> l.tags
    'tag1 tag2 tag3'

Getting tags for a model - i.e. all tags used by all instances of the
model::

    >>> Link.tags
    'tag1 tag2 tag3 tag4 tag5'

This field will also validate that it has been given a valid list of
tag names, separated by a single comma, a single space or a comma
followed by a space, using the ``isTagList`` validator from
``tagging.validators``.


Simplified tagging and retrieval of tags with properties
========================================================

If you're not using ``TagField``, a useful method for simplifying
tagging and retrieval of tags for your models is to set up a
property::

    from django.db import models
    from tagging.models import Tag

    class MyModel(models.Model):
        name = models.CharField(maxlength=100)
        tag_list = models.CharField(maxlength=255)

        def save(self):
            super(MyModel, self).save()
            self.tags = self.tag_list

        def _get_tags(self):
            return Tag.objects.get_for_object(self)

        def _set_tags(self, tag_list):
            Tag.objects.update_tags(self, tag_list)

        tags = property(_get_tags, _set_tags)

        def __str__(self):
            return self.name

Once you've set this up, you can access and set tags in a fairly
natural way::

    >>> obj = MyModel.objects.get(pk=1)
    >>> obj.tags = 'foo bar'
    >>> obj.tags
    [<Tag: bar>, <Tag: foo>]

Remember that ``obj.tags`` will return a ``QuerySet``, so you can
perform further filtering on it, should you need to.


Template tags
=============

The ``tagging.templatetags.tagging_tags`` module defines a number of
template tags which may be used to work with tags.

Tag reference
-------------

tag_for_object
~~~~~~~~~~~~~~~~

Retrieves a list of tags associated with an object and stores them in
a context variable.

Example usage::

    {% tags_for_object widget as tag_list %}

tagged_objects
~~~~~~~~~~~~~~

Retrieves a list of objects for a given model which are tagged with
a given tag and stores them in a context variable.

The tag must be an instance of a ``Tag``, not the name of a tag.

The model is specified in ``[appname].[modelname]`` format.

Example usage::

    {% tagged_objects house_tag in products.Widget as widgets %}