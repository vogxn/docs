# Check-In tests for Apache CloudStack

The agent [simulator](1) and [marvin](2) are integrated into maven build phases to help you run basic tests before pushing a commit. These tests are _integration tests_ that will test the CloudStack system as a whole. Management Server will be running during the tests with the Simulator Agent responding to hypervisor commands. For running the checkin tests, your developer environment needs to have [Marvin](2) installed and working with the latest CloudStack APIs. These tests are lightweight and should ensure that your commit doesnt break critical functionality for others working with the master branch.

### Environment Setup
Follow the test environment [setup](TestEnvironmentSetup) to get started with Python, Eclipse and PyDev configurations

### Marvin Installation and Configuration
The checkin-tests utilize marvin and a one-time installation of marvin will need to be done so as to fetch all the related dependencies. Further updates to marvin can be done by using the sync mechanism described later.

Please refer to the [Marvin](Marvin) wiki page for how to configure marvin and its dependencies.

## Running integrated checkin tests

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

In a separate session you can use the following commands to bring up an advanced zone with two simulator hypervisors followed by run tests that are _tagged_ to work on the simulator:

##### marvin.setup to bring up a zone with the config specified
```bash
$ mvn -Pdeveloper,marvin.setup -Dmarvin.config=setup/dev/advanced.cfg -pl :cloud-marvin integration-test
```
Example configs are available in `setup/dev/advanced.cfg` and `setup/dev/basic.cfg`

##### marvin.test to run the checkin tests that are tagged to run on the simulator
```bash
$ mvn -Pdeveloper,marvin.test -Dmarvin.config=setup/dev/advanced.cfg -pl :cloud-marvin integration-test
```
## Simulator Configuration
The sample simulator configurations for advanced and basic zone is available in `setup/dev/` directory. The default configuration `setup/dev/advanced.cfg` deploys an advanced zone with two simulator hypervisors in a single cluster in a single pod, two primary NFS storage pools and a secondary storage NFS store. 

If your test requires any extra hypervisors, storage pools, additional IP allocations, VLANs etc - you should adjust the configuration accordingly. Ensure that you have run all the checkin tests in the new configuration. For this you can directly edit the JSON file or generate a new configuration file. The `setup/dev/advanced.cfg` was generated as follows

```bash
$ cd tools/marvin/marvin/sandbox/advanced
$ python advanced_env.py -i setup.properties -o advanced.cfg
```

These configurations are generated using the marvin *configGenerator* module. You can write your own configuration by following the examples shown in the `configGenerator` module:

- `describe_setup_in_basic_mode()`
- `describe_setup_in_advanced_mode()`
- `describe_setup_in_eip_mode()`

More detailed explanation of how the JSON configuration works is in the [Marvin tutorial](How to generate it).

## Extending tests
Instructions on how to roll your own tests into the checkin tests can be found [here|https://cwiki.apache.org/confluence/display/CLOUDSTACK/Marvin+-+Testing+with+Python#Marvin-TestingwithPython-CheckinTests]

### Test considerations
*Test considerations* - Tests written to be part of the checkin run must run on the simulator. For

[1] Simulator Wiki Link
[2] Marvin wiki link
[3] python download page
[4] http://docs.python.org/2/using/unix.html#building-python
[5] https://pypi.python.org/pypi/setuptools
[6] http://www.python.org/download/releases/
[7] http://www.lfd.uci.edu/~gohlke/pythonlibs/#pip
[8] https://pypi.python.org/pypi/setuptools#cygwin-mac-os-x-linux-other
[9] http://pydev.org/manual_101_root.html
