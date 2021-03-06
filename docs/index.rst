*************************
AstrodbKit2 Documentation
*************************

AstrodbKit2 is an astronomical database handler code built on top of SQLAlchemy.
The goal behind this code is to provide SQLAlchemy's
powerful `Object Relational Mapping (ORM) <https://docs.sqlalchemy.org/en/13/orm/>`_
infrastructure to access astronomical database contents regardless of the underlying architecture.

This was designed to work with the `SIMPLE database <https://github.com/kelle/SIMPLE>`_, though
similar databases will work.
**Astrodbkit2** is inspired from the original **astrodbkit**, which is hardcoded for the SQLite BDNYC database.

Getting Started
===============

To install **Astrodbkit2**, do::

    pip install astrodbkit2

or directly via the Github repo::

    pip install git+https://github.com/dr-rodriguez/AstrodbKit2

Creating a Database
===================

To create a database from scratch users will need a database schema coded with the SQLAlchemy ORM.
An example schema is provided (see schema_example.py),
but users can also refer to the `SIMPLE schema <https://github.com/kelle/SIMPLE/blob/main/simple/schema.py>`_.
With that on hand, users should import their schema and prepare the database::

    from astrodbkit2.astrodb import create_database
    from simple.schema import *

    connection_string = 'sqlite:///SIMPLE.db'  # connection string for a SQLite database named SIMPLE.db
    create_database(connection_string)

Accessing the Database
======================

To start using the database, launch Python, import the module,
then initialize the database with the :py:class:`astrodbkit2.astrodb.Database()` class like so::

    from astrodbkit2.astrodb import Database

    connection_string = 'sqlite:///SIMPLE.db'  # SQLite connection string
    db = Database(connection_string)

The database is now read to be used. If the database is empty, see below how to populate it.

Loading the Database
--------------------

**Astrodbkit2** contains methods to output the full contents of the database as a list of JSON files.
It can likewise read in a directory of these files to populate the database.
This is how SIMPLE is currently version controlled. To load a database of this form, do the following::

    from astrodbkit2.astrodb import Database

    connection_string = 'sqlite:///SIMPLE.db'  # SQLite connection string
    db_dir = 'data'  # directory where JSON files are located
    db = Database(connection_string)
    db.load_database(db_dir)

.. note:: Database contents are cleared when loading from JSON files to ensure that the database only contains
          sources from on-disk files. We describe later how to use the :py:meth:`~astrodbkit2.astrodb.Database.save_db` method
          to produce JSON files from the existing database contents.

Querying the Database
=====================

Upon connecting to a database, **Astrodbkit2** creates methods for each table defined in the schema.
This allows for a more python approach to writing queries. There are also methods to perform specialized queries.

Exploring the Schema
--------------------

The database schema is accessible via the :py:attr:`~astrodbkit2.astrodb.Database.metadata` attribute.

For example, to see the available tables users can do::

    for table in db.metadata.tables:
        print(table)

And users can also examine column information for an existing table::

    for c in db.metadata.tables['Sources'].columns:
        print(c.name, c.type, c.primary_key, c.foreign_keys, c.nullable)

    # Example output
    source VARCHAR(100) True set() False
    ra FLOAT False set() True
    dec FLOAT False set() True
    shortname VARCHAR(30) False set() True
    reference VARCHAR(30) False {ForeignKey('Publications.name')} False
    comments VARCHAR(1000) False set() True

Specialized Searches
--------------------

To search for an object by name, users can use the :py:meth:`~astrodbkit2.astrodb.Database.search_object`
method to do fuzzy searches on the provided name, output results from any table,
and also include alternate Simbad names for their source. Refer to the API documentation for full details.

Search for TWA 27 and return default results in Astropy Table format::

    db.search_object('twa 27', fmt='astropy')

Search for TWA 27 and any of its alternate designations from Simbad and return results from the Names table::

    db.search_object('twa 27', resolve_simbad=True, output_table='Names')

Search for any source with 1357+1428 in its name and return results from the Photometry table in pandas Dataframe format::

    db.search_object('1357+1428', output_table='Photometry', fmt='astropy')

**Astrodbkit2**  also contains an :py:meth:`~astrodbkit2.astrodb.Database.inventory` method to return all data for a source by its name::

    data = db.inventory('2MASS J13571237+1428398')
    print(data)  # output as a dictionary, with individual tables as results

