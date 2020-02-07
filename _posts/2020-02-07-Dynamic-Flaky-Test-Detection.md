---
layout: post
title: Dynamic Variant Analysis for Flaky Test Detection
---

[Last post](/Static-Variant-Analysis/) was about finding flaky test by using static analysis. This time I'll be doing the same thing but dynamically using the technique described on my [first post](/Dinamic-Variant-Analysis/)

## How am I going to do that?

I will be instrumenting the ***\__getitem__*** method of the Django Queryset class.

What I'll be doing is saving all the indexes used to reference the queryset into a set. If after the test has completed running, the number of unique index references within the set is more than 2 then that means the test could be flaky, if you don't understand why, check the previous post :)

Note also that I'm only saving the indexes if the QuerySet is not ordered, because in that case the test is not going to be flaky.

To avoid lots of false positives I'm only going to be saving the index if the call to the \__getitem__ method happens withing the file that contains the tests, as you can imagine the \__getitem__ method for a QuerySet gets called all over the place.  

## The instrumentation

{% highlight python %}

def custom_getitem(self, k):
    # Only process integer indexes if the queryset is not ordered
    if isinstance(k, int) and not self.ordered:

        call_stack = traceback.extract_stack()
        caller_filename = os.path.basename(call_stack[-2].filename)

        # Only count the call if it was called within the file that contains the tests
        if caller_filename.startswith('test_'):
            data.add(k)

    # Call the original __getitem__ otherwise
    return django.db.models.query.QuerySet.original_getitem(self, k)


@pytest.yield_fixture(autouse=True)
def flaky_detect(monkeypatch):
    global data
    data = set() # Saves all the integer references to the queryset

    django.db.models.query.QuerySet.original_getitem = django.db.models.query.QuerySet.__getitem__
    monkeypatch.setattr(django.db.models.query.QuerySet, '__getitem__', custom_getitem)

    yield # Run the test

    if len(data) >= 2:
        raise Exception('Possible flaky test detected.')
        
{% endhighlight %}


If you only have experience with Python unitest module and have not used pytest before, let me explain the way the flaky_detect fixture works, as we saw before this fixture will be executed on each test because it has autouse set to True. 

The ***yield*** statement is where the actual test will be executed, the stuff that is before the yield is like the setUp of the test and the stuff after the yield is like the tearDown of the test.

So, before the test runs we apply the instrumentation by replacing the \__getitem__ implementation and defining a data variable which will hold the references to the indexes used while accessing the queryset, and after the test runs we perform the checks to see if the test is flaky.   

## Comparing the two approaches
If we compare the results of this approach versus the previous one we can see that we managed to detect the flaky tests that the previous one failed to detected.

This approach happens to be really simple and easy to implement compared to the static approach as well. 

One of the drawbacks of the dynamic approach we saw last time was that you needed good test coverage in order to be able to find more bugs on the code, but since this time we are finding bugs on the tests themselves, this means this time we are not missing anything if the project coverage is not high enough.

I have not been able to come up with any false positive that might be generated with this approach, it might still return false positives, but I believe the ratio should be prety low compared to static analysis.

If you can identify the pattern that's making your tests flaky and you can come up with a way to instrument your system so that your test fails when they follow the pattern, finding flaky test gets so much easier, because you can remove the non-determinism.
You can even detect tests that are flaky but haven't failed yet. 

## Further Steps
Taking this further you can also create a custom pipeline (not the main pipeline since we are modifying the way the system works, and we don't want to mess up or tests) on your CI to run your tests with this instrumentation and prevent flaky tests to entering the codebase (merge to dev could be a good trigger for this). 

## Code
The code for this example lives in the same repository used for the [static variant analysis post](https://github.com/daviddanielarch/flaky_tests_variant_analysis/), and the actual instrumentation is at the [conftest.py](https://github.com/daviddanielarch/flaky_tests_variant_analysis/blob/master/flaky/example/conftest.py) file.
