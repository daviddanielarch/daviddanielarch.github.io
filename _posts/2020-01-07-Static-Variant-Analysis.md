---
layout: post
title: Static Variant Analysis with Python AST
---

[Last post](https://daviddanielarch.github.io/Dinamic-Variant-Analysis/) was about dynamic variant analysis with Python, this time I'll be performing static variant analysis on Python code by using the [ast module](https://docs.python.org/3.7/library/ast.html) in order to find flaky tests. 

## The problem
In case you are unfamiliar with the term, a flaky test is a test that passes sometimes and others don't. There are many reasons for a test to be flaky.
As an example, given this function that returns a random number between 1 and 10:
{% highlight python %}

import random
def randomize():
    return random.randint(1,10)

{% endhighlight %}

A flaky test for it can be the following:
{% highlight python %}

def test_randomize():
    number = randomize()
    assert number < 5

{% endhighlight %}

This test will pass 50% of the times (when randomize returns a number between 1 and 5) and will fail the other 50% of the times (when randomize returns a number between 6 and 10).

Detecting flaky tests can be hard sometimes because the fail rate can be really low which makes it really hard to debug.
If you are wondering what is the problem with flaky tests, I'd encourage you to check [this](https://martinfowler.com/articles/nonDeterminism.html) article.
     
## The Django pattern

We will see one simple pattern for Django tests that might turn them into flaky tests.

The approach is going to be pretty simple, the pattern I will be looking this time is tests that look like this.

{% highlight python %}

class TestFlaky(unittest.TestCase):
    @pytest.mark.django_db
    def test_flaky(self):
        TheModel.objects.create(name=1)
        TheModel.objects.create(name=2)
        TheModel.objects.create(name=3)
        TheModel.objects.create(name=4)

        models = TheModel.objects.all()

        self.assertEqual(models[0].name, 1)
        self.assertEqual(models[1].name, 2)
        self.assertEqual(models[2].name, 3)
        self.assertEqual(models[3].name, 4)
    
{% endhighlight%}

Note that this pattern assumes that you are using the Python unittest framework for testing your code.
 
Let's see why this test is flaky in the first place. As stated on the [Django documentation](https://docs.djangoproject.com/en/3.0/ref/models/options/#ordering), 
there is no default ordering when fetching items from the database. So by fetching more than one item from the database 
and trying to assert some properties on these, assuming some kind of default ordering, will lead to a flaky test. 
 In the example ***models[0], models[1], models[2], models[3]*** might be different on each test run.

## How are we going to find the variants?

We are going to build a function that looks for this pattern by using the python ast module.
What we are going to be doing is basically:

1) Look for assignments in which the right side of the expression is a Django queryset call and save the variable name, e.g
 {% highlight python %}

    models = TheModel.objects.all()

{% endhighlight%}

We will be saving ***models*** in this case.

2) Check for calls to self.assertEqual in which one of the expressions is an indexing of one of the variables found on Step 1, and save the index numbers e.g
 {% highlight python %}
 
    self.assertEqual(models[0].name, 1)
    
{% endhighlight%}

In this case ***models[0]***, so we would be saving the index 0. We will end up with a mapping which looks like this:
{% highlight python %}
    {
        'models': [0]
    }
{% endhighlight%}

in which the keys are the variable names gathered on Step 1 and the values are all the indexes found on that variable. 

3) Finally, if there is more than 1 assertEqual call using different indexes for any of the variables we gathered on Step 1, we will mark the test as flaky. 
 In our mapping that would look like:
 {% highlight python %}
    {
        'models': [0, 1, 2]
    }
{% endhighlight%}
    
The high level algorithm is this one and can be found [here](https://github.com/daviddanielarch/static_variant_analysis/blob/master/flaky/check/analyze.py) as well.

 {% highlight python %}
 
 def is_flaky(node):
    variable_names = set()
    variable_usage_indexes = defaultdict(list)

    for child in node.body:
        # Step 1
        if isinstance(child, ast.Assign):
            if is_model_queryset_call(child.value):
                for target in child.targets:
                    variable_names.add(get_variable_full_name(target))

        # Step 2
        if is_assert_equal(child):
            args = child.value.args
            for arg in args:
                if has_numeric_subscript(arg):
                    subscript = get_subscript(arg)
                    name, index = get_numeric_subscript_variable_name_and_index(subscript)

                    if name in variable_names:
                        variable_usage_indexes[name].append(index)

    # Step 3
    return any([len(set(usages)) >= 2 for usages in variable_usage_indexes.values()])

{% endhighlight%}

## Static analysis drawbacks

One of the main issues with static analysis is that usually the number of false positives is high, there are many reason for these, some of these are:
* Every function working on ast object needs to keep in mind all the possible variations of the syntax, and there are a [lot](https://docs.python.org/3.7/library/ast.html#abstract-grammar) of them
* There are a lot of ways to do the same thing (specially in dynamic languages like Python).
* Analyzing how a program behaves without actually executing it is complicated by itself, there are many techniques that help with these like Abstract Interpretation, Data-flow analysis, symbolic execution, etc.

As an example of a test that does the exact same thing as the test presented at the beginning, but does not follow the pattern
we are looking we have this test:
 {% highlight python %}
     @pytest.mark.django_db
    def test_also_flaky_not_detected(self):
        TheModel.objects.create(name=1)
        TheModel.objects.create(name=2)

        models = TheModel.objects.all()

        model1 = models[0]
        model2 = models[1]

        self.assertEqual(model1.name, 1)
        self.assertEqual(model2.name, 2)
{% endhighlight%}

Note that it has the exact same semantic but a different syntax, which makes it go unnoticed when running our flaky check on it.

## Reducing the false positives 

In order to try to reduce the number of false positives a bit I'm also adding an (optional) dynamic check. 
The check consists on checking if the model has an ordering defined, since it is possible to define the ordering for a Django model in the model definition like:
 {% highlight python %}
 
class OrderedModel(models.Model):
    name = models.IntegerField()

    class Meta:
        ordering = ['name']

{% endhighlight%}


If that's the case when we grab a Queryset of OrderedModel, the results will be ordered by `name` unless we explicitly tell it otherwise.
Since the returned items are ordered a test following this pattern with this model will not be flaky.

All this means if we take the test at the beginning of the blogpost and replace the model with OrderedModel, it will not show up on the analysis results.

To enable this dynamic check all the models for the app need to be imported at [model_helpers.py]().  


## How do I use this thing
 
In order to run the flaky test detection I created a Django management command that takes as single argument: the path to start looking for
tests recursively. 

The criteria to decide if a file needs to be processed will be pretty simple, if the filename starts with the prefix `test` we will analyze it. 

Let's see an example run on the sample app I created to test this:
 {% highlight python %}

davida@davida-laptop:~/flaky$ python manage.py check_flaky example/

Found flaky test: test_flaky (example/test_flaky.py)

Found flaky test: test_flaky_with_self_vars (example/test_flaky.py)
{% endhighlight%}

The first flaky test found is the one presented at the beginning of the blogpost, and the last is a small variation
in which the queryset is stored in a class attribute instead of a local variable. 
 
The full code can be found [here](https://github.com/daviddanielarch/flaky_tests_variant_analysis)

In the next post I'll show how to do the same check but dynamically instead of statically using the same technique used on the previous post.
