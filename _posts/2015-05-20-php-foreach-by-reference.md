---
title: PHP foreach by Reference
layout: post
---

Today, I came across a little gotcha in PHP which I find tends to rear it's head every once and a
while. The symptoms are apparent when iterating through an array using `foreach`. You'll find that
the penultimate element is identical to the last element and that the original last element has been
removed from the array.

Here's an example:

{% highlight php startinline %}
$people = [
    ['name' => 'Daphne', 'score' => 0],
    ['name' => 'Portia', 'score' => 0.25],
    ['name' => 'Agnus', 'score' => 0.5],
    ['name' => 'Denise', 'score' => 0.75],
    ['name' => 'Gary', 'score' => 1],
];

foreach ($people as &$person) {
    $person['score'] *= 100;
}

foreach ($people as $person) {
    printf(
        "%s = %d%%\n",
        $person['name'],
        $person['score']
    );
}
{% endhighlight %}

In our first iteration we're converting each person's score into a percentage. On the second
iteration we're outputing the data from the array. This will output:

```
Daphne = 0%
Portia = 25%
Agnus = 50%
Denise = 75%
Denise = 75%
```

Denise's score is shown twice and Gary is not shown at all! From the outset it appears to be a bug
in PHP, but upon closer inspection the issue is from referencing `$person` in the first iteration.

The [PHP manual page for `foreach`](http://php.net/manual/en/control-structures.foreach.php) warns
about this:

> Reference of a `$value` and the last array element remain even after the foreach loop. It is
> recommended to destroy it by `unset()`.

During the second iteration `$person` is still a reference to the last element in the array, so
each time we're iterating over `$people` the last element is changing. When it comes to iterating
over the last element in the array it's set as the penultimate element, which is why we see it
twice.

This can be seen is this example:

{% highlight php startinline %}
$people = ['Daphne', 'Portia', 'Agnus', 'Denise', 'Gary'];

foreach ($people as &$person) {
    printf(
        "[%s] %s\n",
        $person,
        implode(', ', $people)
    );
}

echo "\n";

foreach ($people as $person) {
    printf(
        "[%s] %s\n",
        $person,
        implode(', ', $people)
    );
}
{% endhighlight %}

During the first iteration, the `$people` array is unchanged, but in the second the last element
as the reference of `$person` keeps changing:

```
[Daphne] Daphne, Portia, Agnus, Denise, Gary
[Portia] Daphne, Portia, Agnus, Denise, Gary
[Agnus] Daphne, Portia, Agnus, Denise, Gary
[Denise] Daphne, Portia, Agnus, Denise, Gary
[Gary] Daphne, Portia, Agnus, Denise, Gary

[Daphne] Daphne, Portia, Agnus, Denise, Daphne
[Portia] Daphne, Portia, Agnus, Denise, Portia
[Agnus] Daphne, Portia, Agnus, Denise, Agnus
[Denise] Daphne, Portia, Agnus, Denise, Denise
[Denise] Daphne, Portia, Agnus, Denise, Denise
```

The answer? If we unset `$person` after the first iteration, this removes the reference and
everything works as we'd expect:

{% highlight php startinline %}
unset($person);
{% endhighlight %}
