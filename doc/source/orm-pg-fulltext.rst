PostgreSQL Full Text Search
===========================

Full Text Searching (or just text search) provides the capability to identify natural-language 
documents that satisfy a query, and optionally to sort them by relevance to the query. The most 
common type of search is to find all documents containing given query terms and return them in 
order of their similarity to the query. Notions of query and similarity are very flexible and 
depend on the specific application. The simplest search considers query as a set of words and 
similarity as the frequency of query words in the document. (`From postgresql documentation.`)


Currently these classes are implemented:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

`django_orm.postgresql.fulltext.fields.VectorField`
    An tsvector index field which stores converted text into special format.

`django_orm.manager.FtsManager`
    Django manager that contains helper methods for search and re/genereate indexes.


How to use it from django?
--------------------------

To use it, you will need to add a new field and modifying one or the other method in the model.

.. code-block:: python
    
    from django_orm.manager import FtsManager as SearchManager
    from django_orm.postgresql.fulltext.fields import VectorField
    from django.db import models

    class Page(models.Model):
        name = models.CharField(max_length=200)
        description = models.TextField()

        search_index = VectorField()

        objects = SearchManager(
            fields = ('name', 'description'),
            config = 'pg_catalog.english', # this is default
            search_field = 'search_index'  # this is default
        )

        def save(self):
            super(Page, self).save()
            if hasattr(self, '_orm_manager'):
                self._orm_manager.update_index(pk=self.pk)


The manager automatically injected, the method ``update_index`` to the model instance. 
Also, not to override the save method, you can pass the parameter ``auto_update_index = True``, so 
the index is updated automatically by calling the ``save`` method.


Usage examples:
^^^^^^^^^^^^^^^

- The config parameter is optional and defaults to 'pg_catalog.english'.
- The fields parameter is optional. If a list of tuples, you can specify the ranking of each field, if it is None, it gets 'A' as the default.
- It can also be a simple list of fields, and the ranking will be selected by default. If the field is empty, the index was applied to all fields ``CharField`` and ``TextField``.

To search, use the ``search`` method of the manager. The current version, the method used by default unaccented, so ignore the accents and searches are case insencitive.

.. code-block:: python

    >>> Page.objects.search("documentation & about")
    [<Page: Page: Home page>]
    >>> Page.objects.search("about | documentation | django | home", raw=True)
    [<Page: Page: Home page>, <Page: Page: About>, <Page: Page: Navigation>]

FTS extension by default uses plainto_tsquery instead of to_tosquery, for this reason the use of raw parameter. 


General notes:
^^^^^^^^^^^^^^

You must ensure you have installed the extension `unaccent`:

.. code-block:: sql

    CREATE EXTENSION unaccent;
    ALTER FUNCTION unaccent(text) IMMUTABLE;


In postgresql90 sometimes does not work as we wish, we can use one like this:

.. code-block:: sql

    CREATE OR REPLACE FUNCTION unaccent(text) RETURNS text AS $$ 
    DECLARE input_string text := $1; 
    BEGIN 
        input_string := translate(input_string, 'àáâäãåāăąÀÁÂÄÃÅĀĂĄ', 'aaaaaaaaaaaaaaaaaa'); 
        input_string := translate(input_string, 'èéêëēĕėęěÈÉÊËÊĒĔĖĘĚ', 'eeeeeeeeeeeeeeeeeee'); 
        input_string := translate(input_string, 'ìíîïĩīĭÌÍÎÏĨĪĬ', 'iiiiiiiiiiiiii'); 
        input_string := translate(input_string, 'òóôöõōŏőÒÓÔÖÕŌŎŐ', 'oooooooooooooooo'); 
        input_string := translate(input_string, 'ùúûüũūŭůÙÚÛÜŨŪŬŮ', 'uuuuuuuuuuuuuuuu'); 
        input_string := translate(input_string, 'ñÑçÇ', 'nncc'); 
        return input_string; 
    END; $$ LANGUAGE plpgsql IMMUTABLE;
