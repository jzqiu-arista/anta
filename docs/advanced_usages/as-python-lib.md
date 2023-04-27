# How to use ANTA as a Python Library

ANTA has been built to allow user to embeded its tools in your own application. This section describes how you can leverage ANTA modules to help you create your own NRFU solution.

## Inventory Manager

Inventory class is in charge of creating a list of hosts with their information and an eAPI session ready to be consummed. To do that, it connects to all devices to check reachability and ensure eAPI is running.

```python
from anta.inventory import AntaInventory

inventory = AntaInventory(
    inventory_file="inventory.yml",
    username="username",
    password="password",
    enable_password="enable",
    auto_connect=True,
    timeout=1,
)
```

Then it is easy to get all devices or only active devices with the following method:

```python
# print the non reachable devices
for device in inventory.get_inventory(established_only=False):
    if device.established is False:
        print(f"Could not connect to device {device.host}")

# run an EOS commands list on the reachable devices from the inventory
for device in inventory.get_inventory(established_only=True):
    device.session.runCmds(
        1, ["show version", "show ip bgp summary"]
    )
```

You can find data model for [anta.inventory.AntaInventory](./api/inventory.md) in the [auto-generated documentation](./api/inventory.models.md).

??? note "How to create your inventory file"
    Please visit this [dedicated section](./usage-inventory-catalog.md) for how to use inventory and catalog files.


## Use tests from ANTA

All the test functions are based on the exact same input and returns a generic structure with different information.

### Test input

Any test input is based on an `InventoryDevice` object and a list of options. Here is an example to check uptime and check it is higher than `minimum` option.

```python
def verify_uptime(device: InventoryDevice, minimum: int = None) -> TestResult:
```

In general, [`InventoryDevice`](./api/inventory.models.md) is an object created by `AntaInventory`. But it can be manually generated by following required data model.

Here is an example of a list of `InventoryDevice`

```python
[
    {
        "InventoryDevice(host=IPv4Address('192.168.0.17')",
        "username='ansible'",
        "password='ansible'",
        "session=<ServerProxy for ansible:ansible@192.168.0.17/command-api>",
        "url='https://ansible:ansible@192.168.0.17/command-api'",
        "established=True",
        "is_online=True",
        "hw_model=cEOS-LAB",
    },

    {
        "InventoryDevice(host=IPv4Address('192.168.0.2')",
        "username='ansible'",
        "password='ansible'",
        "session=None",
        "url='https://ansible:ansible@192.168.0.2/command-api'",
        "established=False"
        "is_online=False",
        "tags": ['dc1', 'spine', 'pod01'],
        "hw_model=unset",
    }
]
```

### Test output

All tests return a TestResult structure with the following elements:

- `result`: Can be `success`, `skipped`, `failure`, `error` and report result of the test
- `host`: IP address of the tested device
- `test`: Test name runs on `host`
- `message`: Optional message returned by the test.

### Test structure

All tests are based on this structure:

```python
from anta.inventory.models import InventoryDevice
from anta.result_manager.models import TestResult
from anta.test import anta_test

# Use the decorator that wraps the function and inject result argument
@anta_test
async def <test name>(device: InventoryDevice, result: TestResut, <list of args>, minimum: int) -> TestResult:
    """
    dosctring desccription

    Args:
        device (InventoryDevice): InventoryDevice instance containing all devices information.
        result (TestResult): TestResult instance for the test, injected
                             automatically by the anta_test decorator.
        minimum (int): example of test with int parameter

    Returns:
        TestResult instance with
        * result = "unset" if the test has not been executed
        * result = "skipped" if the `minimum` parameter is  missing
        * result = "success" if uptime is greater than minimun
        * result = "failure" otherwise.
        * result = "error" if any exception is caught

    """
    # Test if options are valid (optional)
    if not minimum:
        result.is_skipped("verify_uptime was not run as no minimum were given")
        return result

    # Use await for the remote device call
    response = await device.session.cli(command="show uptime", ofmt="json")
    # Add a debug log entry
    logger.debug(f'query result is: {response}')

    response_data = response["upTime"]
    # Check conditions on response_data
    # ...

    # Return data to caller
    return result
```

## Get test function documentation

Open an interactive python shell and run following commands:

```python
>>> from anta.tests.system import *

>>> help(verify_ntp)

Help on function verify_ntp in module anta.tests.system:

verify_ntp(device: anta.inventory.models.InventoryDevice) -> anta.result_manager.models.TestResult
    Verifies NTP is synchronised.

    Args:
        device (InventoryDevice): InventoryDevice instance containing all devices information.

    Returns:
        TestResult instance with
        * result = "unset" if the test has not been executed
        * result = "success" if synchronized with NTP server
        * result = "failure" otherwise.
        * result = "error" if any exception is caught

>>> exit()
```

If you need to expose test description, you can use this workaround:

```python
from anta.tests.system import *

print(f'{verify_ntp.__doc__.split("\n")[0]}')
```