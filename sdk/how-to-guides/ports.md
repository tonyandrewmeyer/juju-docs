> See first: [Juju `open-port`](https://juju.is/docs/juju/hook-tool#open-port)

**Contents**:

1. [Open a port based on the charm config](#heading--open-a-port-to-your-charm)
1. [Test the port is opened](#heading--test-the-port-is-opened)

<a href="#heading--heading--open-a-port-to-your-charm"><h2 id="heading--heading--open-a-port-to-your-charm">Open a port based on the charm config</h2></a>

To make a service permanently reachable under a stable URL on the cluster, the charm needs to open (expose) a port. This ensures that the charm will be consistently accessible even if the pod gets recycled and the IP address changes.

The port that the charm's service should be reachable on is typically defined as a config option. As such, all the usual procedure for adding a config option to a charm applies:

1. In your `charmcraft.yaml` define a config that sets the port to be opened/exposed. For example:

```yaml
options:
  config:
    server-port:
      type: int
      description: the port on which to offer the service
      default: 8000
```

2. In your `src/charm.py` observe the `config-changed` event and define a handler. For example:

```python
self.framework.observe(self.on.config_changed, self._on_config_changed)
```

> See more: `[`ops.ConfigChangedEvent`](https://ops.readthedocs.io/en/latest/#ops.ConfigChangedEvent)

Now, in the body of the charm definition, define the event handler, and set the port that should be open - this defaults to a TCP port, but UDP and ICMP can also be specified. For example:

```python
def _on_config_changed(self, event: ops.ConfigChangedEvent):
    self.unit.set_ports(self.config["server-port"])
```

> Examples: [`loki-k8s` sets the open port in the charm `__init__`](https://github.com/canonical/loki-k8s-operator/blob/2e4674d64b692f82ae3dec8f7f1f0745f087827f/src/charm.py#L150), [`mysql-router` sets ports based on whether the charm is configured to be externally accessible](https://github.com/canonical/mysql-router-operator/blob/04c7b3a0ebb6279648bd3f72c5c04b0f7bad1e0c/src/machine_charm.py#L111)

> See more: [How to add a config option to a charm](https://juju.is/docs/sdk/config), [`ops.Unit.set_ports`](https://ops.readthedocs.io/en/latest/#ops.Unit.set_ports)

<a href="#heading--test-the-port-is-opened"><h2 id="heading--test-the-port-is-opened">Test the port is opened</h2></a>

> See first: [Get started with charm testing](https://juju.is/docs/sdk/get-started-with-charm-testing)

You'll want to add three levels of tests: unit, Scenario, and integration. For a charm that exposes the ports to open via config options, the tests are much the same as for testing adding config options, but also verify that the port is opened.

- [Test setting the workload version -- unit tests](#heading--write-unit-tests)
- [Test setting the workload version -- Scenario tests](#heading--write-scenario-tests)
- [Test setting the workload version -- integration tests](#heading--write-integration-tests)

<a href="#heading--write-unit-tests"><h3 id="heading--write-unit-tests">Write unit tests</h3></a>

> See first: [How to write unit tests for a charm](/t/4461)

To use a unit test to verify that updating the charm config opens a port, for this charm that opens a port based on the config, the test needs to trigger the `config-changed` event and then check which ports are open. In your `tests/unit/test_charm.py` file, add the following test function to the file:

```python
def test_port_configuration():
    harness = ops.testing.Harness()
    harness.begin()

    port = 8080
    harness.update_config({"server-port": port})

    assert harness.model.unit.opened_ports() == {ops.Port("tcp", port)}
```

> Examples: [`charm-microk8s` checks that the `install` hook opens a port](https://github.com/canonical/charm-microk8s/blob/1e7fd26e4f85f361a813106325d84bf079cd7b1a/tests/unit/test_charm_control_plane.py#L34)

> See more: [`ops.Unit.opened_ports`](https://ops.readthedocs.io/en/latest/#ops.Unit.opened_ports)

<a href="#heading--write-scenario-tests"><h3 id="heading--write-scenario-tests">Write scenario tests</h3></a>

> See first: [How to write scenario tests for a charm](/t/10585)

To use a Scenario test to verify that the `config-changed` event results in the port being opened, pass the new config to the `State`, and, after running the event, check the `State.opened_ports` attribute. For example, in your `tests/scenario/test_charm.py` file, add the following test function:

```python
def test_open_port():
    ctx = scenario.Context(MyCharm)

    port = 8080
    state_out = ctx.run("config_changed", scenario.State(config={"server-port": port}))

    assert len(state.opened_ports) == 1
    assert state_out.opened_ports[0].port == port
    assert state_out.opened_ports[0].protocol == "tcp"
```

> See more: [`scenario.Ports`](https://github.com/canonical/ops-scenario/?tab=readme-ov-file#ports)

<a href="#heading--write-integration-tests"><h3 id="heading--write-integration-tests">Write integration tests</h3></a>

> See first: [How to write integration tests for a charm](/t/12734)

To verify that the port is opened when the charm is deployed to a Juju model, add an integration test that opens a real connection to the appropriate port. In your `tests/integration/test_charm.py` file, add helper methods that can be used to check if a port is open:

```python
async def get_address(ops_test: OpsTest, app_name, unit_num=0) -> str:
    """Get the address for a the service for an app."""
    status = await ops_test.model.get_status()
    return status["applications"][app_name].get_public_address()

def is_port_open(host: str, port: int, timeout: float=5.0) -> bool:
    """Check if a port is opened on a particular host."""
    try:
        with socket.create_connection((host, port), timeout=timeout):
            # If the connection succeeds, the port is open.
    except (ConnectionRefusedError, TimeoutError):
        # If the connection fails, the port is not open.
        return False
```

> See more: [`Unit.get_public_address`](https://pythonlibjuju.readthedocs.io/en/latest/api/juju.unit.html#juju.unit.Unit.get_public_address)

> See more: [`socket.create_connection`](https://docs.python.org/3/library/socket.html#socket.create_connection)

Now add the test case that will trigger the `config-changed` event that will open the port:

```python
@pytest.mark.abort_on_fail
async def test_open_ports(ops_test: OpsTest):
    app = ops_test.model.applications.get(APP_NAME)

    # Get the service address of the application:
    address = await get_address(ops_test=ops_test, app_name=APP_NAME)
    # Verify that the default port is opened:
    assert is_port_open(address, 8000)

    # Change the config to choose a different port, and verify that it is opened:
    new_port = 8001
    await app.set_config({"server-port": new_port})
    await ops_test.model.wait_for_idle(
        apps=[APP_NAME], status="active", timeout=600
    ),
    assert is_port_open(address, new_port)
```

> Examples: [`loki-k8s` verifies that the rules are externally accessible](https://github.com/canonical/loki-k8s-operator/blob/2e4674d64b692f82ae3dec8f7f1f0745f087827f/tests/integration/helpers.py#L52), [`mysql-router-k8s` verifies that queries can be performed using the external address](https://github.com/canonical/mysql-router-k8s-operator/blob/bd2f317f1ae49df2df03364250e743aff016b45f/tests/integration/helpers.py#L516)

<br>

> <small>**Contributors:**@adithya-raj, @mmkay @ibraaoad, @tmihoc, @tony-meyer</small>
