# Marvin - Testing with Python

[Marvin](http://en.wikipedia.org/wiki/Marvin_the_Paranoid_Android) - is a python testing framework for Apache CloudStack. Tests written using the framework use the unittest module under the hood. The unittest module is the Python version of the unit-testing framework originally developed by Kent Beck et al and will be familiar to the Java people in the form of JUnit. The following document will act as a tutorial introduction to those interested in testing CloudStack with python. This document does not cover the python language and we will be pointing the reader instead to explore some tutorials that are more thorough on the topic. In the following we will be assuming basic python scripting knowledge from the reader. The reader is encouraged to walk through the tutorial in full and explore the existing tests in the repository before beginning to write tests.

## Installation
Marvin requires Python 2.6 for installation but the tests written using marvin utilize Python 2.7 to the fullest. You should follow the test environment setup instructions [here](TestEnvironmentSetup) before proceeding further.

### Dependencies
* Marvin dependencies are installed automatically by the installer 
    * python-paramiko - remote ssh access, ssh-agent 
    * mysql-connector-python - db connectivity 
    * nose - test runner, plugins
    * marvin-nose - marvin plugin for nose
    * should_dsl - should assertions dsl-style

### Windows specifics
* For Windows (non-cygwin) development environment, following additional steps are needed to be able to install Marvin:
    * The mysql-connector-python for Windows can be found [here](http://dev.mysql.com/downloads/connector/python/)
    * pycrypto binaries are required since Windows does not have gcc/glibc. Download [here](http://www.voidspace.org.uk/python/modules.shtml#pycrypto)

### Compiling and Packaging
The `developer` profile compiles and packages Marvin. It does *NOT* install marvin your default PYTHONPATH.

```bash
$ mvn -P developer -pl :cloud-marvin
```
Alternately, you can fetch marvin from jenkins as well: packages are built every hour [here](https://builds.apache.org/job/cloudstack-marvin/)

### Installation
```bash
$ pip install tools/marvin/dist/Marvin-0.1.0.tar.gz
```

To upgrade an existing marvin installation and sync it with the latest APIs on the managemnet server, follow the steps [here](CheckInTests#Sync)

## First Things First
You will need a CloudStack management server that is already deployed, configured and ready to accept API calls. You can pick any management server in your lab that has a few VMs running on it or you can use [DevCloud](DevCloud) or the simulator environment deployed for a [checkin test](CheckInTest). Create a sample json config file `demo.cfg` telling marvin where your management server and database server are. 

The `demo.cfg` json config looks as shown below:

```json
{
    "dbSvr": {
        "dbSvr": "marvin.testlab.com",
        "passwd": "cloud",
        "db": "cloud",
        "port": 3306,
        "user": "cloud"
    },
    "logger": [
        {
            "name": "TestClient",
            "file": "/tmp/testclient.log"
        },
        {
            "name": "TestCase",
            "file": "/tmp/testcase.log"
        }
    ],
    "mgtSvr": [
        {
            "mgtSvrIp": "marvin.testlab.com",
            "port": 8096
        }
    ]
}
```

> * Note: `dbSvr` is the location where mysql server is running and passwd is the password for user cloud
> * open up the integration.port on your management server `iptables -I INPUT -p tcp --dport 8096 -j ACCEPT`
> * Change the global setting `integration.api.port` on the CloudStack GUI to `8096` and restart the management server

## Writing the test
Without much ado, let us jump into test case writing. Following is a working scenario we will test using Marvin:

* create a testcase class
* setUp a user account - name: user, passwd: password
* deploy a VM into that user account using the default small service offering and CentOS template
* verify that the VM we deployed reached the "Running" state
* tearDown the user account - basically delete it to cleanup acquired resources

```python
from marvin.cloudstackTestCase import *
from marvin.cloudstackAPI import *

from marvin.integration.lib.utils import *
from marvin.integration.lib.base import *
from marvin.integration.lib.common import *

class TestData(object):
    def __init__(self):
       self.testdata = {
                "account": {
                    "email": "test@test.com",
                    "firstname": "Test",
                    "lastname": "User",
                    "username": "test",
                    "password": "password",
                },
                "service_offering": {
	                "small": {
	                        "name": "Small Instance",
	                        "displaytext": "Small Instance",
	                        "cpunumber": 1,
	                        "cpuspeed": 100,
	                        "memory": 256,
		                },
                },
                "ostype": 'CentOS 5.3 (64-bit)',
        }

class TestDeployVM(cloudstackTestCase):
    """Test deploy a VM into a user account
    """

    def setUp(self):
        self.testdata = TestData().testdata
        self.apiclient = self.testClient.getApiClient()

        # Get Zone, Domain and Default Built-in template
        domain = get_domain(self.apiclient, self.testdata)
        zone = get_zone(self.apiclient, self.testdata)
        self.testdata["mode"] = zone.networktype
        template = get_template(self.apiclient, zone.id, self.testdata["ostype"])

        self.account = Account.create(
            self.apiclient,
            self.testdata["account"],
            domainid=domain.id,
            zoneid=zone.id
        )
        self.service_offering = ServiceOffering.create(
            self.apiclient,
            self.testdata["service_offering"]["small"]
        )
        self.cleanup = [
            self.service_offering,
            self.account
        ]

    def test_deploy_vm(self):
        """Test Deploy Virtual Machine

        # Validate the following:
        # 1. Virtual Machine is accessible via SSH
        # 2. listVirtualMachines returns accurate information
        """
        self.virtual_machine = VirtualMachine.create(
            self.apiclient,
            self.services["small"],
            accountid=self.account.name,
            domainid=self.account.domainid,
            serviceofferingid=self.service_offering.id,
            mode=self.services['mode']
        )

        list_vms = VirtualMachine.list(self.apiclient, id=self.virtual_machine.id)

        self.debug(
                "Verify listVirtualMachines response for virtual machine: %s" \
                % self.virtual_machine.id
            )

        self.assertEqual(
                            isinstance(list_vms, list),
                            True,
                            "List VM response was not a valid list"
                        )
        self.assertNotEqual(
                            len(list_vms),
                            0,
                            "List VM response was empty"
                        )

        vm = list_vms[0]
        self.assertEqual(
                            vm.id,
                            self.virtual_machine.id,
                            "Virtual Machine ids do not match"
                        )
        self.assertEqual(
                    vm.name,
                    self.virtual_machine.name,
                    "Virtual Machine names do not match"
                    )
        self.assertEqual(
            vm.state,
            "Running",
             msg="VM is not in Running state"
        )

    def tearDown(self):
        try:
            cleanup_resources(self.apiclient, self.cleanup)
        except Exception as e:
            self.debug("Warning! Exception in tearDown: %s" % e)
```

## Running the test

### IDE - Eclipse and PyDev
In PyDev you will have to setup the default test runner to be nose. For this:
- goto Window->Preferences->PyDev
- PyUnit - set the testrunner to nosetests
- In the options window specify the following
  * --with-marvin
  * --marvin-config=/path/to/demo.cfg
  * --load
- Save/Apply

Create a new python module `test_deploy_vm` of type unittest, copy the above test class and save the file. Running debug will now run the test case using the nosetests runner. You will also be able to set breakpoints, inspect values, evaluate expressions while debugging like you do with Java code in Eclipse.

### CLI - bash run with nosetests 
```bash
tsp@cloud:~/cloudstack# nosetests --with-marvin --marvin-config=demo.cfg --load test_deploy_vm.py

test_DeployVm (testDeployVM.TestDeployVm) ... ok
----------------------------------------------------------------------
Ran 1 test in 100.511s
OK
```

Congratulations, your test has passed!

## Advanced test case

We do not know for sure that the CentOS VM deployed earlier actually started up on the hypervisor host. The API tells us it did - so Cloudstack assumes the VM is up and running, but did the hypervisor successfully spin up the VM? In this example we will login to the CentOS VM that we deployed earlier using a simple ssh client that is exposed by the test framework. The example assumes that you have an Advanced Zone deployment of Cloudstack running. The test case is further simplified if you have a Basic Zone deployment. It is left as an exercise to the reader to refactor the following test to work for a basic zone.

Let's get started. We will take the earlier test as is and extend it as follows:

* Creating a NAT (PortForwarding) rule that allows ssh (port 22) traffic
* Open up the firewall to allow all SSH traffic to the account's VMs
* Add the deployed VM returned in our previous test to this port forward rule
* ssh to the NAT-ed IP using our ssh-client and get the hostname of the VM
* Compare the hostname of the VM and the name of the VM deployed by CloudStack.
Both should match for our test to be deemed : PASS

> NOTE: This test has been written for the >3.0 CloudStack. On 2.2 versions we do not explicitly create a firewall rule.

```python
```

Observe that unlike the previous test class - `TestDeployVM` - we do not have methods `setUp` and `tearDown`. Instead, we have the methods `setUpClass` and `tearDownClass`. We do not want the initialization (and cleanup) code in setup (and teardown) to run after every test in the suite which is what `setUp` and `tearDown` will do. Instead we will have the initialization code (creation of account etc) done once for the entire lifetime of the class. This is accomplished using the `setUpClass` and `tearDownClass` classmethods. Since the API client is only visible to instances of `cloudstackTestCase` we expose the API client at the class level using the `getClsTestClient()` method. So to get the API client we call the parent class (super(`TestSshDeployVm`, cls)) ie `cloudstackTestCase` and ask for a class level API client.

## Test Pattern

An astute reader would by now have found that the following pattern has been used in the test examples shown so far:

* creation of an account
* deploying Vms, running some test scenario
* deletion of the account

This pattern is useful to contain the entire test into one atomic piece. It helps prevent tests from becoming entangled in each other ie we have failures localized to one account and that should not affect the other tests.  Advanced examples in our basic verification suite are written using this pattern. Those writing tests are encouraged to follow the examples in `test/integration/smoke` directory. 

## User Tests

The test framework by default runs all its tests under 'admin' mode which means you have admin access and visibility to resources in cloudstack. In order to run the tests as a regular user/domain-admin - you can apply the @user decorator which takes the arguments (account, domain, accounttype) at the head of your test class. The decorator will create the account and domain if they do not exist. 

If the decorator is not suitable for you, for instance, you have to make some API calls as an admin and others as a user you can get two apiClients in your test. One for the admin accessiblity operations and another for user access only. `getUserApiClient()`

## Debugging & Logging

The logs from the test client detailing the requests sent by it and the responses fetched back from the management server can be found under `/tmp/testclient.log`. By default all logging is in `INFO` mode. In addition, you may provide your own set of `DEBUG` log messages in tests you write. Each `cloudstackTestCase` inherits the debug logger and can be used to output useful messages that can help troubleshooting the testcase when it is running.

eg:
```python
list_zones_response = self.apiclient.listZones(listzonesample)
self.debug("Number of zones: %s" % len(list_zones_response)) #This shows us how many zones were found in the deployment
```

While debugging with the PyDev plugin you can also place breakpoints in Eclipse for a more interactive debugging session.

## Marvin Deployment Configurations 

Marvin can be used to configure a deployed Cloudstack installation with Zones, Pods and Hosts automatically in to _Advanced_ or _Basic_ network types. This is done by describing the required deployment in a hierarchical json configuration file. But writing and maintaining such a configuration is cumbersome and error prone. Marvin's configGenerator is designed for this purpose. A simple hand written python description passed to the configGenerator will generate the compact json configuration of our deployment.

Examples of how to write the configuration for various zone models is within the configGenerator.py module in your marvin source directory. Look for methods `describe_setup_in_advanced_mode()/ describe_setup_in_basic_mode()`.

### What does it look like?

Shown below is such an example describing a simple one host deployment:
```json
{
    "zones": [
        {
            "name": "Sandbox-XenServer",
            "guestcidraddress": "10.1.1.0/24",
            "physical_networks": [
                {
                    "broadcastdomainrange": "Zone",
                    "name": "test-network",
                    "traffictypes": [
                        {
                            "typ": "Guest"
                        },
                        {
                            "typ": "Management"
                        },
                        {
                            "typ": "Public"
                        }
                    ],
                    "providers": [
                        {
                            "broadcastdomainrange": "ZONE",
                            "name": "VirtualRouter"
                        }
                    ]
                }
            ],
            "dns1": "10.147.28.6",
            "ipranges": [
                {
                    "startip": "10.147.31.150",
                    "endip": "10.147.31.159",
                    "netmask": "255.255.255.0",
                    "vlan": "31",
                    "gateway": "10.147.31.1"
                }
            ],
            "networktype": "Advanced",
            "pods": [
                {
                    "endip": "10.147.29.159",
                    "name": "POD0",
                    "startip": "10.147.29.150",
                    "netmask": "255.255.255.0",
                    "clusters": [
                        {
                            "clustername": "C0",
                            "hypervisor": "XenServer",
                            "hosts": [
                                {
                                    "username": "root",
                                    "url": "http://10.147.29.58",
                                    "password": "password"
                                }
                            ],
                            "clustertype": "CloudManaged",
                            "primaryStorages": [
                                {
                                    "url": "nfs://10.147.28.6:/export/home/sandbox/primary",
                                    "name": "PS0"
                                }
                            ]
                        }
                    ],
                    "gateway": "10.147.29.1"
                }
            ],
            "internaldns1": "10.147.28.6",
            "secondaryStorages": [
                {
                    "url": "nfs://10.147.28.6:/export/home/sandbox/secondary"
                }
            ]
        }
    ],
    "dbSvr": {
        "dbSvr": "10.147.29.111",
        "passwd": "cloud",
        "db": "cloud",
        "port": 3306,
        "user": "cloud"
    },
    "logger": [
        {
            "name": "TestClient",
            "file": "/var/log/testclient.log"
        },
        {
            "name": "TestCase",
            "file": "/var/log/testcase.log"
        }
    ],
    "globalConfig": [
        {
            "name": "storage.cleanup.interval",
            "value": "300"
        },
        {
            "name": "account.cleanup.interval",
            "value": "600"
        }
    ],
    "mgtSvr": [
        {
            "mgtSvrIp": "10.147.29.111",
            "port": 8096
        }
    ]
}
```

What you saw in the beginning of the tutorial was a condensed form of this complete configuration file. If you are familiar with the CloudStack installation you will recognize that most of these are settings you give in the install wizards as part of configuration. What is different from the simplified configuration file are the sections _zones_ and _globalConfig_. The _globalConfig_ section is nothing but a simple listing of (_key_, _value_) pairs for the _Global Settings_ section of CloudStack.

The `zones` section defines the hierarchy of our cloud. At the top-level are the availability zones. Each zone has its set of pods, secondary storages, providers and network related configuration. Every pod has a bunch of clusters and every cluster a set of hosts and their associated primary storage pools. These configurations are easy to maintain and deploy by just passing them through marvin.

```bash
tsp@cloud:~/cloudstack# nosetests --with-marvin --marvin-config=advanced.cfg -w /tmp #Empty directory where there are no tests to be discovered
```

Notice that we did not pass the --load option to nose. The reason being we want to deploy the configuration specified. This is the default behaviour of Marvin wherein the cloud configuration is deployed and the tests in the directory are run against it. Here we have pointed to a likely empty directory so as to only deploy and configure the zone.

### How do I generate it?

The above one host configuration was described as follows:

```python
import random
import marvin
from marvin.configGenerator import *

def describeResources():
    zs = cloudstackConfiguration()

    z = zone()
    z.dns1 = '10.147.28.6'
    z.internaldns1 = '10.147.28.6'
    z.name = 'Sandbox-XenServer'
    z.networktype = 'Advanced'
    z.guestcidraddress = '10.1.1.0/24'

    pn = physical_network()
    pn.name = "test-network"
    pn.traffictypes = [traffictype("Guest"), traffictype("Management"), traffictype("Public")]
    z.physical_networks.append(pn)

    p = pod()
    p.name = 'POD0'
    p.gateway = '10.147.29.1'
    p.startip =  '10.147.29.150'
    p.endip =  '10.147.29.159'
    p.netmask = '255.255.255.0'

    v = iprange()
    v.gateway = '10.147.31.1'
    v.startip = '10.147.31.150'
    v.endip = '10.147.31.159'
    v.netmask = '255.255.255.0'
    v.vlan = '31'
    z.ipranges.append(v)

    c = cluster()
    c.clustername = 'C0'
    c.hypervisor = 'XenServer'
    c.clustertype = 'CloudManaged'

    h = host()
    h.username = 'root'
    h.password = 'password'
    h.url = 'http://10.147.29.58'
    c.hosts.append(h)

    ps = primaryStorage()
    ps.name = 'PS0'
    ps.url = 'nfs://10.147.28.6:/export/home/sandbox/primary'
    c.primaryStorages.append(ps)

    p.clusters.append(c)
    z.pods.append(p)

    secondary = secondaryStorage()
    secondary.url = 'nfs://10.147.28.6:/export/home/sandbox/secondary'
    z.secondaryStorages.append(secondary)

    '''Add zone'''
    zs.zones.append(z)

    '''Add mgt server'''
    mgt = managementServer()
    mgt.mgtSvrIp = '10.147.29.111'
    zs.mgtSvr.append(mgt)

    '''Add a database'''
    db = dbServer()
    db.dbSvr = '10.147.29.111'
    db.user = 'cloud'
    db.passwd = 'cloud'
    zs.dbSvr = db

    '''Add some configuration'''
    [zs.globalConfig.append(cfg) for cfg in getGlobalSettings()]

    ''''add loggers'''
    testClientLogger = logger()
    testClientLogger.name = 'TestClient'
    testClientLogger.file = '/var/log/testclient.log'

    testCaseLogger = logger()
    testCaseLogger.name = 'TestCase'
    testCaseLogger.file = '/var/log/testcase.log'

    zs.logger.append(testClientLogger)
    zs.logger.append(testCaseLogger)
    return zs

def getGlobalSettings():
   globals = { "storage.cleanup.interval" : "300",
               "account.cleanup.interval" : "60",
            }

   for k, v in globals.iteritems():
        cfg = configuration()
        cfg.name = k
        cfg.value = v
        yield cfg

if __name__ == '__main__':
    config = describeResources()
    generate_setup_config(config, 'advanced_cloud.cfg')
```

The `zone(), pod(), cluster(), host()` are plain objects that carry just attributes. For instance a _zone_ consists of the attributes - `name, dns entries, network type` etc. Within a zone I create `pod()s` and append them to my zone object, further down creating `cluster()s` in those pods and appending them to the pod and within the clusters finally my `host()s` that get appended to my cluster object. Once I have defined all that is necessary to create my cloud I pass on the described configuration to the `generate_setup_config()` method which gives me my resultant configuration in JSON format.

## Nose Plugins

Nose extends unittest to make testing easier. Nose comes with plugins that help integrating your regular unittests into external build systems, coverage, profiling etc. Marvin comes with its own nose plugin for this so you can use nose to drive CloudStack tests. The plugin is installed on installing marvin. Running `nosetests -p` will show if the plugin registered successfully.
```bash
$ nosetests -p
Plugin xunit
Plugin multiprocess
Plugin capture
Plugin logcapture
Plugin coverage
Plugin attributeselector
Plugin doctest
Plugin profile
Plugin collect-only
Plugin isolation
Plugin pdb
Plugin marvin
```

### Usage and running tests
```bash
$ nosetests --with-marvin --marvin-config=/path/to/basic_zone.cfg --load /path/to/tests
```

### Attribute selection
The smoke tests and component tests contain attributes that can be used to filter the tests that you would like to run against your deployment. You would use nose attrib plugin for this. Following tags are available for filtering:

* advanced - Typical Advanced Zone
* basic - a basic zone without security groups
* sg - a basic zone with security groups
* eip - an elastic ip basic zone
* advancedns - advanced zone with a netscaler device
* advancedsg - advanced zone with security groups
* devcloud - tests that will run only for the basic zone on a devcloud setup done using tools/devcloud/devcloud.cfg
* speed = 0/1/2 (greater the value lesser the speed)
* multihost/multipods/mulitcluster (test requires multiple set of hosts/pods/clusters)

### Running Devcloud Tests

Some tests have been tagged to run only for devcloud environment. In order to run these tests you can use the following command after you have setup your management server and the host only devcloud is running with `tools/devcloud/devcloud.cfg` as its deployment configuration. 

```bash
tsp@cloud:~/cloudstack# nosetests --with-marvin --marvin-config=tools/devcloud/devcloud.cfg --load -a tags='devcloud' test/integration/smoke

Test Deploy Virtual Machine ... ok
Test Stop Virtual Machine ... ok
Test Start Virtual Machine ... ok
Test Reboot Virtual Machine ... ok
Test destroy Virtual Machine ... ok
Test recover Virtual Machine ... ok
Test destroy(expunge) Virtual Machine ... ok

----------------------------------------------------------------------

Ran 7 tests in 10.001s

OK
```

## Checkin Tests

These are tests that can be run if you have a development environment of CloudStack. Light-Weight tests such as these can be run using the simulator as the hypervisor. The run time for tests is short and you should ensure that you have run them before making a commit that could potentially break basic functionality. Instructions for setting up and running these tests is explained [here](CheckInTests)

Check-In tests are the same as any other tests written using Marvin. The only additional step you need to do is ensure that your test is driven entirely by the API only. This makes it possible to run the test on a simulator. Once you have your test, you need to tag it to run on the simulator so the marvin test runner can pick it up during the checkin-test run. Then place your test module in the `test/integration/smoke` folder and it will become part of the checkin test run.

For eg:
```python
@attr(tags =["simulator", "advanced", "smoke"])
def test_deploy_virtualmachine(self):
    """Tests deployment of VirtualMachine
    """
```

## Existing Tests - Smoke and Component

Tests with more backend verification and complete integration of suites for network, snapshots, templates etc can be found in the `test/integration/smoke` and `test/integration/component`. Almost all of these test suites use common library wrappers written around the test framework to simplify writing tests. These libraries are part of the `marvin.integration` package. Ensure that you have gone through the existing tests related to your feature before writing your own. 

The integration library takes advantage of the fact that every resource - VirtualMachine, ISO, Template, PublicIp etc follows the pattern of
* create - where we cause creation of the resource eg: deployVirtualMachine
* delete - where we delete our resource eg: deleteVolume
* list - where we look for some state of the resource eg: listPods

Marvin can auto-generate these resource classes using API discovery. The auto-generation ability is being added as part of [this refactor](MarvinRefactor)

## Guidelines to choose scenarios for integration

There are a few do's and don'ts in choosing the automated scenario for an integration test. These are mostly for the system to blend well with the continuous test infrastructure and to keep environments pure and clean without affecting other tests.

### Scenario

* Every test should happen within a CloudStack user account. The order in which you choose the type of account to test within should be
            User > DomainAdmin > Admin
  At the end of the test we delete this account so as to keep tests atomic and contained within a tenant's users space.

* All tests must be written with the perspective of the API. UI directions are often confusing and using the rich API often reveals further test scenarios. You can capture the API arguments using cloudmonkey/firebug.
* Tests should be generic enough to run in any environment/lab - under any hypervisor. If this is not possible then it is appropriate to mark the test with an @attr attribute to signify any specifics. eg: @attr(hypervisor='vmware') for runs only on vmware
* Every resource should be creatable in the test from scratch. Referring to an Ubuntu template is probably not a good idea. Your test must show how to fetch this template or give a static location from where the test can fetch it.
* Do not change global settings configurations in between a test. Make two separate tests for this. All tests run against one given deployment and altering the settings midway is not effective

### Backend Verification with paramiko/and other means

* Verifying status of resources within hypervisors is fine. Most hypervisors provide standard SSH server access
* Your tests should include the complete command and its expected output.&nbsp; e.g. `iptables -L INPUT` to list the INPUT chain of iptables
* If you execute multiple commands then all of them should be chained together on one line e.g: `service iptables stop; service iptables start #to stop and start iptables`
* Your script must execute over ssh because this is how Marvin will execute it `ssh <target-backend-machine> "<your script>"`
* Most external devices like F5/ NetScaler have ssh open to execute commands. But you must include the command and its expected output in the test as not everyone is aware of the device's CLI
* If you are using a UI like vCenter to verify something most likely you cannot automate what you see there because there are no ASLv2 licensed libraries for vmware/esx as of today.

## Python Resources

* The single largest python resource is the [python website itself](http://www.python.org)
* Mark Pilgrim's - "[Dive Into Python](http://www.diveintopython.net)" - is another great resource. The book is available for free online. Chapter 1 to 6 cover a good portion of language basics and Chapter 13 & 14 are essential for anyone doing test script development. 
* You can also use Zed Shaw's book, [Learn Python the Hard Way](http://learnpythonthehardway.org/), if programming is something completely new to you
* To read more about the assert methods the language reference is the ideal place - [http://docs.python.org/library/unittest.html].

## Acknowledgements
* The original author of the testing framework - Edison Su
* Maintenance and bug fixes - Prasanna Santhanam
* Documentation - Prasanna and Edison

For any feedback, typo corrections please email the dev@cloudstack.apache.org lists
