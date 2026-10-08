Batch Processing
================

This chapter shows you how to accomplish bulk inserts, updates and
deletes with Doctrine in an efficient way. The main problem with
bulk operations is usually not to run out of memory and this is
especially what the strategies presented here provide help with.

.. warning::

    An ORM tool is not primarily well-suited for mass
    inserts, updates or deletions. Every RDBMS has its own, most
    effective way of dealing with such operations and if the options
    outlined below are not sufficient for your purposes we recommend
    you use the tools for your particular RDBMS for these bulk
    operations.


.. note::

    Having an SQL logger enabled when processing batches can have a
    serious impact on performance and resource usage.
    To avoid that, you should use a PSR logger implementation that can be
    disabled at runtime.
    For example, with Monolog, you can use ``Logger::pushHandler()``
    to push a ``NullHandler`` to the logger instance, and then pop it
    when you need to enable logging again.

    With DBAL 2, you can disable the SQL logger like below:

.. code-block:: php

    <?php
    $em->getConnection()->getConfiguration()->setSQLLogger(null);

Bulk Inserts
------------

Bulk inserts in Doctrine are best performed in batches, taking
advantage of the transactional write-behind behavior of an
``EntityManager``. The following code shows an example for
inserting 10000 objects with a batch size of 20. You may need to
experiment with the batch size to find the size that works best for
you. Larger batch sizes mean more prepared statement reuse
internally but also mean more work during ``flush``.

.. code-block:: php

    <?php
    $batchSize = 20;
    for ($i = 1; $i <= 10000; ++$i) {
        $user = new CmsUser;
        $user->setStatus('user');
        $user->setUsername('user' . $i);
        $user->setName('Mr.Smith-' . $i);
        $em->persist($user);
        if (($i % $batchSize) === 0) {
            $em->flush();
            $em->clear(); // Detaches all objects from Doctrine!
        }
    }
    $em->flush(); // Persist objects that did not make up an entire batch
    $em->clear();

Bulk Updates
------------

There are 2 possibilities for bulk updates with Doctrine.

DQL UPDATE
~~~~~~~~~~

The by far most efficient way for bulk updates is to use a DQL
UPDATE query. Example:

.. code-block:: php

    <?php
    $q = $em->createQuery('update MyProject\Model\Manager m set m.salary = m.salary * 0.9');
    $numUpdated = $q->execute();

Iterating results
~~~~~~~~~~~~~~~~~

An alternative solution for bulk updates is to use the
``Query#toIterable()`` facility to iterate over the query results step
by step instead of hydrating all entities at once.
The following example shows how to do this, combining the iteration
with the batching strategy that was already used for bulk inserts:

.. code-block:: php

    <?php
    $batchSize = 20;
    $i = 0;
    $q = $em->createQuery('select u from MyProject\Model\User u');
    foreach ($q->toIterable() as $user) {
        $user->increaseCredit();
        $user->calculateNewBonuses();
        ++$i;
        if (($i % $batchSize) === 0) {
            $em->flush(); // Executes all updates.
            $em->clear(); // Detaches all objects from Doctrine!
        }
    }
    $em->flush();

.. note::

    Iterating results is not possible with queries that
    fetch-join a collection-valued association. The nature of such SQL
    result sets is not suitable for incremental hydration.

.. note::

    ``toIterable()`` limits how many entities are hydrated at once, but does
    not disable result buffering in the database driver. With their default
    settings, ``pdo_mysql`` and ``pdo_pgsql`` buffer the result set. The libpq
    buffer used by ``pdo_pgsql`` is not included in ``memory_get_usage()`` or
    constrained by PHP's ``memory_limit``. Large results can therefore exhaust
    memory even when the UnitOfWork is cleared regularly.

Processing bounded batches
~~~~~~~~~~~~~~~~~~~~~~~~~~

To bound both driver buffering and the number of managed entities, retrieve
one limited batch at a time. For example, for an entity with a single integer
primary key that is not changed during processing:

