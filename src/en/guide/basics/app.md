# Sanic Application

## Instance

---:1
The most basic building block is the `Sanic()` instance. It is not required, but the custom is to instantiate this in a file called `server.py`.
:--:1
```python
# /path/to/server.py

from sanic import Sanic

app = Sanic("MyHelloWorldApp")
```
:---

## Application context

Most applications will have the need to share/reuse data or objects across different parts of the code base. The most common example is DB connections. 

---:1
In versions of Sanic prior to v21.3, this was commonly done by attaching an attribute to the application instance
:--:1
```python
# Raises a warning as deprecated feature in 21.3
app = Sanic("MyApp")
app.db = Database()
```
:---

---:1
Because this can create potential problems with name conflicts, and to be consistent with [request context](./request.md#context) objects, v21.3 introduces application level context object.
:--:1
```python
# Correct way to attach objects to the application
app = Sanic("MyApp")
app.ctx.db = Database()
```
:---

## App Registry

---:1

When you instantiate a Sanic instance, that can be retrieved at a later time from the Sanic app registry. This can be useful, for example, if you need to access your Sanic instance from a location where it is not otherwise accessible.
:--:1
```python
# ./path/to/server.py
from sanic import Sanic

app = Sanic("my_awesome_server")

# ./path/to/somewhere_else.py
from sanic import Sanic

app = Sanic.get_app("my_awesome_server")
```
:---

---:1

If you call `Sanic.get_app("non-existing")` on an app that does not exist, it will raise `SanicException` by default. You can, instead, force the method to return a new instance of Sanic with that name.
:--:1
```python
app = Sanic.get_app(
    "non-existing",
    force_create=True,
)
```
:---

---:1
If there is **only one** Sanic instance registered, then calling `Sanic.get_app()` with no arguments will return that instance
:--:1
```python
Sanic("My only app")

app = Sanic.get_app()
```
:---

## Configuration

---:1
Sanic holds the configuration in the `config` attribute of the `Sanic` instance. Configuration can be modified **either** using dot-notation **OR** like a dictionary.
:--:1
```python
app = Sanic('myapp')

app.config.DB_NAME = 'appdb'
app.config['DB_USER'] = 'appuser'

db_settings = {
    'DB_HOST': 'localhost',
    'DB_NAME': 'appdb',
    'DB_USER': 'appuser'
}
app.config.update(db_settings)
```
:---

::: tip Heads up
Config keys _should_ be uppercase. But, this is mainly by convention, and lowercase will work most of the time.
```
app.config.GOOD = "yay!"
app.config.bad = "boo"
```
:::

There is much [more detail about configuration](/guide/deployment/configuration.md) later on.


## Customization

The Sanic application instance can be customized for your application needs in a variety of ways at instantiation. 

### Custom configuration
---:1

This simplest form of custom configuration would be to pass your own object directly into that Sanic application instance

If you create a custom configuration object, it is *highly* recommended that you subclass the Sanic `Config` option to inherit its behavior. You could use this option for adding properties, or your own set of custom logic.

*Added in v21.6*
:--:1
```python
from sanic.config import Config

class MyConfig(Config):
    FOO = "bar"

app = Sanic(..., config=MyConfig())
```
:---

---:1
A useful example of this feature would be if you wanted to use a config file in a form that differs from what is [supported](../deployment/configuration.md#using-sanic-update-config).
:--:1
```python
from sanic import Sanic, text
from sanic.config import Config

class TomlConfig(Config):
    def __init__(self, *args, path: str, **kwargs):
        super().__init__(*args, **kwargs)

        with open(path, "r") as f:
            self.apply(toml.load(f))

    def apply(self, config):
        self.update(self._to_uppercase(config))

    def _to_uppercase(self, obj: Dict[str, Any]) -> Dict[str, Any]:
        retval: Dict[str, Any] = {}
        for key, value in obj.items():
            upper_key = key.upper()
            if isinstance(value, list):
                retval[upper_key] = [
                    self._to_uppercase(item) for item in value
                ]
            elif isinstance(value, dict):
                retval[upper_key] = self._to_uppercase(value)
            else:
                retval[upper_key] = value
        return retval

toml_config = TomlConfig(path="/path/to/config.toml")
app = Sanic(toml_config.APP_NAME, config=toml_config)
```
:---
### Custom context
---:1
By default, the application context is a [`SimpleNamespace()`](https://docs.python.org/3/library/types.html#types.SimpleNamespace) that allows you to set any properties you want on it. However, you also have the option of passing any object whatsoever instead.

*Added in v21.6*
:--:1
```python
app = Sanic(..., ctx=1)
```

```python
app = Sanic(..., ctx={})
```

```python
class MyContext:
    ...

app = Sanic(..., ctx=MyContext())
```
:---
### Custom requests
---:1
It is sometimes helpful to have your own `Request` class, and tell Sanic to use that instead of the default. One example is if you wanted to modify the default `request.id` generator.

::: tip Important

It is important to remember that you are passing the *class* not an instance of the class.

:::
:--:1
```python
import time

from sanic import Request, Sanic, text


class NanoSecondRequest(Request):
    @classmethod
    def generate_id(*_):
        return time.time_ns()


app = Sanic(..., request_class=NanoSecondRequest)


@app.get("/")
async def handler(request):
    return text(str(request.id))
```
:---

### Custom error handler

---:1
See [exception handling](../best-practices/exceptions.md#custom-error-handling) for more
:--:1
```python
from sanic.handlers import ErrorHandler

class CustomErrorHandler(ErrorHandler):
    def default(self, request, exception):
        ''' handles errors that have no error handlers assigned '''
        # You custom error handling logic...
        return super().default(request, exception)

app = Sanic(..., error_handler=CustomErrorHandler())
```
:---

### Custom dumps function

---:1
It may sometimes be necessary or desirable to provide a custom function that serializes an object to JSON data.
:--:1
```python
import ujson

dumps = partial(ujson.dumps, escape_forward_slashes=False)
app = Sanic(__name__, dumps=dumps)
```
:---

---:1
Or, perhaps use another library or create your own.
:--:1
```python
from orjson import dumps

app = Sanic(__name__, dumps=dumps)
```
:---

### Custom loads function

---:1
Similar to `dumps`, you can also provide a custom function for deserializing data.

*Added in v22.9*
:--:1
```python
from orjson import loads

app = Sanic(__name__, loads=loads)
```
:---
