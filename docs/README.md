
## Django Manifest Loader 

[![Build Status](https://img.shields.io/travis/shonin/django-manifest-loader/main?label=latest%20published%20branch&style=flat-square
)](https://travis-ci.org/shonin/django-manifest-loader)
[![Build Status](https://img.shields.io/travis/shonin/django-manifest-loader/dev?label=development%20branch&style=flat-square
)](https://travis-ci.org/shonin/django-manifest-loader)
[![contributions welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg?style=flat-square)](#)


_Always have access to the latest webpack assets, with minimal configuration. Wraps Django's built in 
`{% static %}` templatetag to allow you to link to assets according to a webpack manifest file. Handles webpack's 
split chunks._

## About

At its heart Django Manifest Loader is an extension to Django's built-in `static` template tag. 
When you use the provided `{% manifest %}` template tag, all the manifest loader is doing is 
taking the input string, looking it up against the manifest file, modifying the value, and then
passing along the result to the `{% static %}` template tag. The `{% manifest_match %}` tag works
similarly, just with a bit of additional logic to find all the necessary files and to render the output.


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
The above example assumes that your webpack configuration is set up to output all files into a directory `dist/` that is 
in the `BASE_DIR` of your project.

`BASE_DIR`'s default value, as set by `$ djagno-admin startproject` is `BASE_DIR = Path(__file__).resolve().parent.parent`, in general 
you shouldn't be modifying it.

**Optional settings,** default values shown.
```python
# settings.py

MANIFEST_LOADER = {
    'output_dir': None,  # where webpack outputs to, if not set, will search in STATICFILES_DIRS for the manifest. 
    'manifest_file': 'manifest.json',  # name of your manifest file
    'cache': False,  # recommended True for production, requires a server restart to pick up new values from the manifest.
}
```

## Reference

### Manifest Tag
Returns the manifest tag

```python
@register.tag('manifest')
def do_manifest(parser, token): 

    return ManifestNode(token)

```
### Manifest Match Tag
Returns manifest_match tag

```python
@register.tag('manifest_match')
def do_manifest_match(parser, token):
    return ManifestMatchNode(token)

```
### ManifestNode
Initializes and renders the creation of the manifest tag and 


```python
 class ManifestNode(template.Node):
    """ Initalizes the creation of the manifest template tag"""
    def __init__(self, token):
        bits = token.split_contents()
        if len(bits) < 2:
            raise template.TemplateSyntaxError(
                "'%s' takes one argument (name of file)" % bits[0])
        self.bits = bits


    def render(self, context):
        """Renders the creation of the manifest tag"""
        manifest_key = get_value(self.bits[1], context)
        manifest = get_manifest()
        manifest_value = manifest.get(manifest_key, manifest_key)
        return make_url(manifest_value, context)
```
### ManifestMatch Node
Initalizes and renders the creation of the manifest match tag 

```python
class ManifestMatchNode(template.Node):
    """ Initalizes the creation of the manifest match template tag"""
    def __init__(self, token):
        self.bits = token.split_contents()
        if len(self.bits) < 3:
            raise template.TemplateSyntaxError(
                "'%s' takes two arguments (pattern to match and string to "
                "insert into)" % self.bits[0]
            )

    def render(self, context):
        """ Renders the manifest match tag"""
        urls = []
        search_string = get_value(self.bits[1], context)
        output_tag = get_value(self.bits[2], context)

        manifest = get_manifest()

        matched_files = [file for file in manifest.keys() if
                         fnmatch.fnmatch(file, search_string)]
        mapped_files = [manifest.get(file) for file in matched_files]

        for file in mapped_files:
            url = make_url(file, context)
            urls.append(url)
        output_tags = [output_tag.format(match=file) for file in urls]
        return '\n'.join(output_tags)


def get_manifest():
    """ Returns the manifest file from the output directory """
    cached_manifest = cache.get('webpack_manifest')
    if APP_SETTINGS['cache'] and cached_manifest:
        return cached_manifest

    if APP_SETTINGS['output_dir']:
        manifest_path = os.path.join(APP_SETTINGS['output_dir'],
                                     APP_SETTINGS['manifest_file'])
    else:
        manifest_path = find_manifest_path()

    try:
        with open(manifest_path) as manifest_file:
            data = json.load(manifest_file)
    except FileNotFoundError:
        raise WebpackManifestNotFound(manifest_path)

    if APP_SETTINGS['cache']:
        cache.set('webpack_manifest', data)

    return data
```


### Finding the Manifest File
Returns manifest_file
```python
def find_manifest_path():
    static_dirs = settings.STATICFILES_DIRS
    if len(static_dirs) == 1:
        return os.path.join(static_dirs[0], APP_SETTINGS['manifest_file'])
    for static_dir in static_dirs:
        manifest_path = os.path.join(static_dir, APP_SETTINGS['manifest_file'])
        if os.path.isfile(manifest_path):
            return manifest_path
    raise WebpackManifestNotFound('settings.STATICFILES_DIRS')

```
### String Validator 
Method validates if it's a string

```python

def is_quoted_string(string):
    if len(string) < 2:
        return False
    return string[0] == string[-1] and string[0] in ('"', "'")
```

### Value Validator 
Method validates the value 

```python

def get_value(string, context):
    
    if is_quoted_string(string):
        return string[1:-1]
    return context.get(string, '')
```


### URL Validator 
Function validates if it's a URL 

```python

def is_url(potential_url):
 
   
    validate = URLValidator()
    try:
        validate(potential_url)
        return True
    except ValidationError:
        return False

```

### URL Generator 
Returns the URL that will be outputed to the static file directory

```python
def make_url(manifest_value, context):


    if is_url(manifest_value):
        url = manifest_value
    else:
        url = StaticNode.handle_simple(manifest_value)
    if context.autoescape:
        url = conditional_escape(url)
    return url


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


## Usage

Django Manifest Loader comes with two template tags that house all logic. The `manifest` tag takes a single string 
input, such as `'main.js'`, looks it up against the webpack manifest, and then outputs the URL to that compiled file.
It works just like Django's built it `static` tag, except it's finding the correct filename.

The `manifest_match` tag takes two arguments, a string to pattern match filenames against and a string to embed matched file urls into. See the `manifest_match` section for more information.

### Single file use (for cache busting) (`manifest` tag)

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

This is worthwhile because of the content hash after the original filename, which will invalidate the browser cache every time the file is updated,which will ensure that your users always have the latest assets. 

### Split chunks (`manifest_match` tag)

```djangotemplate
{% load manifest %}

{% manifest_match '*.js' '<script src="{match}"></script>' %}
```

turns into

```html
<script src="/static/vendors~main.3ad032adfa281590f2a21.js"></script>
<script src="/static/main.8f7705adfa281590b8dd.js"></script>
```

This tag takes two arguments, a pattern to match against, according to the python fnmatch package rules, 
and a string to input the file URLs into. The second argument must contain the string `{match}`, as it is replaced with the URLs. 

## URLs in Manifest File

If your manifest file points to full URLs, instead of file names, the full URL will be output instead of pointing to the static file directory in Django.

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

# Documentation

Documentation was developed using [Sphinx](https://www.sphinx-doc.org/en/master/usage/configuration.html).


## Installation
In order to install sphinx

```shell script
pip install -U sphinx 
```

## Dependencies for installation
To use .md with Sphynx, it requires Recommonmark. 


```shell script
pip install recommonmark
```
## How to run
After installation of sphinx and recommonmark, to generate the '_build' directory that has doc trees and html
you would run the 'make html' command.


# Contributing

Do it. Please feel free to file an issue or open a pull request. The code of conduct is basic human kindness.

# License 

Django Manifest Loader is distributed under the [3-clause BSD license](https://opensource.org/licenses/BSD-3-Clause). 
This is an open source license granting broad permissions to modify and redistribute the software.

