![Configuration](images/5.png)

# Configuration
\label{cha:configuration}

When you're learning Flask, configuration seems simple. You just define some variables in _config.py_ and everything works. That simplicity starts to fade away when you have to manage configuration for a production application. You may need to protect secret API keys or use different configurations for different environments (e.g. development and production environments). In this chapter we'll go voer some advanced Flask features that makes this managing configuration easier.

## The simple case

A simple application may not need any of these complicated features. You may just need to put _config.py_ in the root of your repository and load it in _app.py_ or _yourapp/\_\_init\_\_.py_

 The _config.py_ file should contain one variable assignment per line. When your app is initialized, the variablse in _config.py_ are configure Flask and it's extensions and are accessible via the `app.config` dictionary -- e.g. `app.config[“DEBUG”]`.

\begin{codelisting}
\label{code:config}
\codecaption{A typical \textit{config.py} file for a small project}
```python
DEBUG = True # Turns on debugging features in Flask
BCRYPT_LEVEL = 12 # Configuration for the Flask-Bcrypt extension
MAIL_FROM_EMAIL = "robert@example.com" # For use in application emails
```
\end{codelisting}

Configuration variables can be used by Flask, extensions or you. In this example, we could use `app.config[“MAIL_FROM_EMAIL”]` whenever we needed the default “from” address for a transactional email  -- e.g. password resets. Putting that information in a configuration variable makes it easy to change it in the future.

\begin{codelisting}
\label{code:load_config}
\codecaption{An example of how \textit{config.py} is loaded}
```python
# app.py or app/__init__.py
from flask import Flask

app = Flask(__name__)
app.config.from_object('config')

# Now we can access the configuration variables via app.config["VAR_NAME"].
```
\end{codelisting}

\begin{table}
\caption{Some common variables found in Flask applications \label{table:variables}}
\begin{tabular}{|l|l|l|}

  \hline
  Variable & Description & Default \\
  \hline
  DEBUG & Gives you some handy tools for debugging errors. This includes a web-based stack trace and interactive Python console for errors. & Should be set to \texttt{True} in development and \texttt{False} in production. \\
  SECRET\_KEY & This is a secret key that is used by Flask to sign cookies. It's also used by extensions like Flask-Bcrypt. You should define this in your instance folder to keep it out of version control. You can read more about instance folders in the next section. & This should be a complex random value. \\
  BCRYPT\_LEVEL & If you’re using Flask-Bcrypt to hash user passwords, you’ll need to specify the number of “rounds” that the algorithm executes in hashing a password. If you aren't using Flask-Bcrypt, you should probably start. The more rounds used to hash a password, the longer it'll take for an attacker to guess a password given the hash. The number of rounds should increase over time as computing power increases. & Section~\ref{sec:passwords} covers some of the best practices for using Bcrypt in your Flask application. \\
  \hline
\end{tabular}
\end{table}

\begin{aside}
\label{aside:debug_warning}
\heading{WARNING}

Make sure \texttt{DEBUG} is set to \texttt{False} in production. Leaving it on will allow users to run arbitrary Python code on your server.
\end{aside}

## Instance folder

Sometimes you’ll need to define configuration variables that contain sensitive information. We’ll want to separate these variables from those in _config.py_ and keep them out of the repository. You may be hiding secrets like database passwords and API keys, or defining variables specific to a given machine. To make this easy, Flask gives us a feature called **instance folders**. The instance folder is a sub-directory of the repository root and contains a configuration file specifically for this instance of the application. We don't want to commit it into version control.

\begin{codelisting}
\label{code:instance_listing}
\codecaption{An example of a repository with an instance folder}
```
config.py
requirements.txt
run.py
instance/
  config.py
yourapp/
  __init__.py
  models.py
  views.py
  templates/
  static/
```
\end{codelisting}

### Using instance folders

To load configuration variables from an instance folder, we use `app.config.from_pyfile()`. If we set `instance_relative_config=True` when we create our app with the `Flask()` call, `app.config.from_pyfile()` will load the specified file from the _instance/_ directory.

\begin{codelisting}
\label{code:instance_relative}
\codecaption{Loading configuration variables from an instance folder}
```python
# app.py or app/__init__.py
app = Flask(__name__, instance_relative_config=True)
app.config.from_object('config')
app.config.from_pyfile('config.py')
```
\end{codelisting}

Now, we can define variables in _instance/config.py_ just like you did in _config.py_. You should also add the instance folder to your version control system’s ignore list. To do this with Git, you would add `instance/` on a new line in  _.gitignore_.

### Secret keys

The private nature of the instance folder makes it a great candidate for defining keys that you don't want exposed in version control. These may include your app's secret key or third-party API keys. This is especially important if your application is open source, or might be at some point in the future. We usually want other uses and contributors to use their own keys.

