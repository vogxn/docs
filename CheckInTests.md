# Check-In tests for Apache CloudStack

The agent [simulator](1) and [marvin](2) are integrated into maven build phases to help you run basic tests before pushing a commit. These tests are _integration tests_ that will test the CloudStack system as a whole. Management Server will be running during the tests with the Simulator Agent responding to hypervisor commands. For running the checkin tests, your developer environment needs to have [Marvin](2) installed and working with the latest CloudStack APIs. These tests are lightweight and should ensure that your commit doesnt break critical functionality for others working with the master branch.

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

### Marvin Installation and Configuration
Please refer to the [Marvin](Testing with Python) wiki page for how to install and configure marvin.

## Running integrated Simulator+Marvin tests

These build steps are similar to the regular build, deploydb and run of the management server. Only some extra switches are required to run the tests and should be easy to recall and run anytime:

### Building

Build with the -Dsimulator switch to enable simulator hypervisors
```bash
$ mvn -Pdeveloper -Dsimulator clean install 
```

### Deploy database

In addition to the regular deploydb you will be deploying the `simulator` database where all the agent information is stored for the mockvms, mockvolumes etc.
```bash
$ mvn -Pdeveloper -pl developer -Ddeploydb
$ mvn -Pdeveloper -pl developer -Ddeploydb-simulator
```

### Start the management server

Same as regular jetty:run.

```bash
$ mvn -pl client jetty:run
```

> To enable the debug ports before the run
> `export MAVEN_OPTS="-XX:MaxPermSize=512m -Xmx2g -Xdebug -Xrunjdwp:transport=dt_socket,address=8787,server=y,suspend=n"`

### Sync Marvin APIs

The integration test uses marvin and needs it to be installed in your system libraries where python can find them. The developer profile compilation above builds marvin for you and puts the tarball in tools/marvin/dist/Marvin-0.1.0.tar.gz. Marvin also provides the sync facility which contacts the API discovery plugin to rebuild API classes from your management server:
 
To install marvin from packages:

```bash
$ pip install Marvin-0.1.0.tar.gz
```

You can install/upgrade marvin using the sync mechanism.

```bash
$ sudo mvn -Pdeveloper,marvin.sync -Dendpoint=localhost -pl :cloud-marvin
```

This needs sudo privileges since it will call on pip to upgrade the existing marvin installation on your machine. The endpoint is where your management server is running and is exposing the API discovery plugin.

### Run the integration test

In a separate session you can use the following commands to bring up an advanced zone with two simulator hypervisors followed by run tests that are _tagged_ to run on the simulator:

##### marvin.setup to bring up a zone with the config specified
```bash
$ mvn -Pdeveloper,marvin.setup -Dmarvin.config=setup/dev/advanced.cfg -pl :cloud-marvin integration-test
```
Example configs are available in `setup/dev/advanced.cfg` and `setup/dev/basic.cfg`

##### marvin.test to run the checkin tests that are tagged to run on the simulator
```bash
$ mvn -Pdeveloper,marvin.test -Dmarvin.config=setup/dev/advanced.cfg -pl :cloud-marvin integration-test
```

Instructions on how to roll your own tests into the checkin tests can be found [here|https://cwiki.apache.org/confluence/display/CLOUDSTACK/Marvin+-+Testing+with+Python#Marvin-TestingwithPython-CheckinTests]


## Extending tests

## Simulator Configuration

## Test considerations 

[1] Simulator Wiki Link
[2] Marvin wiki link
[3] python download page
[4] http://docs.python.org/2/using/unix.html#building-python
[5] https://pypi.python.org/pypi/setuptools
[6] http://www.python.org/download/releases/
[7] http://www.lfd.uci.edu/~gohlke/pythonlibs/#pip
[8] https://pypi.python.org/pypi/setuptools#cygwin-mac-os-x-linux-other
[9] http://pydev.org/manual_101_root.html
