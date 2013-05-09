# Test Environment Setup 

To be able to write and run the tests written for Apache CloudStack the test environment setup is required. The tests are written using the [Marvin](marvin) test framework and use python libraries. Following document describes the prerequsite tools to get started with tests.

## Prerequisites and Setup

If you have come to this wiki you should have the cloudstack code cloned to your local developer environment. Some basic developer tools - Eclipse IDE, shell environment, python installation are assumed. `mvn` should be building your code without issues.

> The shell environment should *not* have MAVEN_OPTS set with
> debug ports enabled. This intervenes with the multiple mvn runs we will be
> doing simultaneously - one to run the management server and another to drive
> the tests. If this is set - remember to unset the MAVEN_OPTS in the shell
> environment where tests are run.

### Python

#### Linux, Mac

All the tests require Python 2.7 to be installed. If you have multiple pythons ensure that python2.7 is the default python selected when you type python in the shell prompt. If not - alias the `python2.7` binary in `/usr/local/bin` to `python`. 

If python 2.7 is not available on your machine or you have an old python - you will need to get python from the [python](3) website. Compiling from source is the easiest way to do this. Some old RHEL/CentOS machines have python 2.6 installed and installing python 2.7 can overwrite your yum installation. To overcome this - use the [make altinstall](4) option.

pip is a package manager for python but is not installed with python installation. To install pip you will need [setuptools](5) for your platform. Once setup tools is installed you can install pip as shown here:

```bash
$ easy_install pip
```

Some linux platforms also package pip as an rpm so you can simply do:

```bash
$ yum install python-pip
```
Note: python-pip package would have been packaged for the default python installation on your distro. 

For those with multiple python enviornments make sure to alias pip-2.7 to pip so that the packages go to the python2.7 package directories and not python 2.6/2.4 as the case may be. To see which python the default pip is pointing to:

```bash
$ pip --version
pip 1.2.1 from /Library/Python/2.7/site-packages/pip-1.2.1-py2.7.egg (python 2.7)
```

Mac OSX users who have homebrew can simply install python using brew. This will also install the right pip version for you:

```bash
$ brew install python
```

#### Windows
Windows python installation is easier since binary packages are simpler to click and install.

##### Non-Cygwin
If you are not using cygwin download the python installer for Python 2.7 from the [python downloads page](6) and run it.Then install [pip](7) from binaries for python 2.7.

##### Cygwin
Run the Cygwin setup.exe executable. On the dialog boxes under the Python section you should see the latest python 2.7 interpreter. Install it. If you have an older python already installed in Cygwin make sure to uninstall that before upgrading to 2.7. Set your environment variables to point `python` to the new `python 2.7` installed python on your cygdrive. Try alternate mirrors if you find python 2.7 is not available on your default mirror.

To install pip the package manager you will have to install [setuptools](8) for cygwin and then do:

```bash
$ easy_install pip
```
This should link the python 2.7 installation and the pip installation for you.

> Warning: The pydev plugin does not work well with cygwin and has problems
> resolving paths between cygwin and windows environments. At this time there
> is no available alternative for auto-completion plugins for Eclipse

### PyDev installation

[PyDev](9) is a python plugin for eclipse which features auto-completion of python modules. To install PyDev -
- go to Help->Install New Software
- in the Work With: dialog box - add `http://pydev.org/updates`
- select the pydev packages shown and click on Install

That should be it. If you face any problems follow the more in-depth pydev installation [here](http://pydev.org/manual_101_root.html). 

You should configure your pydev installation to use the python2.7 interpreter you installed and configured earlier. To do this :
- go to Window -> Preferences
- Pydev -> Interpreter
- Either use AutoConfig or select the path to you python executable

