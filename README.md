# Installing Django on 1&1 (shared hosting)

## Introduction
Installing Django with limited privileges, such as those found commonly in shared hosting situations, can be challenging. Some problems include:

- No root access
- Available Apache does not support WSGI.
- Available Python does not supply virtual environments or pip.

In order to use this method, the following requirements must be available:

- SSH access
- Available Apache supports FCGI
- Ability to compile from source
- .htaccess support
- tar support

Django post version 1.8 does not support FastCGI anymore, preferring WSGI, but using the python package flup we can make use of FastCGI again.
 
This Git and these instructions have been made using 1&1 as a shared hosting service, but it can probably be used with other shared hosting services if the requirements are satisfied. The 1&1 servers use Debian.

## build-python.sh usage

```
Usage: build-python.sh [-b <build_dir>] [-i <install_dir>] [-S] [-Q] [-h]
    -h                  : print out usage help
    -i <install_dir>    : installation directory, default ./install
    -b <build_dir>      : build directory, default ./build
    -S                  : skip building SSL (if already done)
    -Q                  : skip building SQLITE3 (if already done)
```

## Installation
This git provides tools to help simplify the installation process. If it doesn't work for you (for example you don't have access to wget), you can try the manual installation at the bottom of this page.

### 1. Preparation

Access your server via SSH, which will most likely take you to your base directory (your htdocs directory where the directories to your websites are).

Make a directory where you want your Python files to reside and cd into it.

```
mkdir python_files

cd python_files
```

git clone this repository and move everything into the python_files directory.

```
git clone https://github.com/sparagus/django-shared-hosting-1and1.git

mv django-shared-hosting-1and1/* ./
```

### 2. Compile and install Python

Run the build-python.sh

```
./build-python.sh
```

This will download and compile the sources of Python 3.6.3, SQLite3 (3240000), and OpenSSL 1.1.0e in a "build" directory, and install said programs in an "install" directory.

### 3. Test Python installation

After this is finished, create a virtual environment in the python_files directory.

```
install/python3.6.3/bin/python3 -m venv venv

source venv/bin/activate
```

Check that the correct python and pip are being targeted.

```
which python

which pip
```

Check that SQLite3 is accessible by Python. Enter Python interpreter

```
python
```

In Python interpreter run

```
import sqlite3
```

If no error is thrown, Python can access SQLite3. Exit the Python interpreter.

```
exit()
```

### 4. Install Django

Now  upgrade pip and install the modules in requirements.txt. Newer versions of these modules might work, but these are the newest versions currently available and they work. Future versions might not work with this method, hence I put the specific versions that currently work into the requirements.txt file.

```
pip install --upgrade pip

pip install -r requirements.txt
```

If you get an SSL error here then there's a problem with OpenSSL.

Check if Django is installed correctly.

```
python -m django --version
```

### 5. Make a project for testing the installation

Go back to your base directory and start a Django project.

```
cd ..

django-admin startproject your_site
```

We're going to make a small app to test if everything works.

```
cd your_site

python manage.py startapp apache_test

nano apache_test/views.py
```

In views.py, write

```python
from django.shortcuts import render
from django.http import HttpResponse

def home(request):
    return HttpResponse('<h1>Congratulations, Django works.</h1>')
```

Save and close. Now run

```
touch apache_test/urls.py

nano apache_test/urls.py
```

Write this to the file.

```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.home, name='apache-test'),
]
```

Save and close. Edit urls.py in your_site (the subdirectory, so your_site/your_site)

```
nano your_site/urls.py
```

Write this to the file.

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('apache-test/', include('apache_test.urls')),
]
```

Save and close. Edit settings.py. Under BASE_DIR add basepath variable. /path/to/your/htdocs looks like this in 1und1: /kunden/homepages/(...)/(...)/htdocs. Where (...) is a strings of numbers and/or letters. To find the path to your current location, you can use the pwd command and note it down.

```
pwd

nano your_site/settings.py
```

Make the edit in settings.py:

```python
...

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
basepath = '/path/to/your/htdocs'