{ THIS COULD BE A GOOD CHANCE TO ENCODE BACKER NAMES! }
{ IDEAL SECRET KEY? }

\begin{codelisting}
\label{code:instance_eg}
\codecaption{An example of \textit{instance/config.py} with some secret variables}
```python
# instance/config.py
SECRET_KEY = 'ABCDEFG' # This is a terrible secret key!
STRIPE_API_KEY = 'yekepirtsaton'
SQLALCHEMY_DATABASE_URI="postgresql://username:password@localhost/databasename"
```
\end{codelisting}

### Minor environment-based configuration

If the difference between your production and development environments are pretty minor, you may want to use your instance folder to handle the configuration changes. Variables defined in the _instance/config.py_ file can override the value in _config.py_. You just need to make the call to `app.config.from_pyfile()` after `app.config.from_object()`.  One way to take advantage of this is to change the way your app is configured on different machines.

\begin{codelisting}
\label{code:instance_env}
\codecaption{Using an instance folder to do override your default configuration}
```python
# config.py
DEBUG = False
SQLALCHEMY_ECHO = False


# instance/config.py
DEBUG = True
SQLALCHEMY_ECHO = True
```

\end{codelisting}

In production, we would leave the variables in Listing~\ref{code:instance_env} out of _instance/config.py_ and it would fall back to the values defined in _config.py_. 

\begin{aside}
\label{aside:instance_links}
\heading{Related Links}

- Read about Flask-SQLAlchemy’s configuration keys here: [http://pythonhosted.org/Flask-SQLAlchemy/config.html#configuration-keys](http://pythonhosted.org/Flask-SQLAlchemy/config.html#configuration-keys)

\end{aside}

## Configuring based on environment variables

The instance folder shouldn’t be in version control. This means that you won’t be able to track changes to your instance configurations. That might not be a problem with one or two variables, but if you have finely tuned configurations for various environments (production, staging, development, etc.) you don’t want to risk losing that. 

Flask gives us the ability to choose a configuration file on load based on the value of an environment variable. This means that we can have several configuration files in our repository and always load the right one. Once we have several configuration files, we can move them to their own `config` directory.

\begin{codelisting}
\label{code:config_package}
\codecaption{A repository with a directory for configuration files}
```
requirements.txt
run.py
config/
  __init__.py # Empty, just here to tell Python that it's a package.
  default.py
  production.py
  development.py
  staging.py
instance/
  config.py
yourapp/
  __init__.py
  models.py
  views.py
  static/
  templates/
```
\end{codelisting}

In Listing~\ref{code:config_package} we have a few different configuration files.


\begin{table}
\caption{Some example configuration files \label{table:config_files}}

\begin{tabular}{|l|l|}

  \hline
  \textit{config/default.py} & Default values, to be used for all environments or overridden by individual environments. An example might be setting `DEBUG = False` in _config/default.py_ and `DEBUG = True` in _config/development.py_. \\
  \textit{config/development.py} & Values to be used during development. Here you might specify the URI of a database sitting on localhost. \\
  \textit{config/production.py} & Values to be used in production. Here you might specify the URI for your database server, as opposed to the localhost database URI used for development. \\
  \textit{config/staging.py} & Depending on your deployment process, you may have a staging step where you test changes to your application on a server that simulates a production environment. You’ll probably use a different database, and you may want to alter other configuration values for staging applications. \\
  \hline

\end{tabular}
\end{table}

To decide which configuration file to load, we'll call `app.config.from_envvar()`.

\begin{codelisting}
\label{code:load_envvar}
\codecaption{Loading a configuration file based on an environment variable}
```python
# yourapp/__init__.py

app = Flask(__name__, instance_relative_config=True)

# Load the default configuration
app.config.from_object('config.default')

# Load the configuration from the instance folder
app.config.from_pyfile('config.py')

# Load the file specified by the APP_CONFIG_FILE environment variable
# Variables defined here will override those in the default configuration
app.config.from_envvar('APP_CONFIG_FILE')
```
\end{codelisting}

The value of the environment variable should be the absolute path to a configuration file. 

How we set this environment variable depends on the platform in which we’re running the app. If we’re running on a regular Linux server, we can set up a shell script that sets our environment variables and runs _run.py_.

\begin{codelisting}
\label{code:start_sh}
\codecaption{A script that can be modified for each environment}
```bash
# start.sh

APP_CONFIG_FILE=/var/www/yourapp/config/production.py
python run.py
```
\end{codelisting}

_start.sh_ is unique to each environment, so it should be left out of version control. On Heroku, we'll want to set the environment variables with the Heroku tools. The same idea applies to other “PaaS” platforms.

## Summary

* A simple app may only need one configuration file: _config.py_.
* Instance folders can help us hide secret configuration values.
* Instance folders can be used to alter an application’s configuration for a specific environment.
* We should use environment variables and `app.config.from_envvar()` for more complicated environment-based configurations.