.. code-block:: php

    <?php
    $batchSize = 1000;
    $lastId = null;
    do {
        $qb = $em->createQueryBuilder()
            ->select('u')
            ->from(MyProject\Model\User::class, 'u')
            ->orderBy('u.id', 'ASC')
            ->setMaxResults($batchSize);
        if ($lastId !== null) {
            $qb->where('u.id > :lastId')->setParameter('lastId', $lastId);
        }
        $users = $qb->getQuery()->getResult();
        foreach ($users as $user) {
            $user->increaseCredit();
            $user->calculateNewBonuses();
            $lastId = $user->getId();
        }
        $em->flush();
        $em->clear(); // Detaches all objects from Doctrine!
    } while (count($users) === $batchSize);

The first batch has no lower bound, so zero and negative identifiers are
included. The ordering key must be unique and immutable. Concurrent inserts,
deletes or changes to filtering columns mean that this traversal does not
represent a fixed snapshot. Choose transaction isolation and selection criteria
according to the application's consistency requirements. Queries that fetch-join
collections need a different pagination strategy; see :doc:`../tutorials/pagination`.

Unbuffered results
~~~~~~~~~~~~~~~~~~

For a single pass over a result, driver-specific streaming can avoid the full
result buffer. For ``pdo_mysql``, set ``PDO::MYSQL_ATTR_USE_BUFFERED_QUERY`` to
``false`` (``Pdo\Mysql::ATTR_USE_BUFFERED_QUERY`` on PHP 8.4 and later). For
``pdo_pgsql``, lazy fetching is available with ``PDO::ATTR_PREFETCH`` set to
``0`` on PHP 8.5 and later.

Consume the active result before executing another query on the same connection,
including queries triggered by ``flush()`` or loading an association. Depending
on the driver, another query may fail or silently discard unread rows from the
first result, ending the iteration early. Use bounded batches or a separate
connection if additional queries are necessary. Closing a cursor also discards
its unread rows; it does not preserve them for later processing.


Bulk Deletes
------------

There are two possibilities for bulk deletes with Doctrine. You can
either issue a single DQL DELETE query or you can iterate over
results removing them one at a time.

DQL DELETE
~~~~~~~~~~

The by far most efficient way for bulk deletes is to use a DQL
DELETE query.

Example:

.. code-block:: php

    <?php
    $q = $em->createQuery('delete from MyProject\Model\Manager m where m.salary > 100000');
    $numDeleted = $q->execute();

Iterating results
~~~~~~~~~~~~~~~~~

An alternative solution for bulk deletes is to use the
``Query#toIterable()`` facility to iterate over the query results step
by step instead of hydrating all entities at once.
The following example shows how to do this:

.. code-block:: php

    <?php
    $batchSize = 20;
    $i = 0;
    $q = $em->createQuery('select u from MyProject\Model\User u');
    foreach($q->toIterable() as $row) {
        $em->remove($row);
        ++$i;
        if (($i % $batchSize) === 0) {
            $em->flush(); // Executes all deletions.
            $em->clear(); // Detaches all objects from Doctrine!
        }
    }
    $em->flush();

.. note::

    Iterating results is not possible with queries that
    fetch-join a collection-valued association. The nature of such SQL
    result sets is not suitable for incremental hydration.


Iterating Large Results for Data-Processing
-------------------------------------------

You can use the ``toIterable()`` method just to iterate over a large
result and no UPDATE or DELETE intention. ``$query->toIterable()`` returns ``iterable``
so you can detach each entity after processing it. Driver buffering still
applies; use the bounded batches or streaming options described above when
necessary:

.. code-block:: php

    <?php
    $q = $this->_em->createQuery('select u from MyProject\Model\User u');
    foreach ($q->toIterable() as $user) {
        // do stuff with the user

        // detach from Doctrine, so that it can be Garbage-Collected immediately
        $this->_em->detach($user);
    }

.. note::

    Iterating results is not possible with queries that
    fetch-join a collection-valued association. The nature of such SQL
    result sets is not suitable for incremental hydration.
