---
layout: post
title: Dynamic Variant Analysis with Python
---

In this post I will present a way to find performance issues withing Python code with the help of pytest by doing variant analysis.


> Variant analysis is the process of using a known vulnerability as a seed to find similar problems in your code.


After checking some of the work [Semmle](https://semmle.com/variant-analysis) is doing regarding variant analysis I started wondering if there was a way to use the same technique for non-security related problems in code, even more, I started wondering if there was any easier way to accomplish the same goal without having to parse the python code, generating an AST and then running the analysis.

That’s when I remembered the monkeypatch example for preventing remote operations for the requests library on the [pytest monkeypatch docs](https://docs.pytest.org/en/latest/monkeypatch.html#global-patch-example-preventing-requests-from-remote-operations)
and realized that I could perform some sort of dinamic variant analysis by instrumenting functions, classes or methods with a custom implementation via monkeypatching. This way I could add additional security, performance or other type of checks to the original functions.

Doing dinamic variant analysis, has a nice benefit over static variant analysis, and that is that the number of false positives found will be close to 0.

In order to do this dynamic analysis feasible we would need a way to automatically exercise all the potentially vulnerable places in the code.
This might look like a problem at first sight, but is actually not for most projects, since these days most projects count with test suites that should execute a good percentage of the codebase, even more they usually rely on CI system to run this test suites effortlessly.
If the test coverage is high, doing analysis in this way should not be a problem at all.

### The goal

I’m going to focus on performance related issues here, in particular finding performance issues on Django applications. 
Although this example will be targeting Django, this technique could be used by any python project with pytest support.

The concrete bug I will be chasing is calls to the django’s length templatetag with a queryset. As can be seen on the [django documentation](https://docs.djangoproject.com/en/2.2/ref/models/querysets/#when-querysets-are-evaluated), calling length on 
a queryset triggers a full evaluation of the queryset which might be a performance hit when the queryset is big enough.
This kind of bug might go unnotice because:

* The database used for local development is usually a subset of the one used on production, so evaluating a small queryset is fast enough to go unseen.
* Most projects at an early stage will probably have a small database, but as time passes by and the database increases its size, the performance will be slower and slower.


## Show me the code

### The buggy template

{% highlight html %}
{% raw %}
<html>
...

Total elements {{ data|length }}

...
</html>
{% endraw %}
{% endhighlight %}

<h3 id="the-view-calling-the-template">The view calling the template</h3>

{% highlight python%}

def buggy_template(request):
    # This might return 2**32 elements
    data = TestModel.objects.all()  
    return render(request, 'buggy.html', locals())
    
{% endhighlight%}

### The test triggering the call to the templatetag

{% highlight python %}

def test_buggy(client):
    response = client.get('/buggy')
    assert response.status_code == 200
    
{% endhighlight%}

<h3 id="the-instrumentation">The instrumentation</h3>
{% highlight python %}
import django
import pytest
from django.db.models import QuerySet
 
 
def queryset_check_length(value):
    """
    New |length implementation which checks for a queryset instance value type and raises an error
    """
    if isinstance(value, QuerySet):
        raise Exception('Calling length with a QuerySet')
    else:
        # call the default length templatetag
        return django.template.defaultfilters.length(value) 
 
@pytest.fixture(autouse=True) 
def template_length_check(monkeypatch):
    # Replace the length templatetag implementation with a custom one
    monkeypatch.setitem(django.template.defaultfilters.register.filters, 'length', queryset_check_length)
{% endhighlight %}

Lets explain what this little piece of code is doing. First of all I’m defining a pytest fixture `template_length_check` with `autouse=True` 
this will cause the fixture to be called before the execution of each test, which means that each test will be run 
with our custom implementation of the length templatetag, `queryset_check_length`

#### What do we want our custom implementation to do?
We want it to raise an exception in case the argument used to call the length templatetag is of type QuerySet, 
otherwise it will call the default length templatetag implementation. Raising an exception will cause the test to fail, so we will be able to find all the buggy templates by checking the failing tests.

> Keep in mind that we are doing this only for finding bugs, having this kind of fixtures on your test suite is discouraged, since we are modifying the original behaviour of the function.

### Results
When running the test suite, this is how the error looks like:

{% highlight python %}
ERROR    django.request:log.py:228 Internal Server Error: /buggy
Traceback (most recent call last):
  File "/home/davida/dinamic_variant_analysis/env/lib/python3.6/site-packages/django/core/handlers/exception.py", line 34, in inner
    response = get_response(request)
  File "/home/davida/dinamic_variant_analysis/env/lib/python3.6/site-packages/django/core/handlers/base.py", line 115, in _get_response
    response = self.process_exception_by_middleware(e, request)
  File "/home/davida/dinamic_variant_analysis/env/lib/python3.6/site-packages/django/core/handlers/base.py", line 113, in _get_response
    response = wrapped_callback(request, *callback_args, **callback_kwargs)
  File "/home/davida/dinamic_variant_analysis/example/variant/views.py", line 8, in buggy_template
    return render(request, 'buggy.html', locals())
  File "/home/davida/dinamic_variant_analysis/env/lib/python3.6/site-packages/django/shortcuts.py", line 36, in render
    content = loader.render_to_string(template_name, context, request, using=using)
  File "/home/davida/dinamic_variant_analysis/env/lib/python3.6/site-packages/django/template/loader.py", line 62, in render_to_string
    return template.render(context, request)
  File "/home/davida/dinamic_variant_analysis/env/lib/python3.6/site-packages/django/template/backends/django.py", line 61, in render
    return self.template.render(context)
  File "/home/davida/dinamic_variant_analysis/env/lib/python3.6/site-packages/django/template/base.py", line 171, in render
    return self._render(context)
  File "/home/davida/dinamic_variant_analysis/env/lib/python3.6/site-packages/django/test/utils.py", line 96, in instrumented_test_render
    return self.nodelist.render(context)
  File "/home/davida/dinamic_variant_analysis/env/lib/python3.6/site-packages/django/template/base.py", line 937, in render
    bit = node.render_annotated(context)
  File "/home/davida/dinamic_variant_analysis/env/lib/python3.6/site-packages/django/template/base.py", line 904, in render_annotated
    return self.render(context)
  File "/home/davida/dinamic_variant_analysis/env/lib/python3.6/site-packages/django/template/base.py", line 987, in render
    output = self.filter_expression.resolve(context)
  File "/home/davida/dinamic_variant_analysis/env/lib/python3.6/site-packages/django/template/base.py", line 698, in resolve
    new_obj = func(obj, *arg_vals)
  File "/home/davida/dinamic_variant_analysis/example/variant/conftest.py", line 11, in queryset_check_length
    raise Exception('Calling length with a QuerySet')
Exception: Calling length with a QuerySet

{% endhighlight %}

The full code example can be found [here](https://github.com/daviddanielarch/dinamic_variant_analysis)
