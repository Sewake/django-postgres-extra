.. include:: ./snippets/postgres_doc_links.rst

.. warning::

   Table partitioning is a relatively new and advanded PostgreSQL feature. It has plenty of ways to shoot yourself in the foot with.

   We HIGHLY RECOMMEND you only use this feature if you're already deeply familiar with table partitioning and aware of its advantages and disadvantages.

   Do study the PostgreSQL documentation carefuly.

.. _table_partitioning_page:


Table partitioning
==================

:class:`~psqlextra.models.PostgresPartitionedModel` adds support for `PostgreSQL Declarative Table Partitioning`_.

The following partitioning method are available:

* ``PARTITION BY RANGE``
* ``PARTITION BY LIST``

.. note::

   Although table partitioning is available in PostgreSQL 10.x, it is highly recommended you use PostgresSQL 11.x. Table partitioning got a major upgrade in PostgreSQL 11.x.

   PostgreSQL 10.x does not support creating foreign keys to/from partitioned tables and does not automatically create an index across all partitions.


Creating partitioned tables
---------------------------

Partitioned tables are declared like regular Django models with a special base class and two extra options to set the partitioning method and key. Once declared, they behave like regular Django models.


Declaring the model
*******************

Inherit your model from :class:`psqlextra.models.PostgresPartitionedModel` and declare a child class named ``PartitioningMeta``. On the meta class, specify the partitioning method and key.

* Use :attr:`psqlextra.types.PostgresPartitioningMethod.RANGE` to ``PARTITION BY RANGE``
* Use :attr:`psqlextra.types.PostgresPartitioningMethod.LIST` to ``PARTITION BY LIST``

.. code-block:: python

   from django.db import models

   from psqlextra.types import PostgresPartitioningMethod
   from psqlextra.models import PostgresPartitionedModel

   class MyModel(PostgresPartitionedModel):
       class PartitioningMeta:
           method = PostgresPartitioningMethod.RANGE
           key = ["timestamp"]

       name = models.TextField()
       timestamp = models.DateTimeField()


Generating a migration
**********************
Run the following command to automatically generate a migration:

.. code-block:: bash

   python manage.py pgmakemigrations

This will generate migrationt that creates the partitioned table with a default partition.


Adding/removing partitions manually
-----------------------------------

Postgres does not have support for automatically creating new partitions as needed. Therefor, one must manually add new partitions. Depending on the partitioning method you have chosen, the partition has to be created differently.

Partitions are tables. Each partition must be given a unique name. :class:`~psqlextra.models.PostgresPartitionedModel` does not require you to create a model for each partition because you are not supposed to query partitions directly.


Using migrations
****************

Migrations for the creation and deletion of partitioned models can be handled automatically using the special `pgmakemigrations` command:

.. code-block:: bash

   python manage.py pgmakemigrations


Adding a range partition
~~~~~~~~~~~~~~~~~~~~~~~~

Use the :class:`~psqlextra.migrations.operations.PostgresAddRangePartition` operation to add a new range partition. Only use this operation when your partitioned model uses the :attr:`psqlextra.types.PostgresPartitioningMethod.RANGE`.

.. code-block:: python

   from django.db import migrations, models

   from psqlextra.migrations.operations import PostgresAddRangePartition

   class Migration(migrations.Migration):
       operations = [
           PostgresAddRangePartition(
              model_name="mypartitionedmodel",
              name="pt1",
              from_values="2019-01-01",
              to_values="2019-02-01",
           ),
       ]


Adding a list partition
~~~~~~~~~~~~~~~~~~~~~~~

Use the :class:`~psqlextra.migrations.operations.PostgresAddListPartition` operation to add a new list partition. Only use this operation when your partitioned model uses the :attr:`psqlextra.types.PostgresPartitioningMethod.LIST`.

.. code-block:: python

   from django.db import migrations, models

   from psqlextra.migrations.operations import PostgresAddRangePartition

   class Migration(migrations.Migration):
       operations = [
           PostgresAddListPartition(
              model_name="mypartitionedmodel",
              name="pt1",
              values=["car", "boat"],
           ),
       ]


Adding a default partition
~~~~~~~~~~~~~~~~~~~~~~~~~~

Use the :class:`~psqlextra.migrations.operations.PostgresAddDefaultPartition` operation to add a new default partition. A default partition is the partition where records get saved that couldn't fit in any other partition.

Note that you can only have one default partition per partitioned table/model.

