---
layout: post
title: "Use Django's Class-Based Views with Bootstrap Modals"
excerpt: Render a Django view in a Boostrap modal using a little Python and JavaScript.
modified: 2014-05-23 15:45:45 -0400
category: posts
tags: [django, python, views, forms, class based, bootstrap, modals, jquery, ajax, javascript]
image:
  feature:
  credit:
  creditlink:
comments:
share:
---

[Twitter Bootstrap](http://getbootstrap.com) contains an exhaustive set of CSS
classes, components, and jQuery plugins that are responsive across a broad array
of devices. When combined with the [Django web
framework](http://djangoproject.com), creating responsive, data-driven websites
becomes quick and easy for Python developers.

The modal dialog box is one of the components Bootstrap provides. Since Django's
class-based views makes it easy to define templates and forms with complex
validation rules, using Django to generate the contents of these modal dialog
boxes has become a common task.

<figure>
<a href="/images/bootstrap-modal.png"><img src="/images/bootstrap-modal.png"></a>
<figcaption>A typical Bootstrap modal dialog box</figcaption>
</figure>

[Many](http://stackoverflow.com/questions/11276100/how-do-i-insert-a-django-form-in-twitter-bootstrap-modal-window)
[different](http://stackoverflow.com/questions/13394057/django-ajax-modal-login-registration)
[methods](https://groups.google.com/forum/#!topic/twitter-bootstrap-stackoverflow/6Cpxw1Ji_E8)
exist for accomplishing this task, but these solutions often require duplicating
template code, don't address rendering the same view as both a standalone page
and the contents of a modal, or don't account for redirects when submitting
forms.

I recently found myself trying to render a Django view as both a modal and a
standalone page and had the following requirements:

 - minimize code repetition
 - update the modal with any form errors
 - close the modal on successful submission

I was able to accomplish this task with a few lines of Python, a few lines of
JavaScript, and a minor template change.

## Server-Side Changes

The server-side changes consisted of creating an `AjaxTemplateMixin` to render a
different template for AJAX requests and making a small change to an existing
template.

### An AjaxTemplateMixin

{% highlight python linenos %}
class AjaxTemplateMixin(object):

    def dispatch(self, request, *args, **kwargs):
        if not hasattr(self, 'ajax_template_name'):
            split = self.template_name.split('.html')
            split[-1] = '_inner'
            split.append('.html')
            self.ajax_template_name = ''.join(split)
        if request.is_ajax():
            self.template_name = self.ajax_template_name
        return super(AjaxTemplateMixin, self).dispatch(request, *args, **kwargs)
{% endhighlight %}

The first step required writing a mixin to add an `ajax_template_name` attribute
Django's class-based views. If this attribute is not explicitly defined, it will
default to adding `_inner` to the end of the `template_name` attribute.  For
example, take the following FormView class:

{% highlight python linenos %}
class TestFormView(SuccessMessageMixin, AjaxTemplateMixin, FormView):
    template_name = 'test_app/test_form.html'
    form_class = TestForm
    success_url = reverse_lazy('home')
    success_message = "Way to go!"
{% endhighlight %}

In this example, the `ajax_template_name` defaults to
`test_app/test_form_inner.html`.  If the request is AJAX, then the view renders
this template. Otherwise, the view renders the `test_app/test_form.html`
template.

### Create the AJAX Template

Now that the view will render `ajax_template_name` for AJAX requests we have to
create it. This template could be unique, but more than likely it will be the
same as `template_name` but without extending the base template containing the
site's header, navigation, and footer. This could be as simple as changing
`test_app/test_form.html` from:

{% highlight html+django linenos %}
{% raw %}
{% extends 'test_app/home.html' %}

{% block content %}
{% load crispy_forms_tags %}
<div class="row">
    <form class="form-horizontal" action="{% url 'test-form' %}" method="post">
        {% crispy form %}
        <input type="submit" class="btn btn-submit col-md-offset-2">
    </form>
</div>
{% endblock content %}
{% endraw %}
{% endhighlight %}

to:

{% highlight html+django linenos %}
{% raw %}
{% extends 'test_app/home.html' %}

{% block content %}
{% include 'test_app/test_form_inner.html' %}
{% endblock content %}
{% endraw %}
{% endhighlight %}

and creating `test_app/test_form_inner.html` containing:

{% highlight html+django linenos %}
{% raw %}
{% load crispy_forms_tags %}
<div class="row">
    <form class="form-horizontal" action="{% url 'test-form' %}" method="post">
        {% crispy form %}
        <input type="submit" class="btn btn-submit col-md-offset-2">
    </form>
</div>
{% endraw %}
{% endhighlight %}

All we've done here is moved the HTML within the content block to its own
template. The example template uses
[django-crispy-forms](http://django-crispy-forms.readthedocs.org/en/latest/) to
generate the form markup using Bootstrap CSS classes but this is not a
requirement.

## Front-end Changes

At this point, rendering your view is easy, unless it contains a form.

### Rendering a View in a Modal the Easy Way

Given the following modal in your HTML:

{% highlight html linenos %}
<div class="modal fade" id="form-modal" tabindex="-1" role="dialog" aria-labelledby="myModalLabel" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <button type="button" class="close" data-dismiss="modal" aria-hidden="true">&times;</button>
        <h4 class="modal-title">Modal title</h4>
      </div>
      <div id="form-modal-body" class="modal-body">
        ...
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-default" data-dismiss="modal">Close</button>
      </div>
    </div>
  </div>
</div>
{% endhighlight %}

rendering a Django view in it can be as simple as adding:

{% highlight html+django linenos %}
{% raw %}
<a data-toggle="modal" href="{% url 'test-form' %}" data-target="#form-modal">Click me</a>
{% endraw %}
{% endhighlight %}

to your template. Behind the scenes, Bootstrap is using the data attributes to
call jQuery's `.load()` method to make an AJAX call to the `test-form` url and
replace the HTML within `#form-modal`. However, there are a couple problems with
this:

 - Using data attributes replaces the entire contents of the modal, so your
template will need to contain the `.modal-dialog`, `.modal-content` and
`.modal-body` DIVs to render properly.
 - jQuery's `.load()` is only called once the first time the modal is opened.
 - Any redirects that occur, such as from submitting a form, will redirect the
entire page.

If none of this matters to you, then great, you're done! Otherwise, keep
reading.

### Rendering a View in a Modal the Slightly Harder Way

The following JavaScript solves the problems above:

{% highlight javascript linenos %}
var formAjaxSubmit = function(form, modal) {
    $(form).submit(function (e) {
        e.preventDefault();
        $.ajax({
            type: $(this).attr('method'),
            url: $(this).attr('action'),
            data: $(this).serialize(),
            success: function (xhr, ajaxOptions, thrownError) {
                if ( $(xhr).find('.has-error').length > 0 ) {
                    $(modal).find('.modal-body').html(xhr);
                    formAjaxSubmit(form, modal);
                } else {
                    $(modal).modal('toggle');
                }
            },
            error: function (xhr, ajaxOptions, thrownError) {
                // handle response errors here
            }
        });
    });
}
$('#comment-button').click(function() {
    $('#form-modal-body').load('/test-form/', function () {
        $('#form-modal').modal('toggle');
        formAjaxSubmit('#form-modal-body form', '#form-modal');
    });
}
{% endhighlight %}

This code binds to the click event on `#comment-button` and loads the
`/test-form/` HTML asynchronously into the body of the modal. Since this is
an AJAX call, the `test_form/test_form_inner.html` template will be rendered and
the form will be displayed without any site navigation or footer.

Additionally, this code also calls `formAjaxSubmit()`.  This function binds to
the form's submit event. By calling `preventDefault()`, the callback function
prevents the form from performing its default submit action. Instead, the form's
content is serialized and sent via an AJAX call using the form's defined
`action` and `method`.

If the server sends back a successful response, the success function is
called. The `xhr` parameter contains the HTML received from the server.  Note
that a successful response from the server does not mean that the form validated
successfully.  Therefore, `xhr` is checked to see if it contains any field
errors by looking for the `has_error` Bootstrap class in its contents. If any
errors are found, the modal's body is updated with the form and its errors.
Otherwise, the modal is closed.

<figure>
<a href="/images/bootstrap-modal-with-errors.png"><img src="/images/bootstrap-modal-with-errors.png"></a>
<figcaption>A form with errors in a Bootstrap modal dialog box</figcaption>
</figure>

## Summary

In conclusion, we were able to define our form and view in Django and render it
correctly in a Bootstrap modal by adding one short Django mixin, making a small
change to an existing template, and adding a few lines of JavaScript.

The example site used in this post is hosted [on
GitHub](https://github.com/d-m/django-modal-forms).