...
```

Also in settings.py, change ALLOWED_HOSTS to contain the domain you'll be connecting to the "your_site" directory. You can of course use subdomains, etc., too, like your_sub.your_domain.com

```python
...

ALLOWED_HOSTS = ['your_domain.com']

...
```

At the very bottom of the file, under STATIC_URL, add STATIC_ROOT.

```
...

STATIC_URL = '/static/'
STATIC_ROOT = basepath + '/your_site/static'
```

### 6. Add .htaccess and FastCGI files

Save and close. Create cgi-bin directory and either copy the application.fcgi from the cloned git to /path/to/your/htdocs/your_site or create application.fcgi.

```
mkdir cgi-bin

touch cgi-bin/application.fcgi
```

**Make sure this file has propper permissions to be executed, otherwise it won't be able to be run by the FastCGI protocol and nothing will work when the domain is called!**

```
chmod 705 cgi-bin/application.fcgi

ls -l cgi-bin
```

In the reply check that application.fcgi has read and execution permissions by everyone (it should be -rwx—r-x).

Now edit this file.

```
nano cgi-bin/application.fcgi
```

Write this to the file. Things you need to change are on lines 1 (/path/to/your/htdocs), 7 (/path/to/your/htdocs), 12 (your_site), and 20 (your_site). If you installed the virtual environment containing the python files somewhere else, you have to change line 1 and line 9 accordingly. Note that the first line is not a comment, so don't remove it!

```python
#!/path/to/your/htdocs/python_files/venv/bin/python

import os
import sys
import traceback

home = '/path/to/your/htdocs'
try:
    os.environ['VIRTUAL_ENV'] = os.path.join(home, 'python_files/venv/bin')
    os.environ['PATH'] = os.environ['VIRTUAL_ENV'] + ':' + os.environ['PATH']

    project = os.path.join(home, 'your_site')
    # Add a custom Python path.
    sys.path.insert(0, project)

    # Switch to the directory of your project.
    os.chdir(project)

    # Set the DJANGO_SETTINGS_MODULE environment variable.
    os.environ['DJANGO_SETTINGS_MODULE'] = "your_site.settings"

    from django_fastcgi.servers.fastcgi import runfastcgi
    from django.core.servers.basehttp import get_internal_wsgi_application

    wsgi_application = get_internal_wsgi_application()
    runfastcgi(wsgi_application, method="prefork", daemonize="false", minspare=1, maxspare=1, maxchildren=1)
except:
    with open(os.path.join(home, 'tmp/error.log'), 'w') as fp:
        traceback.print_exc(file = fp)
```

Save and close. Make and edit .htaccess, or copy it from the git.

```
touch .htaccess

nano .htaccess
```

Write this to .htaccess. It directs all incoming traffic to the application.fcgi file.

```
AddHandler cgi-script .fcgi
RewriteEngine on
RewriteBase /
# The following two lines are for FastCGI:
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule ^(.*)$ cgi-bin/application.fcgi/$1 [QSA,L]
```

Save and close. 

### 7. Prepare Django and check if it works

Collect the static files.

```
python manage.py collectstatic
```

Connect your domain to the folder and you should get Django's debug 404 error. Navigate to your_domain.com/apache-test and you should see the message we wrote earlier (Congratulations, Django works.).

If the fcgi is working but there's another problem, python's traceback will be written to /path/to/your/htdocs/tmp/error.log. If there's a problem but no tmp/error.log, your fcgi most likely isn't being executed. Make sure the applications.fcgi has propper permissions (also the cgi-bin folder has propper permissions) and the .htaccess file is working.

Remember to make the appropriate changes when going from a development to a production environment.


## Manual compiling and installing

Do step 1 of the normal installation above.

### Compile and install OpenSSL

Make the build and install directories. Make a variable installpath which contains the full server path to install.

```
mkdir build

mkdir install

installpath=$(realpath install)
```

Get OpenSSL 1.1.0e. If you can't wget, find another way to download the file and get it to your server.

```
cd build

