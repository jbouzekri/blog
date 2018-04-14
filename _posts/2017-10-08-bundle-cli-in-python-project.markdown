---
layout: post
title: Bundle a CLI script in your Python project
description: Different way to provide a CLI with your Python project
date: 2017-10-08
tags:
    - cli
    - python
---

## Introduction

A few days ago, I was walking through the code base of the [Flask](http://flask.pocoo.org/) Python microframework and I stumbled upon a `__main__.py` file inside the `flask` module.

After I look it up in the Python doc, I decided to write this short post about different ways of creating CLI feature inside a Python project.

This article is not focused on how to handle arguments and options in the script (things like `argparse`, `getopt` and so on) but how to provide the executable CLI script.

## First solution : a basic python script

A python script can become an executable easily by leveraging the [Shebang](https://bash.cyberciti.biz/guide/Shebang) feature under a UNIX / Linux system.

For example, create a Python script `basicpythonscript` (without the `py` extension to hide the fact that it is based on Python) :

```python
#!/usr/bin/env python

import sys

if __name__ == "__main__":
    print(sys.version)
```

Make it executable : `chmod a+x basicpythonscript`, put it in your `PATH`. You can now execute it from anywhere you want :

```shell
$ basicpythonscript
2.7.12+ (default, Sep 17 2016, 12:08:02)
[GCC 6.2.0 20160914]
```

Beware of the Python interpreter that would be used because of the `env` Shebang. Most recent distributions are switching to Python 3.

On a side note, this line `if __name__ == "__main__":` ensures that the following code block is only executed when you are in the top level execution scope (read from stdout, executed as a script or in the interactive terminal). So it won't be executed if you import this python file as a module in another part of your program.


## Second solution : python -m

This is the solution that prompt me to write this article. It is explained in the Python documentation : [__main__ — Top-level script environment](https://docs.python.org/3/library/__main__.html).

Any module or package can be executed from the python interpreter with the command `python -m <module or package name>`. The module or package needs to be in Python `sys.path` to be found and on execution, the `__name__` of the executing module will be set to `__main__`

### With a module

Create a `basicpythonmodule.py` file with this content :

```python
#!/usr/bin/env python

import sys

if __name__ == "__main__":
    print(sys.version)
```

Then :

```shell
$ python -m basicpythonmodule
2.7.12+ (default, Sep 17 2016, 12:08:02)
[GCC 6.2.0 20160914]
```

### With a package

Let's create a python package and execute it using the `-m` interpreter option:

```shell
$ mkdir basicpythonpackage
$ echo "print('__init__ executed')" > basicpythonpackage/__init__.py
$ python -m basicpythonpackage
__init__ executed
/usr/bin/python: No module named basicpythonpackage.__main__; 'basicpythonpackage' is a package and cannot be directly executed
```

We are missing something here when we use a package instead of a module. The documentation says : *For a package, the same effect can be achieved by including a __main__.py module, the contents of which will be executed when the module is run with -m.*

So that's where the `__main__.py` file comes from in the Flask project. Let's try it for our basic package.

```shell
$ cat >basicpythonpackage/__main__.py <<EOL
import sys

if __name__ == "__main__":
    print(sys.version)
EOL

$ python -m basicpythonpackage
__init__ executed
2.7.12+ (default, Sep 17 2016, 12:08:02)
[GCC 6.2.0 20160914]
```

Now it works.

## Third solution : setuptools console scripts

Setuptools provides a feature to generate script automatically. It is provided as a key `console_scripts` or `gui_scripts` in the `entry_points` section of the `setup` call in your project `setup.py`.

Each console script is defined with a string that follows this convention :

```
<name of the executable> = <fullpath to a python module>:<name of the function to execute in the module>
```

An example is worth all explanation, let's continue to work with our previous `basicpythonpackage`:

```shell
$ cat >setup.py <<EOL
from setuptools import setup, find_packages
setup(
    name="basicpythonpackage",
    version="0.1",
    packages=find_packages(),
    entry_points={
        'console_scripts': [
            'foo = basicpythonpackage.foo:my_func'
        ]
    }
)
EOL

$ cat >basicpythonpackage/foo.py <<EOL
import sys

def my_func(*args, **kwargs):
   print('args', args)
   print('kwargs', kwargs)
EOL

$ pip install .
Processing ...
Installing collected packages: basicpythonpackage
  Running setup.py install for basicpythonpackage ... done
Successfully installed basicpythonpackage-0.1

$ foo
__init__ executed
('args', ())
('kwargs', {})

$ which foo
/usr/local/bin/foo

$ cat /usr/local/bin/foo
#!/usr/bin/python
# EASY-INSTALL-ENTRY-SCRIPT: 'basicpythonpackage==0.1','console_scripts','foo'
__requires__ = 'basicpythonpackage==0.1'
import re
import sys
from pkg_resources import load_entry_point

if __name__ == '__main__':
  sys.argv[0] = re.sub(r'(-script\.pyw?|\.exe)?$', '', sys.argv[0])
  sys.exit(
      load_entry_point('basicpythonpackage==0.1', 'console_scripts', 'foo')()
  )
```

As you can see, we got a global executable command `foo`. On execution, it loads the python function you configured, executes it without any arguments and exits with a status code equal to the return of this function.

*Note : as it writes to `/user/local/bin`, the pip install command has to be executed as root or with a sudo. You can run it in a virtualenv if you want, the script will be available in the bin folder of the virtualenv.*

I invite you to read the full documentation because this feature is much more powerful and complete : [Setuptools Automatic Script Creation](http://setuptools.readthedocs.io/en/latest/setuptools.html#automatic-script-creation)

## Conclusion

We went over 3 ways to create CLI entry point in a Python program. My preference goes to the `setuptools` solution, it seems cleaner, the executable is available in the system without indicating it is a Python script and it can work with Windows too (if needed).

If you know of any other way, don't hesitate to reach out to me in order to update this article.
