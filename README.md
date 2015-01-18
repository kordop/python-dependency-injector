Objects
=======

Python catalogs of objects providers.


Example of objects catalog definition and usage:

```python
"""
Concept example of objects catalogs.
"""

from objects import AbstractCatalog
from objects.providers import Singleton, NewInstance
from objects.injections import InitArg, Attribute
import sqlite3


# Some example classes.
class ObjectA(object):
    def __init__(self, db):
        self.db = db


class ObjectB(object):
    def __init__(self, a, db):
        self.a = a
        self.db = db


# Catalog of objects providers.
class Catalog(AbstractCatalog):
    """
    Objects catalog.
    """

    database = Singleton(sqlite3.Connection,
                         InitArg('database', ':memory:'),
                         Attribute('row_factory', sqlite3.Row))
    """ :type: (objects.Provider) -> sqlite3.Connection """

    object_a = NewInstance(ObjectA,
                           InitArg('db', database))
    """ :type: (objects.Provider) -> ObjectA """

    object_b = NewInstance(ObjectB,
                           InitArg('a', object_a),
                           InitArg('db', database))
    """ :type: (objects.Provider) -> ObjectB """


# Catalog static provides.
a1, a2 = Catalog.object_a(), Catalog.object_a()
b1, b2 = Catalog.object_b(), Catalog.object_b()

# Some asserts.
assert a1 is not a2
assert b1 is not b2
assert a1.db is a2.db is b1.db is b2.db is Catalog.database()


# Dependencies injection (The Python Way) into class.
class Consumer(object):

    dependencies = Catalog(Catalog.object_a,
                           Catalog.object_b)

    def test(self):
        a1 = self.dependencies.object_a()
        a2 = self.dependencies.object_a()

        b1 = self.dependencies.object_b()
        b2 = self.dependencies.object_b()

        # Some asserts.
        assert a1 is not a2
        assert b1 is not b2
        assert a1.db is a2.db is b1.db is b2.db

        try:
            self.dependencies.database()
        except AttributeError:
            pass
        else:
            raise Exception('Database is not listed as a dependency')

Consumer().test()


# Dependencies injection (The Python Way) into a callback.
def consumer_callback(dependencies=Catalog(Catalog.object_a,
                                           Catalog.object_b)):
    a1 = dependencies.object_a()
    a2 = dependencies.object_a()

    b1 = dependencies.object_b()
    b2 = dependencies.object_b()

    # Some asserts.
    assert a1 is not a2
    assert b1 is not b2
    assert a1.db is a2.db is b1.db is b2.db

    try:
        dependencies.database()
    except AttributeError:
        pass
    else:
        raise Exception('Database is not listed as a dependency')
```

Example of overriding object providers:

```python
"""
Concept example of objects overrides.
"""


from objects import AbstractCatalog, overrides
from objects.providers import Singleton, NewInstance
from objects.injections import InitArg, Attribute
import sqlite3


# Some example class.
class ObjectA(object):
    def __init__(self, db):
        self.db = db


class ObjectAMock(ObjectA):
    pass


# Catalog of objects providers.
class Catalog(AbstractCatalog):
    """
    Objects catalog.
    """

    database = Singleton(sqlite3.Connection,
                         InitArg('database', ':memory:'),
                         Attribute('row_factory', sqlite3.Row))
    """ :type: (objects.Provider) -> sqlite3.Connection """

    object_a = NewInstance(ObjectA,
                           InitArg('db', database))
    """ :type: (objects.Provider) -> ObjectA """


# Overriding Catalog by SandboxCatalog with some mocks.
@overrides(Catalog)
class SandboxCatalog(Catalog):
    """
    Sandbox objects catalog with some mocks.
    """

    object_a = NewInstance(ObjectAMock,
                           InitArg('db', Catalog.database))
    """ :type: (objects.Provider) -> ObjectA """


# Catalog static provides.
a1 = Catalog.object_a()
a2 = Catalog.object_a()

# Some asserts.
assert isinstance(a1, ObjectAMock)
assert isinstance(a2, ObjectAMock)
assert a1 is not a2
assert a1.db is a2.db is Catalog.database()
```
