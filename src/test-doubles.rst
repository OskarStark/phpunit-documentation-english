

.. _test-doubles:

************
Test Doubles
************

Gerard Meszaros introduces the concept of Test Doubles in
his "xUnit Test Patterns" book like so:

    Sometimes it is just plain hard to test the system under test (SUT)
    because it depends on other components that cannot be used in the test
    environment. This could be because they aren't available, they will not
    return the results needed for the test or because executing them would
    have undesirable side effects. In other cases, our test strategy requires
    us to have more control or visibility of the internal behavior of the SUT.

    When we are writing a test in which we cannot (or chose not to) use a real
    depended-on component (DOC), we can replace it with a Test Double. The
    Test Double doesn't have to behave exactly like the real DOC; it merely
    has to provide the same API as the real one so that the SUT thinks it is
    the real one!

The ``createStub(string $type)`` and ``createMock(string $type)`` methods can be used
in a test to automatically generate an object that can act as a test double for the
specified original type (interface or class name). This test double object can be used
in every context where an object of the original type is expected or required.

.. admonition:: Limitation: final, private, and static methods

   Please note that ``final``, ``private``, and ``static`` methods cannot
   be doubled. They are ignored by PHPUnit's test double functionality and
   retain their original behavior except for ``static`` methods which will
   be replaced by a method throwing an exception.

.. admonition:: Limitation: Enumerations and readonly classes

   Enumerations (``enum``) are ``final`` classes and therefore cannot be
   doubled. ``readonly`` classes cannot be extended by classes that are
   not ``readonly`` and therefore cannot be doubled.

.. admonition:: Favour doubling interfaces over doubling classes

   Not only because of the limitations mentioned above, but also to improve
   your software design, favour the doubling of interfaces over the doubling
   of classes.

.. _test-doubles.test-stubs:

Test Stubs
==========

The practice of replacing an object with a test double that (optionally) returns
configured return values is referred to as *stubbing*. You can use a *test stub* to
"replace a real component on which the SUT depends so that the test has a control
point for the indirect inputs of the SUT. This allows the test to force the SUT
down paths it might not otherwise execute" (Gerard Meszaros).

createStub()
------------

``createStub(string $type)`` returns a test stub for the specified type. The creation of this
test stub is performed using best practice defaults: the ``__construct()`` and
``__clone()`` methods of the original class are not executed and the arguments passed
to a method of the test double will not be cloned.

If these defaults are not what you need then you can use the ``getMockBuilder(string $type)``
method to customize the test double generation using a fluent interface.

By default, all methods of the original class are replaced with an implementation that
returns an automatically generated value that satisfies the method's return type
declaration (without calling the original method).

willReturn()
------------

Using the ``willReturn()`` method, for instance, you can configure these implementations
to return a specified value when called. This configured value must be compatible with
the method's return type declaration.

Consider that we have a class that we want to test, ``SomeClass``, which depends
on ``Dependency``:

.. literalinclude:: examples/test-doubles/src/SomeClass.php
   :caption: The class we want to test
   :language: php

.. literalinclude:: examples/test-doubles/src/Dependency.php
   :caption: The dependency we want to stub
   :language: php

Here is a first example of how to use the ``createStub(string $type)`` method to
create a test stub for ``Dependency`` so that we can test ``SomeClass`` without
using a real implementation of ``Dependency``:

.. literalinclude:: examples/test-doubles/SomeClassTest.php
   :caption: Stubbing a method call to return a fixed value
   :name: test-doubles.test-stubs.examples.SomeClassTest.php
   :language: php

.. admonition:: Limitation: Methods named "method"

   The example shown above only works when the original interface or class does not
   declare a method named "method".

   If the original interface or class does declare a method named "method" then
   ``$stub->expects($this->any())->method('doSomething')->willReturn('foo');``
   has to be used.

In the example shown above, we first use the ``createStub()`` method to create a test stub,
an object that looks like an instance of ``Dependency``.

We then use the `Fluent Interface <http://martinfowler.com/bliki/FluentInterface.html>`_
that PHPUnit provides to specify the behavior for the test stub.

