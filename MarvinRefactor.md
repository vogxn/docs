# Marvin Refactor
The Apache CloudStack Marvin test framework will undergo some key improvements as part of this refactor:

1. All CloudStack resources modelled as entities
2. Easier test data management with Factories
3. DSL support for basic test case writing

## Resources modelled as entities

Typically to write a test case previously the test case author was expected to know (in advance) all the APIs he/she was going to call to complete his scenario. With the growing list of APIs, their parameters and optional arguments it becomes tedious often to compose a single API call. To overcome this the integration libraries were written. These libraries (`integration.lib.base, integration.lib.common` etc) present a list of resources or entities - eg: VirtualMachine, VPC, VLAN to the library user. Each entity can perform a set of operations that in turn transform into an API call. 

```python
class VirtualMachine(object):
    def deploy(self, apiclient, service, template, zone):
        cmd = deployVirtualMachine.deployVirtualMachineCmd()
        cmd.serviceofferingid = service
        cmd.templateid = template
    ...
    ...
    def list(self,apiclient)
        cmd = listVirtualMachines.listVirtualMachinesCmd()
        return apiclient.listVirtualMachines(cmd)
```
This makes the library usage more object-oriented. So in the testcase the author only has to make a call to the VirtualMachine when dealing with all the operations he/she required off the virtualmachines one would be creating/deleting/restoring/destroying etc.

The disadvantage of this approach is that the integration library is hand-written and brittle when changes are made breaking several tests in the process. There are also inconsistencies caused by mixing the data required for the API call with the arguments of the operation. eg:

```python
class VirtualMachine(object):
....
    @classmethod
    def create(cls, apiclient, services, templateid=None, accountid=None,
                    domainid=None, zoneid=None, networkids=None, serviceofferingid=None,
                    securitygroupids=None, projectid=None, startvm=None,
                    diskofferingid=None, affinitygroupnames=None, group=None,
                    hostid=None, keypair=None, mode='basic', method='GET'):
             ....
             ....
````
Here the services dictionary looks for a key named 'accountid' when it is actually looking for 'accountname'. Also the naming and the size of the API call is daunting enough to anyone using the library. 

> Factories (discussed later) will make passing all this data simpler.

The refactor will address libraries inconsistencies by auto-generating all the class entities and their operations. For this -the entity that an API acts upon will be exposed by the API discovery service using the `entity` attribute. eg: dedicatePublicIpRange

```json
listapisresponse: {
    count: 1,
    api: [
    {
        name: "dedicatePublicIpRange",
        description: "Dedicates a Public IP range to an account",
        isasync: false,
        related: "listVlanIpRanges",
        params: [],
        response: [],
        entity: "VlanIpRange"
     }
    ]
  }
}
````

This transforms into the following Marvin entity class:
```python
class VlanIpRange(CloudStackEntity):

    def dedicate(self, apiclient, account, domainid, **kwargs):
        cmd = dedicatePublicIpRange.dedicatePublicIpRangeCmd()
        cmd.id = self.id
        cmd.account = account
        cmd.domainid = domainid
        [setattr(cmd, key, value) for key,value in kwargs.iteritems()]
        publiciprange = apiclient.dedicatePublicIpRange(cmd)
        return publiciprange if publiciprange else None

```

- [ ] API Exceptions - markDefaultZoneForAccount
- [ ] Splitting verb and entity - transformers, dealing with exceptions 
- [ ] CS Entity Generator depends on codegenerator (_combine?_)

### Required and Optional Arguments

All required arguments to an API will be available in the API operation 
```python
Entity.verb(reqd1=None, reqd2=None, ..., **kwargs)
```

Here the `Entity` (eg:VirtualMachine) can perform an operation `verb()` using the arguments `[reqd1, reqd2]`. The optional arguments (if any) will be passed as key, value pairs to the keyword args `**kwargs`.

## Factories
Every entity in the new framework will also be create-able using its corresponding factory `EntityFactory`. Factories can be thought of as objects that carry necessary and sufficient data to satisfy the API call that creates the `Entity`. For example in order to create an account the `AccountFactory` will carry the `firstname, lastname, email, username` of the Account since these are the required arguments to the `createAccount` API.

So the account factory looks as follows:

```python
class AccountFactory(CloudStackBaseFactory):

    FACTORY_FOR = Account.Account

    accounttype = None
    firstname = None
    lastname = None
    email = None
    username = None
    password = None
```

Factories in cloudstack use the [factory_boy](http://factoryboy.readthedocs.org/en/latest/) framework. The factory_boy framework helps cloudstack define complex relationships between its data objects. For eg. In order to create a virtualmachine typically one needs a service offering, a template and a zone present to be able to launch the VM. Factory boy enables traversing these object relationships effectively (top-down, bottom-up) to create those objects. By using factories one can quickly reduce the time to create a testcase for cloudstack.

Factories are optional since not all data can necessarily be provided by them. This way one can use the complete API or choose the factory when the corresponding data object is provided as a factory. For eg: sometimes one may want to pass in specific names to the account for a test. In this case you can always write your own factory OR override the creation of the default factory to provide it the username you desire OR not use factories at all and call the `Account` creation yourself without the use of factories.
