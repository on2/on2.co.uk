---
title: Collections in PHP
layout: post
---

With PHP being a loosely typed language we don't often concern ourselves with a variable's data
type. In place of formal collections, [arrays](http://php.net/manual/en/language.types.array.php)
are typically used in PHP projects; arrays in PHP are more powerful than in many other languages
and are perfectly sufficient to be used as collections, but errors can occur as we can't specify
what data types are contained in an array.

Here's an example:

{% highlight php startinline %}
class Member
{
    private $name;

    public function __construct($name)
    {
        $this->name = $name;
    }

    public function getName()
    {
        return $this->name;
    }
}

class Service
{
    private $members;

    public function __construct(array $members)
    {
        $this->members = $members;
    }

    public function sayHello()
    {
        foreach ($this->members as $member) {
            echo 'Hello ' . $member->getName() . "\n";
        }
    }
}

$members = [
    new Member('Barry'),
    new Member('Robin'),
    'Maurice',
];

$service = new Service($members);
$service->sayHello();
{% endhighlight %}

When we iterate over `$this->members` we're expecting a `Member` object, but as there are no
restrictions on what data types are in our array we can pass a string causing a fatal error.

## Type Hinting

We're not able to type hint the contents of an array, but we can refactor a little to utilise PHP's
built in [type hinting](http://php.net/manual/en/language.oop5.typehinting.php). By replacing the
`__construct` method with an `add` method we can add each member individually and force the
parameter to be a `Member` object:

{% highlight php startinline %}
class Member
{
    private $name;

    public function __construct($name)
    {
        $this->name = $name;
    }

    public function getName()
    {
        return $this->name;
    }
}

class Service
{
    private $members;

    public function add(Member $member)
    {
        $this->members[] = $member;
    }

    public function sayHello()
    {
        foreach ($this->members as $member) {
            echo 'Hello ' . $member->getName() . "\n";
        }
    }
}

$members = [
    new Member('Barry'),
    new Member('Robin'),
    'Maurice',
];

$service = new Service();
foreach ($members as $member) {
    $service->add($member);
}
$service->sayHello();
{% endhighlight %}

This time, we receive a catchable fatal error informing us that `Maurice` must be a `Member`
instance before we've had chance to `sayHello`. Should this occur it should now be easier to
debug as we know the error occured whilst initialising the class rather than with the `sayHello`
method.

This solution does have its drawbacks though;

- The object isn't fully intialised when it's created;
- It violates the [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single_responsibility_principle).

What would be preferable, is at the time the `Service` is created, we're in a position to know what
data types are contained within the array.

## The Collection Class

We need to seperate the administration of the collection from our service whilst maintaining the
ability to type hint. To achieve this we need to create a collection object, but in order to type
hint we need to extend a base object for each data type we wish to store in the collection. With
more refactoring and making `Maurice` a `Member` we now have the following:

{% highlight php startinline %}
class Member
{
    private $name;

    public function __construct($name)
    {
        $this->name = $name;
    }

    public function getName()
    {
        return $this->name;
    }
}

abstract class BaseCollection implements Iterator
{
    protected $items;

    public function add($item)
    {
        $this->items[] = $item;
    }

    public function current()
    {
        return current($this->items);
    }

    public function key()
    {
        return key($this->items);
    }

    public function next()
    {
        return next($this->items);
    }

    public function rewind()
    {
        reset($this->items);
    }

    public function valid()
    {
        return ($this->current() !== false);
    }
}

class MemberCollection extends BaseCollection
{
    public function add(Member $item)
    {
        parent::add($item);
    }
}

class Service
{
    private $members;

    public function __construct(MemberCollection $members)
    {
        $this->members = $members;
    }

    public function sayHello()
    {
        foreach ($this->members as $member) {
            echo 'Hello ' . $member->getName() . "\n";
        }
    }
}

$members = [
    new Member('Barry'),
    new Member('Robin'),
    new Member('Maurice'),
];

$collection = new MemberCollection();
foreach ($members as $member) {
    $collection->add($member);
}

$service = new Service($collection);
$service->sayHello();
{% endhighlight %}

We've implemented PHP's [Iterator](http://php.net/manual/en/class.iterator.php) so that we're able
to `foreach` through the collection.

Extending a base collection is the approach the authors of
[Professional PHP6](http://www.wrox.com/WileyCDA/WroxTitle/Professional-PHP6.productCd-0470395095.html)
have taken. An article on [Collection Classes in PHP](http://www.sitepoint.com/collection-classes-in-php/)
from SitePoint introduces a similar to approach to collections as described in the book, although
the article does stop short of a complete solution and omits the implementation of an iterator.

We've fixed the drawbacks we experienced earlier, but our collection isn't generic and we need to
extend our base collection each time we want to store a different data type. We are making the best
use of the language's features, but at the cost of having to repeat ourselves.

## Generic Collection Class

We can make our collection class generic by defining the type of object that the collection can
hold in it's constructor. Each time we add an item to the collection we check the type is as
expected. We can't type hint, but we can throw a `InvalidArgumentException` if there's an error.

When our service takes the collection we need to check that it contains the expected types. As we've
checked the type each time we've added an item we can call `getType` to get the types in the
collection. Refactoring our previous example gives us this:

{% highlight php startinline %}
<?php

class Member
{
    private $name;

    public function __construct($name)
    {
        $this->name = $name;
    }

    public function getName()
    {
        return $this->name;
    }
}

class Collection implements Iterator
{
    protected $type;

    protected $items;

    public function __construct($type)
    {
        $this->type = $type;
    }

    public function getType()
    {
        return $this->type;
    }

    public function add($item)
    {
        if (get_class($item) != $this->getType()) {
            throw new InvalidArgumentException(
                'Collection can only contain instances of ' . $this->getType() . ' items'
            );
        }
        $this->items[] = $item;
    }

    public function current()
    {
        return current($this->items);
    }

    public function key()
    {
        return key($this->items);
    }

    public function next()
    {
        return next($this->items);
    }

    public function rewind()
    {
        reset($this->items);
    }

    public function valid()
    {
        return ($this->current() !== false);
    }
}

class Service
{
    private $members;

    public function __construct(Collection $members)
    {
        if ($members->getType() != 'Member') {
            throw new InvalidArgumentException('Collection must contain Member items');
        }
        $this->members = $members;
    }

    public function sayHello()
    {
        foreach ($this->members as $member) {
            echo 'Hello ' . $member->getName() . "\n";
        }
    }
}

$members = [
    new Member('Barry'),
    new Member('Robin'),
    new Member('Maurice'),
];

$collection = new Collection('Member');
foreach ($members as $member) {
    $collection->add($member);
}

$service = new Service($collection);
$service->sayHello();
{% endhighlight %}

It does feel like we're fighting with the language here. There's nothing built-in to PHP to help us
solve this problem.

## Final Thoughts

[Generic programming](https://en.wikipedia.org/wiki/Generic_programming) is ideally suited to our
problem but this isn't something that's implemented in PHP (though it does exist in
[Hack](http://docs.hhvm.com/manual/en/hack.generics.php)). In Java,
[JDK 5 introduced Generics](http://docs.oracle.com/javase/1.5.0/docs/guide/language/index.html):

> [Generics allow] a type or method to operate on objects of various types while providing
> compile-time type safety. It adds compile-time type safety to the Collections Framework and
> eliminates the drudgery of casting

It would be great to eliminate the drudgery of casting but compile-time type safety isn't a concept
in PHP. If we implement our generic collection our IDE will struggle to analyse the code as we're
not type hinting. So whatever we do, if we've not handled our data correctly we'll still encounter a
run-time error. So what exactly are we preventing with our collections? We're in a loosely typed
world, I think we need to give our development team a chance and assume they've followed the
instructions and passed what was required. Let's keep things simple!