wget https://www.openssl.org/source/old/1.1.0/openssl-1.1.0e.tar.gz
```

Unpack, config, make, install

```
tar -zxf openssl-1.1.0e.tar.gz

cd openssl-1.1.0e

./config --prefix=$installpath/openssl-1.1.0e

make

make install

cd ..
```

### Compile and install SQLite3

If we compile Python, SQLite3 isn't added by default, so we need to compile it ourselves.

```
wget https://www.sqlite.org/2018/sqlite-autoconf-3240000.tar.gz

tar -zxvf sqlite-autoconf-3240000.tar.gz

cd sqlite-autoconf-3240000

./configure --prefix=$installpath/python3.6.3

make

make install

cd ..
```

### Compile Python

Create this file and edit it, or copy it from the git to here (the build directory).

```
touch setup.py.patch

nano setup.py.patch
```

Write this to the file.

```
--- setup.py    2017-10-03 01:52:02.000000000 -0400
+++ /patched/setup.py   2018-01-22 14:23:23.071041205 -0500
@@ -811,9 +811,9 @@
         exts.append( Extension('_socket', ['socketmodule.c'],
                                depends = ['socketmodule.h']) )
         # Detect SSL support for the socket module (via _ssl)
+        SSL = os.environ['SSL_DIR']
         search_for_ssl_incs_in = [
-                              '/usr/local/ssl/include',
-                              '/usr/contrib/ssl/include/'
+                os.path.join(SSL, 'include'),
                              ]
         ssl_incs = find_file('openssl/ssl.h', inc_dirs,
                              search_for_ssl_incs_in
@@ -824,16 +824,16 @@
             if krb5_h:
                 ssl_incs += krb5_h
         ssl_libs = find_library_file(self.compiler, 'ssl',lib_dirs,
-                                     ['/usr/local/ssl/lib',
-                                      '/usr/contrib/ssl/lib/'
+                                     [ os.path.join(SSL, 'lib')
                                      ] )

         if (ssl_incs is not None and
             ssl_libs is not None):
             exts.append( Extension('_ssl', ['_ssl.c'],
-                                   include_dirs = ssl_incs,
-                                   library_dirs = ssl_libs,
-                                   libraries = ['ssl', 'crypto'],
+                                   library_dirs = [],
+                                   extra_link_args = [ os.path.join(SSL, 'lib/libssl.a'),
+                                                       os.path.join(SSL, 'lib/libcrypto.a'),
+                                                       '-ldl'],
                                    depends = ['socketmodule.h']), )
         else:
             missing.append('_ssl')
@@ -873,8 +873,11 @@
                 exts.append( Extension('_hashlib', ['_hashopenssl.c'],
                                        depends = ['hashlib.h'],
                                        include_dirs = ssl_incs,
-                                       library_dirs = ssl_libs,
-                                       libraries = ['ssl', 'crypto']) )
+                                       library_dirs = [],
+                                       extra_link_args = [ os.path.join(SSL, 'lib/libssl.a'),
+                                                           os.path.join(SSL, 'lib/libcrypto.a'),
+                                                         '-ldl'],
+                                       ) )
             else:
                 print("warning: openssl 0x%08x is too old for _hashlib" %
                       openssl_ver)
```

Save and exit the file, then

```
wget https://www.python.org/ftp/python/3.6.3/Python-3.6.3.tgz

tar -zxf Python-3.6.3.tgz

cd Python-3.6.3

LD_RUN_PATH=$installpath/python3.6.3/lib configure

LDFLAGS="-L $$installpath/python3.6.3/lib"

CPPFLAGS="-I $$installpath/python3.6.3/include"

LD_RUN_PATH=$$installpath/python3.6.3/lib make

export SSL_DIR=$installpath/openssl-1.1.0e

patch setup.py ../setup.py.patch

ln -s $installpath/openssl-1.1.0e/include/openssl

./configure --prefix=$installpath/python3.6.3

make

make install

cd ../../
```

Resume with step 3 of the normal installation above.