"Behind the scenes", PHPUnit automatically generates a new PHP class that implements
the desired behavior when the ``createStub()`` method is used.

Please note that ``createStub()`` will automatically and recursively stub return values
based on a method's return type. Consider the example shown below:

.. literalinclude:: examples/test-doubles/src/C.php
   :caption: A method with a return type declaration
   :language: php

In the example shown above, the ``C::m()`` method has a return type declaration
indicating that this method returns an object of type ``D``. When a test double
for ``C`` is created and no return value is configured for ``m()`` using
``willReturn()`` (see above), for instance, then when ``m()`` is invoked PHPUnit
will automatically create a test double for ``D`` to be returned.

Similarly, if ``m`` had a return type declaration for a scalar type then a return
value such as ``0`` (for ``int``), ``0.0`` (for ``float``), or ``[]`` (for ``array``)
would be generated.

So far, we have configured simple return values using ``willReturn($value)``.
This is a shorthand syntax provided for convenience. :numref:`test-doubles.test-stubs.shorthands`
shows the available stubbing shorthands alongside their longer counterparts.

.. rst-class:: table
.. list-table:: Stubbing shorthands
    :name: test-doubles.test-stubs.shorthands
    :header-rows: 1

    * - short hand
      - longer syntax
    * - ``willReturn($value)``
      - ``will($this->returnValue($value))``
    * - ``willReturnArgument($argumentIndex)``
      - ``will($this->returnArgument($argumentIndex))``
    * - ``willReturnCallback($callback)``
      - ``will($this->returnCallback($callback))``
    * - ``willReturnMap($valueMap)``
      - ``will($this->returnValueMap($valueMap))``
    * - ``willReturnOnConsecutiveCalls($value1, $value2)``
      - ``will($this->onConsecutiveCalls($value1, $value2))``
    * - ``willReturnSelf()``
      - ``will($this->returnSelf())``
    * - ``willThrowException($exception)``
      - ``will($this->throwException($exception))``

We can use variations on this longer syntax to achieve more complex stubbing behaviour.

createConfiguredStub()
----------------------

The ``createConfiguredStub()`` method is a convenience wrapper around ``createStub()`` that allows configuring
return values using an associative array (``['methodName' => <return value>]``):

.. literalinclude:: examples/test-doubles/CreateConfiguredStubExampleTest.php
   :caption: Using createConfiguredStub() to create a test stub and configure return values
   :language: php

createStubForIntersectionOfInterfaces()
---------------------------------------

The ``createStubForIntersectionOfInterfaces(array $interface)`` method can be used to
create a test stub for an intersection of interfaces based on a list of interface names.

Consider you have the following interfaces ``X`` and ``Y``:

.. literalinclude:: examples/test-doubles/src/X.php
   :caption: An interface named X
   :language: php

.. literalinclude:: examples/test-doubles/src/Y.php
   :caption: An interface named Y
   :language: php

And you have a class that you want to test named ``Z``:

.. literalinclude:: examples/test-doubles/src/Z.php
   :caption: A class named Z
   :language: php

To test ``Z``, we need an object that satisfies the intersection type ``X&Y``. We can
use the ``createStubForIntersectionOfInterfaces(array $interface)`` method to create
a test stub that satisfies ``X&Y`` like so:

.. literalinclude:: examples/test-doubles/StubForIntersectionExampleTest.php
   :caption: Using createStubForIntersectionOfInterfaces() to create a test stub for an intersection type
   :language: php

returnArgument()
----------------

Sometimes you want to return one of the arguments of a method call (unchanged) as the
result of a stubbed method call. Here is an example that shows how you can achieve this
using ``returnArgument()`` instead of ``returnValue()``:

.. literalinclude:: examples/test-doubles/ReturnArgumentExampleTest.php
   :caption: Using returnArgument() to stub a method call to return one of the arguments
   :language: php

returnSelf()
------------

When testing a fluent interface, it is sometimes useful to have a stubbed method return
a reference to the stubbed object. Here is an example that shows how you can use
``returnSelf()`` to achieve this:

