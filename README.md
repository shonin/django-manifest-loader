# Django Manifest Loader

[![Build Status](https://img.shields.io/travis/shonin/django-manifest-loader/main?label=latest%20published%20branch&style=flat-square
)](https://travis-ci.org/shonin/django-manifest-loader)
[![Build Status](https://img.shields.io/travis/shonin/django-manifest-loader/dev?label=development%20branch&style=flat-square
)](https://travis-ci.org/shonin/django-manifest-loader)
[![contributions welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg?style=flat-square)](#)

_Always have access to the latest webpack assets, with minimal configuration. Wraps Django's built in 
`{% static %}` templatetag to allow you to link to assets according to a webpack manifest file. Handles webpack's 
split chunks._

**Turns this**

```djangotemplate
{% load manifest %}
<script src="{% manifest 'main.js' %}" />
```

**Into this**

```djangotemplate
<script src="/static/main.8f7705adfa281590b8dd.js" />
```

* For an in-depth look at this package, check out [this blog post here](https://medium.com/@shonin/django-and-webpack-now-work-together-seamlessly-a90cffdbab8e)
* [Quick start guide](https://medium.com/@shonin/django-and-webpack-in-4-short-steps-b39bd3380c71)

## Installation

```shell script
pip install django-manifest-loader
```

## Django Setup

```python
# settings.py

INSTALLED_APPS = [
    ...
    'manifest_loader',  # add to installed apps
    ...
]

STATICFILES_DIRS = [
    BASE_DIR / 'dist'  # the directory webpack outputs to
]
```

You must add webpack's output directory to the `STATICFILES_DIRS` list. 
The above example assumes that your webpack configuration is setup to output all files into a directory `dist/` that is 
in the `BASE_DIR` of your project.

`BASE_DIR`'s default value, as set by `$ djagno-admin startproject` is `BASE_DIR = Path(__file__).resolve().parent.parent`, in general 
you shouldn't be modifying it.

**Optional settings,** default values shown.
```python
# settings.py

MANIFEST_LOADER = {
    'output_dir': None,  # where webpack outputs to, if not set will search in STATICFILES_DIRS for the manifest. 
    'manifest_file': 'manifest.json',  # name of your manifest file
    'cache': False,  # recommended True for production, requires a server restart to pickup new values from the manifest.
}
```

## Webpack configuration

You must install the `WebpackManifestPlugin`. Optionally, but recommended, is to install the `CleanWebpackPlugin`.

```shell script
npm i --save-dev webpack-manifest-plugin clean-webpack-plugin
```

```javascript
// webpack.config.js

const { CleanWebpackPlugin } = require('clean-webpack-plugin');
const ManifestPlugin = require('webpack-manifest-plugin');

module.exports = {
  ...
  plugins: [
      new CleanWebpackPlugin(),  // removes outdated assets from the output dir
      new ManifestPlugin(),  // generates the required manifest.json file
  ],
  ...
};
```

# Usage

Django Manifest Loader comes with two template tags that house all logic. The `manifest` tag takes a single string 
input, such as `'main.js'`, looks it up against the webpack manifest, and then outputs the url to that compiled file.
It works just like Django's built it `static` tag, except it's finding the correct filename.

The `manifest_match` tag takes two arguments, a sting to pattern match filenames against, and a string to embed matched file 
urls into. See the `manifest_match` section for more information.

## Single file use (for cache busting) (`manifest` tag)

```djangotemplate
{% load manifest %}

<script src="{% manifest 'main.js' %}"></script>
```

turns into

```html
<script src="/static/main.8f7705adfa281590b8dd.js"></script>
```

Where the argument to the tag will be the original filename of a file processed by webpack. If in doubt, check your 
`manifest.json` file generated by webpack to see what files are available. 

The reason this is worth while is because of the content hash after the original filename, which will invalidate the 
browser cache every time the file is updated. This ensures that your users always have the latest assets. 

## Split chunks (`manifest_match` tag)

```djangotemplate
{% load manifest %}

{% manifest_match '*.js' '<script src="{match}"></script>' %}
```

turns into

```html
<script src="/static/vendors~main.3ad032adfa281590f2a21.js"></script>
<script src="/static/main.8f7705adfa281590b8dd.js"></script>
```

This tag takes two arguments, a pattern to match against, according to the rules of the python fnmatch package, 
and a string to input the file urls into. The second argument must contain the string `{match}`, as it is what 
is replaced with the urls. 

## URLs in Manifest File

If your manifest file points to full urls, instead of file names, the full url will be output instead of pointing 
to the static file directory in Django.

Example:

```json
{
  "main.js": "http://localhost:8080/main.js"
}
```

```djangotemplate
{% load manifest %}

<script src="{% manifest 'main.js' %}"></script>
```

Will output as:

```html
<script src="http://localhost:8080/main.js"></script>
```

## Custom Loaders

Custom loaders allow you to implement your own means of extracting data from your manifest file. If your manifest
file has a non-standard structure, this is how you can tell `django-manifest-loader` how to read it.

First import the loader parent abstract class, and subclass it in your new loader class. Your new loader must have two
static methods that each take two required arguments: `get_single_match(manifest, key)` and 
`get_multi_match(manifest, pattern)`.

```python
from manifest_loader.loaders import LoaderABC

class MyCustomLoader(LoaderABC):
    @staticmethod
    def get_single_match(manifest, key):
        pass

    @staticmethod
    def get_multi_match(manifest, pattern):
        pass
```

`get_single_match` - returns a `String`, finds a single file in your manifest file, according to the `key`

`get_multi_match` - returns a `List` of files in your manifest, according to the `pattern`

`manifest` - this is your full manifest file, after being processed by `json.load()`. It will be a dictionary or list
    depending on which it is in your manifest file. 
    
`key` - `String`; the argument passed into the `manifest` template tag. e.g.: in the template tag `{% manifest 'index.js' %}`, 
    the string `'index.js'` is sent to `get_single_match` as `key` (without surrounding quotes)
    
`pattern` - `String`; the first argument passed into the `manifest_match` template tag. e.g.: in the template tag 
    `{% manifest_match '*.js' '<script src="{match}"></script>' %}`, the string `'*.js'` is sent to `get_multi_match` 
    as `pattern` (without surrounding quotes)
    
**Below is the code for the default loader, which is a good starting point:**

```python
import fnmatch
from manifest_loader.loaders import LoaderABC

class DefaultLoader(LoaderABC):
    @staticmethod
    def get_single_match(manifest, key):
        return manifest.get(key, key)

    @staticmethod
    def get_multi_match(manifest, pattern):
        matched_files = [file for file in manifest.keys() if
                         fnmatch.fnmatch(file, pattern)]
        return [manifest.get(file) for file in matched_files]
``` 

In the above example, `get_single_match` retrieves the value on the `manifest` dictionary that matches the key `key`. If
the key does not exist on the dictionary, it instead returns the key.

`get_multi_match` uses the recommended `fnmatch` python standard library to do pattern matching. You could also use 
regex in it's place. Here, it iterates through all the keys in the manifest file, and builds a list of the keys that 
match the given `pattern`. It then returns a list of the values associated with those matched keys. 

### Activating the custom loader 

To put the custom loader into use it needs to be registered in your `settings.py`.

```python
# settings.py
from my_app.utils import MyCustomLoader

MANIFEST_LOADER = {
    ...
    'loader': MyCustomLoader
}
```

# About

At it's heart Django Manifest Loader is an extension to Django's built-in `static` templatetag. 
When you use the provided `{% manifest %}` templatetag, all the manifest loader is doing is 
taking the input string, looking it up against the manifest file, modifying the value, and then
passing along the result to the `{% static %}` template tag. The `{% manifest_match %}` tag works
similarly, just with a bit of additional logic to find all the necessary files and to render the output.

### Suggested Project Structure

```
BASE_DIR
├── dist
│   ├── main.f82c02a005f7f383003c.js
│   └── manifest.json
├── frontend
│   ├── apps.py
│   ├── src
│   │   └── index.js
│   ├── templates
│   │   └── frontend
│   │       └── index.html
│   └── views.py
├── manage.py
├── package.json
├── project
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── requirements.txt
└── webpack.config.js
```

### Cache Busting and Split Chunks (the problem this package solves)

* [What is cache busting?](https://www.keycdn.com/support/what-is-cache-busting)
* [The 100% correct way to split your chunks with Webpack](https://medium.com/hackernoon/the-100-correct-way-to-split-your-chunks-with-webpack-f8a9df5b7758)

### Tests and Code Coverage

Run unit tests and verify 100% code coverage with:

```
git clone https://github.com/shonin/django-manifest-loader.git
cd django-manifest-loader
pip install -e .

# run tests
python runtests.py

# check code coverage
pip install coverage
coverage run --source=manifest_loader/ runtests.py
coverage report
```

### Contributing

Do it. Please feel free to file an issue or open a pull request. The code of conduct is basic human kindness.

### License 

Django Manifest Loader is distributed under the [3-clause BSD license](https://opensource.org/licenses/BSD-3-Clause). 
This is an open source license granting broad permissions to modify and redistribute the software.