The pretty_print parameter can be passed to print out results to the screen in an easier to read format::

    db.inventory('2MASS J13571237+1428398', pretty_print=True)

    # Partial output:
    {
        "Sources": [
            {
                "source": "2MASS J13571237+1428398",
                "ra": 209.301675,
                "dec": 14.477722,
                "shortname": "1357+1428",
                "reference": "Schm10",
                "comments": null
            }
        ],
        "Names": [
            {
                "other_name": "2MASS J13571237+1428398"
            },
            {
                "other_name": "SDSS J135712.40+142839.8"
            }
        ],
        "Photometry": [
            {
                "band": "WISE_W1",
                "ucd": null,
                "magnitude": 13.348,
                "magnitude_error": 0.025,
                "telescope": "WISE",
                "instrument": null,
                "epoch": null,
                "comments": null,
                "reference": "Cutr12"
            },
            ...
        ]
    }

General Queries
--------------------

Frequently, users may wish to perform specialized queries against the full database.
This can be used with the SQLAlchemy ORM and a convenience method, :py:attr:`~astrodbkit2.astrodb.Database.query`, exists for this.
For more details on how to use SQLAlchemy, refer to `their documentation <https://docs.sqlalchemy.org/en/13/orm/>`_.
Here are a few examples.

Query all columns for the table Sources and output in a variety of formats::

    db.query(db.Sources).all()      # default SQLAlchemy output (list of named tuples)
    db.query(db.Sources).astropy()  # Astropy Table output
    db.query(db.Sources).table()    # equivalent to astropy
    db.query(db.Sources).pandas()   # Pandas DataFrame

Example query for sources with declinations larger than 0::

    db.query(db.Sources).filter(db.Sources.c.dec > 0).table()

Example query returning just a single column (source) and sorting sources by declination::

    db.query(db.Sources.c.source).order_by(db.Sources.c.dec).table()

Example query joining Sources and Publications tables and return just several of the columns::

    db.query(db.Sources.c.source, db.Sources.c.reference, db.Publications.c.name)\
            .join(db.Publications, db.Sources.c.reference == db.Publications.c.name)\
            .table()

Example queries showing how to perform ANDs and ORs::

    # Query with AND
    db.query(db.Sources).filter(and_(db.Sources.c.dec > 0, db.Sources.c.ra > 200)).all()

    # Query with OR
    db.query(db.Sources).filter(or_(db.Sources.c.dec < 0, db.Sources.c.ra > 200)).all()

In addition to using the ORM, it is useful to note that a :py:meth:`~astrodbkit2.astrodb.Database.sql_query` method exists
to pass direct SQL queries to the database for users who may wish to write their own SQL statements::

    results = db.sql_query('select * from sources', fmt='astropy')
    print(results)

Modifying Data
==============

As a wrapper against standard SQLAlchemy calls, data can be added fairly simply.

.. note:: Primary and Foreign keys, if present in the database, are verified when modifying data.
          This can prevent duplicated keys from being created and can propagate deletes or updates as specified
          in the database schema.

Adding Data
-----------

The simplest way to add data to an existing database is to construct a list of dictionaries and insert it to a table::

    sources_data = [{'ra': 209.301675, 'dec': 14.477722,
                     'source': '2MASS J13571237+1428398',
                     'reference': 'Schm10',
                     'shortname': '1357+1428'}]
    db.Sources.insert().execute(sources_data)

Updating Data
-------------

Similarly, rows can be updated with standard SQLAlchemy calls.
This example sets the shortname for a row that matches a Source with source name of 2MASS J13571237+1428398::

    stmt = db.Sources.update()\
             .where(db.Sources.c.source == '2MASS J13571237+1428398')\
             .values(shortname='1357+1428')
    db.engine.execute(stmt)

Deleting Data
-------------

Deleting rows can also be done. Here's an example that delets all photometry with band name of WISE_W1::

    db.Photometry.delete().where(db.Photometry.c.band == 'WISE_W1').execute()

Saving the Database
===================

If users perform changes to a database, they will want to output this to disk to be version controlled.
**Astrodbkit2** provides methods to save an individual source as well as the entire data.
We recommend the later to output the entire contents to disk::

    # Save single object
    db.save_json('2MASS J13571237+1428398', 'data')

    # Save entire database to directory 'data'
    db.save_database('data')

.. note:: To properly capture database deletes, the contents of the specified directory is first cleared before
          creating JSON files representing the current state of the database.

Reference/API
=============

.. toctree::
   :maxdepth: 2

   astrodb.rst
   utils.rst

Indices and tables

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
