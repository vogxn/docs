# Check-In tests for Apache CloudStack

The agent [simulator][1] and [marvin][2] are integrated into maven build phases to help you run basic tests before pushing a commit. These tests are _integration tests_ that will test the CloudStack system as a whole. Management Server will be running during the tests with the Simulator Agent responding to hypervisor commands. For running the checkin tests, your developer environment needs to have [Marvin][2] installed and working with the latest CloudStack APIs. These tests are lightweight and should ensure that your commit doesnt break critical functionality for others working with the master branch.

## Prerequisites and Setup

If you have come to this wiki you should have the cloudstack code cloned to your local developer environment. Some basic developer tools - Eclipse IDE, shell environment, python installation are assumed. `mvn` should be building your code without issues.

> The shell environment should *not* have MAVEN_OPTS set with
> debug ports enabled. This intervenes with the multiple mvn runs we will be
> doing simultaneously - one to run the management server and another to drive
> the tests. If this is set - remember to unset the MAVEN_OPTS in the shell
> environment where tests are run.

### Linux, Mac

All the tests require Python 2.7 to be installed. If you have multiple pythons ensure that python2.7 is the default python selected when you type python in the shell prompt. If not - alias the `python2.7` binary in `/usr/local/bin` to `python`. 

If python 2.7 is not available on your machine or you have an old python - you will need to get python from the [python][3] website. Compiling from source is the easiest way to do this. Some old RHEL/CentOS machines have python 2.6 installed and installing python 2.7 can overwrite your yum installation. To overcome this - use the [make altinstall][4] option.

pip is a package manager for python but is not installed with python installation. To install pip you will need [setuptools][5] for your platform. Once setup tools is installed you can install pip as shown here:

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

### Windows
Windows python installation is easier since binary packages are simpler to click and install.

#### non-Cygwin
If you are not using cygwin download the python installer for Python 2.7 and run it from the [python downloads page][6]. Then install [pip][7] from binaries for python 2.7.

#### Cygwin
Run the Cygwin setup.exe executable. On the dialog boxes under the Python section you should see the latest python 2.7 interpreter. Install it. If you have an older python already installed in Cygwin make sure to uninstall that before upgrading to 2.7. Set your environment variables to point `python` to the new `python 2.7` installed python on your cygdrive. Try alternate mirrors if you find python 2.7 is not available on your default mirror.

To install pip the package manager you will have to install [setuptools][8] for cygwin and then do:

```bash
$ easy_install pip
```
This should link the python 2.7 installation and the pip installation for you.

## Running tests
h2. Integrated Simulator+Marvin test

To run the basic tests. Your build steps should be as follows

h4. Building


{code}
$ mvn -Pdeveloper -Dsimulator clean install 
{code}

h4. Deploy database


{code}
$ mvn -Pdeveloper -pl developer -Ddeploydb
$ mvn -Pdeveloper -pl developer -Ddeploydb-simulator
{code}

h4. Start the management server


{code}
mvn -pl client jetty:run
{code}
{color:#000000}{*}_(One time installation of Marvin)_{*}{color}

The integration test uses marvin and needs it to be installed in your system libraries where python can find them. The developer profile compilation above builds marvin for you and puts the tarball in tools/marvin/dist/Marvin-0.1.0.tar.gz.

To install marvin:

{code}
easy_install pip #if you need to install pip
pip install Marvin-0.1.0.tar.gz
{code}

Alternatively one can install/upgrade marvin using the sync mechanism.

h4. *Sync latest APIs*

Sometimes you want to synchronize marvin with the new APIs that you have introduced locally or any alterations you may have made to an existing API. In such cases you can sync marvin's libraries as follows.&nbsp;

{code}
$ sudo mvn -Pdeveloper,marvin.sync -Dendpoint=localhost -pl :cloud-marvin
{code}
This needs sudo privileges since it will call on pip to upgrade the existing marvin installation on your machine. The endpoint is where your management server is running and is exposing the API discovery plugin.

h4. {color:#000000}{*}Run the integration test{*}{color}

In a separate session you can use the following commands to bring up an advanced zone with two simulator hypervisors followed by run tests that are 'tagged' to run on the simulator:
{code}
$ mvn -Pdeveloper,marvin.setup -Dmarvin.config=setup/dev/advanced.cfg -pl :cloud-marvin integration-test #Setup environment
$ mvn -Pdeveloper,marvin.test -Dmarvin.config=setup/dev/advanced.cfg -pl :cloud-marvin integration-test  #Run checkin tests
{code}

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
