# Static files

![Static files](images/static.png)

As their name suggests, static files are the files that don't change. In your average app, this includes CSS files, JavaScript files and images. They can also include audio files and other things of that nature.

## Organizing your static files

We'll create a directory for our static files called _static_ inside our application package.

\begin{codelisting}
\label{code:}
\codecaption{Our app with the static directory}
```
myapp/
    __init__.py
    static/
    templates/
    views/
    models.py
run.py
```
\end{codelisting}


How you organize the files in _static/_ is a matter of personal preference. Personally, I get a little irked by having third-party libraries (e.g. jQuery, Bootstrap, etc.) mixed in with my own JavaScript and CSS files. To avoid this, I recommend separating third-party libraries out into a _lib/_ folder within the appropriate directory. Some projects use _vendor/_ instead of _lib/_. 

\begin{codelisting}
\label{code:}
\codecaption{An average \textit{static/} directory}
```
static/
    css/
        lib/
            bootstrap.css
        style.css
        home.css
        admin.css
    js/
        lib/
            jquery.js
        home.js
        admin.js
    img/
        logo.svg
        favicon.ico
```
\end{codelisting}



### Serving a favicon

The files in our static directory will be served from _example.com/static/_. By default, web browsers and other software expects our favicon to be at _example.com/favicon.ico_. To fix this discrepency, we can add the following in the `<head>` section of our site template.

\begin{codelisting}
\label{code:}
\codecaption{}
```jinja
<link rel="shortcut icon"
    href="{{ url_for('static', filename='img/favicon.ico') }}">
```
\end{codelisting}

## Manage static assets with Flask-Assets

Flask-Assets is an extension for managing your static files. There are two really useful tools that Flask-Assets provides. First, it lets you define **bundles** of assets in your Python code that can be inserted together in your template. Second, it lets you **pre-process** those files. This means that you can combine and minify your CSS and JavaScript files so that the user only has to load two minified files (CSS and JavaScript) without forcing you to develop a complex asset pipeline. You can even compile your files from Sass, LESS, CoffeeScript and a bunch of other sources.

\begin{codelisting}
\label{code:}
\codecaption{The \textit{static/} directory that we'll be working with in this chapter}
```
static/
    css/
        lib/
            reset.css
        common.css
        home.css
        admin.css
    js/
        lib/
            jquery-1.10.2.js
            Chart.js
        home.js
        admin.js
    img/
        logo.svg
        favicon.ico
```
\end{codelisting}


### Defining bundles

Our app has two sections: the public site and the admin panel, referred to as "home" and "admin" respectively in our app. We'll define four bundles to cover this: a JavaScript and CSS bundle for each section. We'll put these in an assets module inside our `util` package.

\begin{codelisting}
\label{code:}
\codecaption{Defining our bundles}
```python
# myapp/util/assets.py

from flask.ext.assets import Bundle, Environment
from .. import app

bundles = {

    'home_js': Bundle(
        'js/lib/jquery-1.10.2.js',
        'js/home.js',
        output='gen/home.js),

    'home_css': Bundle(
        'css/lib/reset.css',
        'css/common.css',
        'css/home.css',
        output='gen/home.css),

    'admin_js': Bundle(
        'js/lib/jquery-1.10.2.js',
        'js/lib/Chart.js',
        'js/admin.js',
        output='gen/admin.js),

    'admin_css': Bundle(
        'css/lib/reset.css',
        'css/common.css',
        'css/admin.css',
        output='gen/admin.css)
}

assets = Environment(app)

assets.register(bundles)
```
\end{codelisting}

\begin{aside}
\label{aside:}
\heading{WARNING}

Flask-Assets combines your files in the order in which they are listed here. If _admin.js_ requires _jquery-1.10.2.js_, make sure jquery is listed first.

We're defining the bundles in a dictionary to make it easy to register them. webassets, the package behind Flask-Assets lets us register bundles in a number of ways, including passing a dictionary like the one we made in this snippet.

Source: [https://github.com/miracle2k/webassets/blob/0.8/src/webassets/\-env.py#L380](https://github.com/miracle2k/webassets/blob/0.8/src/webassets/env.py#L380)
\end{aside}

Since we're registering our bundles in `util.assets`, all we have to do is import that module in _\_\_init\_\_.py_ after our app has been initialized.

\begin{codelisting}
\label{code:}
\codecaption{Import the bundles}
```python
# myapp/__init__.py

# [...] Initialize the app

from .util import assets
```
\end{codelisting}

### Using your bundles

\begin{codelisting}
\label{code:}
\codecaption{Our app's \textit{templates/} directory}
```
templates/
    home/
        layout.html
        index.html
        about.html
    admin/
        layout.html
        dash.html
        stats.html
```
\end{codelisting}


To use our admin bundles, we'll insert them into the parent template for the admin section: _admin/layout.html_.

\begin{codelisting}
\label{code:}
\codecaption{Using our admin bundle in a template}
```jinja
{# myapp/templates/admin/layout.html #}

<!DOCTYPE html>
<html lang="en">
    <head>
        {% assets "admin_js" %}
            <script type="text/javascript" src="{{ ASSET_URL }}"></script>
        {% endassets %}
        {% assets "admin_css" %}
            <link rel="stylesheet" href="{{ ASSET_URL }}" />
        {% endassets %}
    </head>
    <body>
    {% block body %}
    {% endblock %}
    </body>
</html>
```
\end{codelisting}



We can do the same thing for the home bundles in _templates/home/layout\-.html_.

### Using filters

We can use filters to pre-process our static files. This is especially handy for minifying our JavaScript and CSS bundles.

\begin{codelisting}
\label{code:}
\codecaption{Updating our code to use minification filters}
```python
# myapp/util/assets.py

# [...]

bundles = {

    'home_js': Bundle(
        'lib/jquery-1.10.2.js',
        'js/home.js',
        output='gen/home.js',
        filters='jsmin'),

    'home_css': Bundle(
        'lib/reset.css',
        'css/common.css',
        'css/home.css',
        output='gen/home.css',
        filters='cssmin'),

    'admin_js': Bundle(
        'lib/jquery-1.10.2.js',
        'lib/Chart.js',
        'js/admin.js',
        output='gen/admin.js',
        filters='jsmin'),

    'admin_css': Bundle(
        'lib/reset.css',
        'css/common.css',
        'css/admin.css',
        output='gen/admin.css',
        filters='cssmin')
}

# [...]
```
\end{codelisting}

---

\begin{aside}
\label{aside:}
\heading{A note on our minification filters}

To use the `jsmin` and `cssmin` filters, you'll need to install the `jsmin` and `cssmin` packages (e.g. with `pip install jsmin cssmin`). Make sure to add them to _requirements.txt_ too.

\end{aside}

Flask-Assets will merge and compress our files the first time the template is rendered, and it'll automatically update the compressed file when one of the source files changes.

\begin{aside}
\label{aside:}
\heading{A note on debugging with Flask-Assets}

If you set `ASSETS_DEBUG = True` in your config, Flask-Assets will output each source file individually instead of merging them.

\end{aside}

---

\begin{aside}
\label{aside:}
\heading{Related Links}

Take a look at some of these other filters that we can use with Flask-Assets: [http://elsdoerfer.name/docs/webassets/builtin_filters.html#js-css-compilers](http://elsdoerfer.name/docs/webassets/builtin_filters.html#js-css-compilers)

\end{aside}

## Summary

* Static files go in the _static/_ directory.
* Separate third-party libraries from your own static files.
* Specify the location of your favicon in your templates.
* Use Flask-Assets to insert static files in your templates.
* Flask-Assets can compile, combine and compress your static files.