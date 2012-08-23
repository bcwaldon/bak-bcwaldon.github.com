---
layout: post
title: Using Warlock &amp; JSON Schemas
tags:
- warlock
- python
- jsonschema
- github
- openstack

---
The `warlock` [python library](http://github.com/bcwaldon/warlock) was created out of a need to interact with [JSON schemas](http://json-schema.org) in a pythonic way. It is designed to generate self-validating python classes that conform to a JSON schema definition.

## Installing warlock

Install it on Ubuntu 12.10:

    $ apt-get install python-warlock

Install it using pip:

    $ pip install python-warlock

Pull it from GitHub:

    $ git clone http://github.com/bcwaldon/warlock
    $ cd warlock/
    $ python setup.py install

## Using warlock v0.4.0

The first thing you'll need is a JSON schema. The following is a simple example, but warlock can support any valid JSON schema you throw at it:

    >>> schema = {
        'name': 'Country',
        'properties': {
            'name': {'type': 'string'},
            'population': {'type': 'integer'},
        },
    }
    
Creating a new warlock model is easy:

    >>> import warlock
    >>> Country = warlock.model_factory(schema)
   

The `model_factory` call returns your self-validating warlock model. Create a new instance of your model by passing in the necessary data:

    >>> sweden = Country(name='Sweden', population=9453000)

Get and set attributes using dot-syntax or dict-syntax:

    >>> sweden['name']
    'Sweden'
    >>> sweden.population = 9782000
    >>> sweden['population']
    9782000

The power of warlock is in the automatic validation. Trying to change an attribute of `sweden` to something the schema won't allow results in raising a `warlock.core.InvalidOperation` exception:

    >>> sweden.population = 'N/A'
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "warlock/core.py", line 63, in __setattr__
        self.__setitem__(key, value)
      File "warlock/core.py", line 58, in __setitem__
        raise InvalidOperation()
    warlock.core.InvalidOperation

Attempting to create an instance of Country with a bad value for `name` raises a `ValueError`:

    >>> Country(name=None, population=10)
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "warlock/core.py", line 37, in __init__
        raise ValueError()
    ValueError

Accessing an attribute that doesn't exist raises an `AttributeError`:

    >>> sweden.abbreviation
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "warlock/core.py", line 47, in __getattr__
        raise AttributeError(key)
    AttributeError: abbreviation

The `changes` attribute of your model instance returns a dictionary containing only the modified attributes:

    >>> norway = Country(name='Norway', population=4952000)
    >>> norway.changes
    {}
    >>>norway.population = 5013000
    >>> norway.changes
    {'population': 5013000}

## Client-side magic

The use case that drove the creation of warlock was to be able to automagically verify client-side modifications to an entity before sending it back to a REST-like API. The [python client](http://github.com/openstack/python-glanceclient) we provide for v2 of the OpenStack Images API downloads a JSON schema and createa a warlock model representing a virtual image. The following shows how a user can download and modify an image using that python library:

    >>>import glanceclient
    >>>client = glanceclient.Client(…)

    # An instance of a warlock model is returned below after retrieving
    # the JSON schema and specific image data over HTTP
    >>>image = client.images.get('12')
    
    # Sample changes a client might make
    >>> image.tags.append('ubuntu')
    >>> image['min_ram'] = 512
    
    >>>client.images.update(image)
    
The actual `glanceclient` code to fetch and update the image using `warlock` is as simple as this:

    import warlock
    import http
    
    schema = http.get_schema('image')
    model = warlock.model_factory(schema)
    
    def get(image_id):
        image = http.get_image(image_id)
        return model(image)
    
    def update(image):
        http.update_image(image.changes)

## Future of warlock

There are a couple of major things on my wishlist at the moment:

* **Deleted Attributes:** the 'changes' feature doesn't allow a user to remove attributes from an image. This is absolutely necessary when modifying entities.
* **Cleaner Exceptions:** the mix of warlock-specific and standard python exceptions isn't a great experience. The code should use standard exceptions as much as possible and ensure it is returning the proper information.
