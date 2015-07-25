---
layout: post
title: "Django Form Fields and Validation"
modified: 2015-04-12 15:28:31 -0400
category: []
tags: []
image:
  feature: 
  credit: 
  creditlink: 
comments: 
share: 
---

The form is validated by calling its `is_valid` method. This method call validates the submitted form input, populates the form's `cleaned_data` and `errors` properties and returns a boolean indicating whether or not the form is valid. During the validaiton process, the form iterates through its fields and calls each field's `clean` method. The field's `clean` method either returns the sanitized input or raises a `ValidationError` which the form then presents to the user.

## `clean`ing submitted input

A field's `clean` method is responsible for sanitizing user submitted input and raising an error if the input doesn't pass any of the field's validations. When a field's `clean` method is executed, it runs the following three methods in order: `to_python`, `validate`, and `run_validators`. The `to_python` method is responsible for taking the form field input and coercing it into the appropriate Python object. The `validate` method contains custom validation logic for the field. The `run_validators` method iterates through any additional validators assigned to the field's `validators` list when it is declared in a `Form` class. If any one of these three methods throws a `ValidationError, processing stops and the error is returned to the user.

### The `to_python` method

The `to_python` method is responsible for coercing the field's submitted input into the correct Python object. This method takes a single value parameter: a `String` object containing the user submitted form field value. Any errors related to this coercion should be caught and a `ValidationError` thrown instead. If a `ValidationError` is thrown here, the execution of `clean` stops and no validators are run on the input. The coerced value is then submitted to the `validate` and `run_validators` methods.

### The `validate` method

The `validate` method contains any custom field validation logic that is not better suited for a custom validator. It is primarily used for validating that a required field exists. However, the built-in `DecimalField` class also uses the `validate` method to verify that a submitted value is not infinity.

### The `run_validators` method

The `run_validators` method runs any additional validators declared in the field's `validators` list. Django ships with several built-in validators in the `core.validators` module for doing things like verifying that a string matches a regular expression or that a value is within a certain range. It is important to note that these validators don't change the value's type.