.. literalinclude:: examples/test-doubles/ReturnSelfExampleTest.php
   :caption: Using returnSelf() to stub a method call to return a reference to the stub object
   :language: php

returnValueMap()
----------------

Sometimes a stubbed method should return different values depending on a predefined list
of arguments. Here is an example that shows how to use ``returnValueMap()`` to create a map
that associates arguments with corresponding return values:

.. literalinclude:: examples/test-doubles/ReturnValueMapExampleTest.php
   :caption: Using returnValueMap() to stub a method call to return the value from a map
   :language: php

returnCallback()
----------------

When the stubbed method call should return a calculated value instead of a fixed one
(see ``returnValue()``) or an (unchanged) argument (see ``returnArgument()``), you
can use ``returnCallback()`` to have the stubbed method return the result of a callback
function or method. Here is an example:

.. literalinclude:: examples/test-doubles/ReturnCallbackExampleTest.php
   :caption: Using returnCallback() to stub a method call to return a value from a callback
   :language: php

onConsecutiveCalls()
--------------------

A simpler alternative to setting up a callback method may be to specify a list of desired
return values. You can do this with the ``onConsecutiveCalls()`` method. Here is an example:

.. literalinclude:: examples/test-doubles/OnConsecutiveCallsExampleTest.php
   :caption: Using onConsecutiveCalls() to stub a method call to return a list of values in the specified order
   :language: php

throwException()
----------------

Instead of returning a value, a stubbed method can also raise an exception.
Here is an example that shows how to use ``throwException()`` to do this:

.. literalinclude:: examples/test-doubles/ThrowExceptionExampleTest.php
   :caption: Using throwException() to stub a method call to throw an exception
   :language: php

.. _test-doubles.mock-objects:

Mock Objects
============

The practice of replacing an object with a test double that verifies
expectations, for instance asserting that a method has been called, is
referred to as *mocking*.

You can use a *mock object* "as an observation point that is used to verify
the indirect outputs of the SUT as it is exercised. Typically, the mock object
also includes the functionality of a test stub in that it must return values to
the SUT if it hasn't already failed the tests but the emphasis is on the
verification of the indirect outputs. Therefore, a mock object is a lot more than
just a test stub plus assertions; it is used in a fundamentally different way"
(Gerard Meszaros).

createMock()
------------

``createMock(string $type)`` returns a mock object for the specified type. The creation of this
mock object is performed using best practice defaults: the ``__construct()`` and
``__clone()`` methods of the original class are not executed and the arguments passed
to a method of the test double will not be cloned.

If these defaults are not what you need then you can use the ``getMockBuilder(string $type)``
method to customize the test double generation using a fluent interface.

By default, all methods of the original class are replaced with an implementation that
returns an automatically generated value that satisfies the method's return type
declaration (without calling the original method). Furthermore, expectations for invocations
of these methods ("method must be called with specified arguments", "method must not be called", ...)
can be configured.

Here is an example: suppose we want to test that the correct method, ``update()``
in our example, is called on an object that observes another object.

Here is the code for the ``Subject`` class and the ``Observer`` interface that are part
of the System under Test (SUT):

.. literalinclude:: examples/test-doubles/src/Subject.php
   :caption: Subject class that is part of the System under Test (SUT)
   :language: php

.. literalinclude:: examples/test-doubles/src/Observer.php
   :caption: Observer interface that is part of the System under Test (SUT)
   :language: php

Here is an example that shows how to use a mock object to test the interaction between
``Subject`` and ``Observer`` objects:

.. literalinclude:: examples/test-doubles/SubjectTest.php
   :caption: Testing that a method gets called once and with a specified argument
   :language: php

We first use the ``createMock()`` method to create a mock object for the ``Observer``.

Because we are interested in verifying the communication between two objects (that a method is called
and which arguments it is called with), we use the ``expects()`` and ``with()`` methods to specify
what this communication should look like.

The ``with()`` method can take any number of arguments, corresponding to the number of arguments
to the method being mocked. You can specify more advanced constraints on the method's arguments
than a simple match.

