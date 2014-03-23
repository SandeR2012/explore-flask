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

Sometimes you’ll need to define configuration variables that shouldn’t be shared. For this reason, you’ll want to separate them from the variables in _config.py_ and keep them out of the repository. You may be hiding secrets like database passwords and API keys, or defining variables specific to your current machine. To make this easy, Flask gives us a feature called Instance folders. The instance folder is a subdirectory sits in the repository root and contains a configuration file specifically for this instance of the application. It is not committed to version control.

Here’s a simple repository for a Flask application using an instance folder:

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

### Using instance folders

To load the configuration variables defined inside of an instance folder, you can use `app.config.from_pyfile()`. If we set `instance_relative_config=True` when we create our app with the `Flask()` call, `app.config.from_pyfile()` will check for the specified file in the _instance/_ directory.

```
app = Flask(__name__, instance_relative_config=True)
app.config.from_object('config')
app.config.from_pyfile('config.py')
```

Now, you can define variables in _instance/config.py_ just like you did in _config.py_. You should also add the instance folder to your version control system’s ignore list. To do this with git, you would save `instance/` on a new line in  _gitignore_.

### Secret keys

The private nature of the instance folder makes it a great candidate for defining keys that you don't want exposed in version control. These may include your app's secret key or third-party API keys. This is especially important if your application is open source, or might be at some point in the future.

Your _instance/config.py_ file might look something like this:

{ THIS COULD BE A GOOD CHANCE TO ENCODE BACKER NAMES! }

```
SECRET_KEY = 'ABCDEFG' # This is a bad secret key!
STRIPE_API_KEY = 'yekepirtsaton'
SQLALCHEMY_DATABASE_URI = "postgresql://username:password@localhost/databasename"
```

### Minor environment-based configuration

If the difference between your production and development environments are pretty minor, you may want to use your instance folder to handle the configuration changes. Variables defined in the instance/config.py file can override the value in config.py. You just need to make the call to `app.config.from_pyfile()` after `app.config.from_object()`.  One way to take advantage of this is to change the way your app is configured on different machines. Your development repository might look like this:

config.py
```
DEBUG = False
SQLALCHEMY_ECHO = False
```

instance/config.py
```
DEBUG = True
SQLALCHEMY_ECHO = True
```

Then in production, you would leave these lines out of _instance/config.py_ and it would fall back to the values defined in _config.py_. 

{ SEE MORE:
* Read about Flask-SQLAlchemy’s configuration keys here: http://pythonhosted.org/Flask-SQLAlchemy/config.html#configuration-keys
}

## Configuring from envvar

The instance folder shouldn’t be in version control. This means that you won’t be able to track changes to your instance configurations. That might not be a problem with one or two variables, but if you have a finely tuned configurations for various environments (production, staging, development, etc.) you don’t want to risk losing that. 

Flask gives us the ability to choose a configuration file on the fly based on the value of an environment variable. This means that we can have several configuration files in our repository (and in version control) and always load the right one, depending on the environment.

When we’re at the point of having several configuration files in the repository, it’s time to move those files into a `config` package. Here’s what that looks like in a repository:

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

In this case we have a few different configuration files:

{ PUT THIS IN A TABLE }

* _config/default.py_ : Default values, to be used for all environments or overridden by individual environments. An example might be setting `DEBUG = False` in _config/default.py_ and `DEBUG = True` in _config/development.py_.
* _config/development.py_ : Values to be used during development. Here you might specify the URI of a database sitting on localhost.
* _config/production.py_ : Values to be used in production. Here you might specify the URI for your database server, as opposed to the localhost database URI used for development.
* _config/staging.py_ : Depending on your deployment process, you may have a staging step where you test changes to your application on a server that simulates a production environment. You’ll probably use a different database, and you may want to alter other configuration values for staging applications.

To actually use these files in your various environments, you can make a call to `app.config.from_envvar()`:

yourapp/__init__.py
```
app = Flask(__name__, instance_relative_config=True)
app.config.from_object('config.default')
app.config.from_pyfile('config.py') # Don't forget our instance folder
app.config.from_envvar('APP_CONFIG_FILE')
```

`app.config.from_envvar(‘APP_CONFIG_FILE’)` will load the file specified in the environment variable `APP_CONFIG_FILE`. The value of that environment variable should be the full path of a configuration file. 


How you set this environment variable depends on the platform on which you’re running your app. If you’re running on a regular Linux server, you could set up a shell script that sets the environment variables and runs `run.py`:

start.sh
```
APP_CONFIG_FILE=/var/www/yourapp/config/production.py
python run.py
```

If you’re using Heroku, you’ll want to set the environment variables with the Heroku tools. The same idea applies to other “PaaS” platforms.

## Summary

* A simple app may only need one configuration file: _config.py_.
* Instance folders can help us hide secret configuration values.
* Instance folders can be used to alter an application’s configuration for a specific environment.
* We should use environment variables and `app.config.from_envvar()` for complicated, environment-based configurations.