.. code-block:: python

   from django.db import migrations, models

   from psqlextra.migrations.operations import PostgresAddDefaultPartition

   class Migration(migrations.Migration):
       operations = [
           PostgresAddDefaultPartition(
              model_name="mypartitionedmodel",
              name="default",
           ),
       ]


Deleting a default partition
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use the :class:`~psqlextra.migrations.operations.PostgresDeleteDefaultPartition` operation to delete an existing default partition.

.. code-block:: python

   from django.db import migrations, models

   from psqlextra.migrations.operations import PostgresDeleteDefaultPartition

   class Migration(migrations.Migration):
       operations = [
           PostgresDeleteDefaultPartition(
              model_name="mypartitionedmodel",
              name="pt1",
           ),
       ]


Deleting a range partition
~~~~~~~~~~~~~~~~~~~~~~~~~~

Use the :class:`psqlextra.migrations.operations.PostgresDeleteRangePartition` operation to delete an existing range partition.

.. code-block:: python

   from django.db import migrations, models

   from psqlextra.migrations.operations import PostgresDeleteRangePartition

   class Migration(migrations.Migration):
       operations = [
           PostgresDeleteRangePartition(
              model_name="mypartitionedmodel",
              name="pt1",
           ),
       ]


Deleting a list partition
~~~~~~~~~~~~~~~~~~~~~~~~~

Use the :class:`~psqlextra.migrations.operations.PostgresDeleteListPartition` operation to delete an existing list partition.

.. code-block:: python

   from django.db import migrations, models

   from psqlextra.migrations.operations import PostgresDeleteListPartition

   class Migration(migrations.Migration):
       operations = [
           PostgresDeleteListPartition(
              model_name="mypartitionedmodel",
              name="pt1",
           ),
       ]


Using the schema editor
***********************

Use the :class:`psqlextra.backend.PostgresSchemaEditor` to manage partitions directly in a more imperative fashion. The schema editor is used by the migration operations described above.


Adding a range partition
~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   from django.db import connection

   connection.schema_editor.add_range_partition(
       model=MyPartitionedModel,
       name="pt1",
       from_values="2019-01-01",
       to_values="2019-02-01",
   )


Adding a list partition
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   from django.db import connection

   connection.schema_editor.add_list_partition(
       model=MyPartitionedModel,
       name="pt1",
       values=["car", "boat"],
   )


Adding a default partition
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   from django.db import connection

   connection.schema_editor.add_default_partition(
       model=MyPartitionedModel,
       name="default",
   )


Deleting a partition
~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   from django.db import connection

   connection.schema_editor.delete_partition(
       model=MyPartitionedModel,
       name="default",
   )


Adding/removing partitions automatically
----------------------------------------

:class:`psqlextra.partitioning.PostgresPartitioningManager` an experimental helper class that can be called periodically to automatically create new partitions if you're using range partitioning.

.. note::

   There is currently no scheduler or command to automatically create new partitions. You'll have to run this function in your own cron jobs.

The auto partitioner supports automatically creating yearly, monthly, weekly or daily partitions. Use the ``count`` parameter to configure how many partitions it should create ahead.


Partitioning strategies
***********************


Monthly partitioning
~~~~~~~~~~~~~~~~~~~~

Partitions will be named ``[table_name]_[3-letter month name]``.

.. code-block:: python

   from psqlextra.partitioning import (
      PostgresPartitioningManager,
      partition_by_current_time,
   )

   # 3 partitions ahead, each partition is one month
   manager = PostgresPartitioningManager(
      model=MyPartitionedModel,
      count=3,
      months=1,
   )
   manager.apply()


Weekly partitioning
~~~~~~~~~~~~~~~~~~~

Partitions will be named ``[table_name]_week_[week number]``.

.. code-block:: python

   from psqlextra.partitioning import (
      PostgresPartitioningManager,
      partition_by_current_time,
   )

   # 4 partitions ahead, each partition is 1 week
   manager = PostgresPartitioningManager(
      model=MyPartitionedModel,
      count=4,
      weeks=1,
   )
   manager.apply()


   # 6 partitions ahead, each partition is 2 weeks
   manager = PostgresPartitioningManager(
      model=MyPartitionedModel,
      count=6,
      weeks=2,
   )
   manager.apply()


Switching partitioning strategies
*********************************

When switching partitioning strategies, you might encounter the problem that partitions for part of a particular range already exist. In order to combat this, you can specify the ``start_from`` parameter to not create partitions for a date/time earlier than specified.