:ref:`appendixes.assertions.assertThat.tables.constraints` shows the constraints that can be
applied to method arguments and here is a list of the matchers that are available to specify
the number of invocations:

-

  ``any()`` returns a matcher that matches when the method it is evaluated for is executed zero or more times

-

  ``never()`` returns a matcher that matches when the method it is evaluated for is never executed

-

  ``atLeastOnce()`` returns a matcher that matches when the method it is evaluated for is executed at least once

-

  ``once()`` returns a matcher that matches when the method it is evaluated for is executed exactly once

-

  ``exactly(int $count)`` returns a matcher that matches when the method it is evaluated for is executed exactly ``$count`` times

createMockForIntersectionOfInterfaces()
---------------------------------------

The ``createMockForIntersectionOfInterfaces(array $interface)`` method can be used to
create a mock object for an intersection of interfaces based on a list of interface names.

Consider you have the following interfaces ``X`` and ``Y``:

.. literalinclude:: examples/test-doubles/src/X.php
   :caption: An interface named X
   :language: php

.. literalinclude:: examples/test-doubles/src/Y.php
   :caption: An interface named Y
   :language: php

And you have a class that you want to test named ``Z``:

.. literalinclude:: examples/test-doubles/src/Z.php
   :caption: A class named Z
   :language: php

To test ``Z``, we need an object that satisfies the intersection type ``X&Y``. We can
use the ``createMockForIntersectionOfInterfaces(array $interface)`` method to create
a test stub that satisfies ``X&Y`` like so:

.. literalinclude:: examples/test-doubles/MockForIntersectionExampleTest.php
   :caption: Using createMockForIntersectionOfInterfaces() to create a mock object for an intersection type
   :language: php

createConfiguredMock()
----------------------

The ``createConfiguredMock()`` method is a convenience wrapper around ``createMock()`` that allows configuring
return values using an associative array (``['methodName' => <return value>]``):

.. literalinclude:: examples/test-doubles/CreateConfiguredMockExampleTest.php
   :caption: Using createConfiguredMock() to create a mock object and configure return values
   :language: php

.. _test-doubles.mocking-traits-and-abstract-classes:

Abstract Classes and Traits
===========================

getMockForAbstractClass()
-------------------------

The ``getMockForAbstractClass()`` method returns a mock
object for an abstract class. All abstract methods of the given abstract
class are mocked. This allows for testing the concrete methods of an
abstract class.

.. literalinclude:: examples/test-doubles/src/AbstractClass.php
   :caption: An abstract class with a concrete method
   :language: php

.. literalinclude:: examples/test-doubles/AbstractClassTest.php
   :caption: A test for a concrete method of an abstract class
   :language: php

.. admonition:: Deprecation: ``getMockForAbstractClass()`` is deprecated

   As of PHPUnit 10.1, the ``getMockForAbstractClass()`` is soft-deprecated,
   meaning its declaration is annotated with ``@deprecated`` so that IDEs and
   static analysis tools can warn about its usage.

   Starting with PHPUnit 11, using the ``getMockForAbstractClass()`` method
   will trigger a deprecation warning. The method will be removed in PHPUnit 12.

getMockForTrait()
-----------------

The ``getMockForTrait()`` method returns a mock object
that uses a specified trait. All abstract methods of the given trait
are mocked. This allows for testing the concrete methods of a trait.

.. literalinclude:: examples/test-doubles/src/AbstractTrait.php
   :caption: A trait with an abstract method
   :language: php

.. literalinclude:: examples/test-doubles/AbstractTraitTest.php
   :caption: A test for a concrete method of a trait
   :language: php

.. admonition:: Deprecation: ``getMockForTrait()`` is deprecated

   As of PHPUnit 10.1, the ``getMockForTrait()`` is soft-deprecated,
   meaning its declaration is annotated with ``@deprecated`` so that IDEs and
   static analysis tools can warn about its usage.

   Starting with PHPUnit 11, using the ``getMockForTrait()`` method
   will trigger a deprecation warning. The method will be removed in PHPUnit 12.

.. _test-doubles.stubbing-and-mocking-web-services:

Web Services
============

