Resources are models relating to individual resources in an HTTP API.

## class BaseResource

A simple representation of a resource.

**Example:**
```python
# my_resources.py
from beckett import resources


class PersonResource(resources.BaseResource):
    class Meta:
        name = 'Person'
        resource_name = 'people'
        identifier = 'url'
        attributes = (
            'name',
            'birth_year',
            'eye_color',
            'gender',
            'height',
            'mass',
            'url',
        )
        subresources = {
            "books": BookSubResource
        }
        valid_status_codes = (
            200,
        )
        methods = (
            'get',
        )
```

### Meta Attributes

| Attribute            | Required | Type                                                    | Description                                                                                                                                                                                                              |
|:---------------------|:---------|:--------------------------------------------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `name`               | Yes      | String                                                  | The name of this resource instance. Usually a singular noun.                                                                                                                                                             |
| `resource_name`      | No       | String                                                  | The name of this resource used in the url. Usually a plural noun. If not set, we'll attempt to make a pluralised version of the `name` attribute.                                                                        |
| `identifier`         | Yes      | Int/String                                              | The key attribute that can be used to identify this attribute. Used when referring to related resources.                                                                                                                 |
| `attributes`         | Yes      | Tuple of Strings                                        | A tuple list of strings, referring to the key attributes that you want to populate the resource instances with. You can use this for whitelisting and versioning changes in your API.                                    |
| `subresources`       | None     | Dictionary of [SubResource](#class-subresource) classes | A dictionary of [SubResource](#class-subresource) classes. These are complex subresources in your resource that you want to represent as typed isntances.                                                                |
| `valid_status_codes` | No       | Tuple of Ints                                           | A tuple list of integers, referring to the HTTP status codes that are considered "acceptable" when communicating with this resource. If a status code is received that does not match this set, an error will be raised. |
| `methods`            | No       | Tuple of Strings                                        | A tuple list of strings, referring to the HTTP methods that can be used with this resource. For each method, a python method will be generated on the client that registers this resource.                               |
| `pagination_key`     | No       | String                                                  | The key used to look up paginated responses. The value of this key in an API response will be rendered into instances of this resource. See [Pagination](/advanced/#pagination) for more help.                           |


### Customisable Methods

The BaseResource has methods that can be subclassed and customised:

* [BaseResource.get_url](/advanced/#customising-resource-urls)

### URL Generation

Beckett attempts to auto generate URLs based on good RESTful style URI schemes. If you do nothing to manipulate the URLs, Beckett will call the following URLs for the related HTTP Methods:

| Method | URI Structure                                      | Example                            |
|:-------|:---------------------------------------------------|:-----------------------------------|
| GET    | Client.Meta.base_url/Resource.Meta.plural_name/uid | `http://myapi.com/api/products/1/` |
| PUT    | Client.Meta.base_url/Resource.Meta.plural_name/uid | `http://myapi.com/api/products/1/` |
| POST   | Client.Meta.base_url/Resource.Meta.plural_name     | `http://myapi.com/api/products`    |
| PATCH  | Client.Meta.base_url/Resource.Meta.plural_name/uid | `http://myapi.com/api/products/1/` |
| DELETE | Client.Meta.base_url/Resource.Meta.plural_name/uid | `http://myapi.com/api/products/1/` |

URL structures can be completely modified by subclassing the `get_url` method on Resources. See [customising resource urls](advanced/#customising-resource-urls) for more information.

### Assigning properties

properties are assigned to the generated class instances based on the JSON data returned. Consider the following JSON data:

```json
{
    "name": "luke skywalker",
    "age": 18,
    "url": "https://swapi.co/api/people/1"
}
```

If our resource declared the following `attributes` list:

```python
attributes = (
    'name',
    'age',
)
```

Then the generated instance will have the following properties:

```python
person.name
>>> 'luke skywalker'
person.age
>>> 18
```

The `url` property will not be added.

Beckett will try to determine the type of the property from the JSON type. Beckett does not currently support complex type assignments.

#### SubResources

You can use [SubResources](#class-subresource) to generate simple, typed, sub-resources from properties that are dictionaries. These can be generated using the `subresources` attribute on the `BaseResource` meta class.

**Note** that you can set any type of `Resource` class as a subresource, not just `SubResource`.

## class HypermediaResource

A simple representation of a resource that supports hypermedia links and methods to related resources. Beckett will attempt to match related resources with the URL patterns it knows about it's resources, in order to discover them.

**Example:**
```python
# myresources.py
from beckett.resources import HypermediaResource

class Designer(HypermediaResource):
    class Meta(HypermediaResource.Meta):
        name = 'Designer'
        identifier = 'slug'
        attributes = (
            'slug',
            'name',
        )
        methods = (
            'get',
        )
        # Additional required attributes
        base_url = 'http://myapi.com/api'
        related_resources = ()


class Product(HypermediaResource):

    class Meta(HypermediaResource.Meta):
        name = 'Product'
        identifier = 'slug'
        attributes = (
            'slug',
            'name',
            'price',
            'discount'
        )
        methods = (
            'get',
        )
        # Additional required attributes
        base_url = 'http://myapi.com/api'
        related_resources = (
            Designer,
        )

```
**Usage:**
```bash
# Note: Data will usually come straight from the client method
>>> data = {'name': 'Tasty product', 'slug': 'sluggy', 'designer': 'http://myapi.com/api/designers/some-designer'}
>>> product = Product(**data)
>>> product.get_designers()
[<Designer | Some Designer>]
```

### Meta Attributes

HypermediaResource has two additional, required, attributes that are essential for making hypermedia work

| Attribute           | Required | Type             | Description                                                                                                     |
|:--------------------|:---------|:-----------------|:----------------------------------------------------------------------------------------------------------------|
| `base_url`          | Yes      | String           | The base url of this resource                                                                                   |
| `related_resources` | Yes      | Tuple of classes | A tuple of classes that are related to this resource, and should be expected in the JSON response from the API. |

### Customisable Methods

The HypermediaResource has methods that can be subclassed and customised:

* [HypermediaResource.get_url](/advanced/#customising-resource-urls)
* [HypermediaResource.get_http_headers](/advanced/#customise-http-headers)
* [HypermediaResource.prepare_http_request](/advanced/#modify-http-request)


## class SubResource

A basic version of `BaseResource` without any URL-generating abilities. It should be used for a plain JSON dictionary that you want to represent as a typed instance. It cannot be used with Beckett's [clients](/clients).

Consider the following JSON for a "Book" resource:

```json
{
    "author": {
        "name": "Earnest"
    },
    "title": "A Farewell to Arms"
}
```
I can use this class to transform the "author" attribute into
a typed resource:

**Example:**
```python
# my_resources.py
from beckett import resources


class AuthorSubResource(resources.SubResource):
    class Meta:
        name = 'Author'
        identifier = 'name'
        attributes = (
            'name',
        )

class BookResource(resources.Resource):
    class Meta:
        ...other stuff...
        subresources = {
            'author': AuthorSubResource
        }
```

### Meta Attributes

| Attribute       | Required | Type             | Description                                                                                                                                                                           |
|:----------------|:---------|:-----------------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `name`          | Yes      | String           | The name of this subresource instance. Usually a singular noun.                                                                                                                       |
| `resource_name` | No       | String           | The name of this subresource used in the url. Usually a plural noun. If not set, we'll attempt to make a pluralised version of the `name` attribute.                                  |
| `identifier`    | Yes      | Int/String       | The key attribute that can be used to identify this attribute. Used when referring to related resources.                                                                              |
| `attributes`    | Yes      | Tuple of Strings | A tuple list of strings, referring to the key attributes that you want to populate the resource instances with. You can use this for whitelisting and versioning changes in your API. |

SubResources can be a list of values or a single value.