When your application interacts with a web service you want to test it
without actually interacting with the web service. To create stubs
and mocks of web services, the ``getMockFromWsdl()`` method can be used.

This method returns a test stub based on a web service description in WSDL whereas
``createStub()``, for instance, returns a test stub based on an interface or on a class.

Here is an example that shows how to stub the web service described in :file:`HelloService.wsdl`:

.. literalinclude:: examples/test-doubles/WsdlStubExampleTest.php
   :caption: Stubbing a web service
   :language: php

.. admonition:: Deprecation: ``getMockFromWsdl()`` is deprecated

   As of PHPUnit 10.1, the ``getMockFromWsdl()`` is soft-deprecated,
   meaning its declaration is annotated with ``@deprecated`` so that IDEs and
   static analysis tools can warn about its usage.

   Starting with PHPUnit 11, using the ``getMockFromWsdl()`` method
   will trigger a deprecation warning. The method will be removed in PHPUnit 12.

MockBuilder API
===============

As mentioned before, when the defaults used by the ``createStub()`` and ``createMock()`` methods
to generate the test double do not match your needs then you can use the ``getMockBuilder($type)``
method to customize the test double generation using a fluent interface. Here is a list of methods
provided by the Mock Builder:

-

  ``onlyMethods(array $methods)`` can be called on the Mock Builder object to specify the methods that are to be replaced with a configurable test double. The behavior of the other methods is not changed. Each method must exist in the given mock class.

-

  ``addMethods(array $methods)`` can be called on the Mock Builder object to specify the methods that don't exist (yet) in the given mock class. The behavior of the other methods remains the same.

-

  ``setConstructorArgs(array $args)`` can be called to provide a parameter array that is passed to the original class' constructor (which is not replaced with a dummy implementation by default).

-

  ``setMockClassName($name)`` can be used to specify a class name for the generated test double class.

-

  ``disableOriginalConstructor()`` can be used to disable the call to the original class' constructor.

-

  ``enableOriginalConstructor()`` can be used to enable the call to the original class' constructor.

-

  ``disableOriginalClone()`` can be used to disable the call to the original class' clone constructor.

-

  ``enableOriginalClone()`` can be used to enable the call to the original class' clone constructor.

-

  ``disableAutoload()`` can be used to disable ``__autoload()`` during the generation of the test double class.

-

  ``enableAutoload()`` can be used to enable ``__autoload()`` during the generation of the test double class.

-

  ``disableArgumentCloning()`` can be used to disable the cloning of arguments passed to mocked methods.

-

  ``enableArgumentCloning()`` can be used to enable the cloning of arguments passed to mocked methods.

-

  ``enableProxyingToOriginalMethods()`` can be used to enable the invocation of the original methods.

-

  ``disableProxyingToOriginalMethods()`` can be used to disable the invocation of the original methods.

-

  ``setProxyTarget()`` can be used to set the proxy target for the invocation of the original methods.

-

  ``allowMockingUnknownTypes()`` can be used to allow the doubling of unknown types.

-

  ``disallowMockingUnknownTypes()`` can be used to disallow the doubling of unknown types.

-

  ``enableAutoReturnValueGeneration()`` can be used to enable the automatic generation of return values when no return value is configured.

-

  ``disableAutoReturnValueGeneration()`` can be used to disable the automatic generation of return values when no return value is configured.

Here is an example that shows how to use the Mock Builder's fluent interface to configure
the creation of a test stub. The configuration of this test double uses the same best
practice defaults used by ``createStub()`` and ``createMock()``:

.. literalinclude:: examples/test-doubles/MockBuilderExampleTest.php
   :caption: Using the Mock Builder API to configure how the test double class is generated
   :language: php

.. admonition:: Deprecation: ``getMockBuilder()`` is deprecated

   As of PHPUnit 10.1, the ``getMockBuilder()`` is soft-deprecated,
   meaning its declaration is annotated with ``@deprecated`` so that IDEs and
   static analysis tools can warn about its usage.

   Starting with PHPUnit 11, using the ``getMockBuilder()`` method
   will trigger a deprecation warning. The method will be removed in PHPUnit 12.
