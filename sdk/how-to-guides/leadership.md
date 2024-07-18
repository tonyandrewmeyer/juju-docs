> See first: [Juju | Leader](https://juju.is/docs/juju/leader)

**Contents:**

- [Observe the leader-elected event and define an event handler](#heading--observe-the-leader-elected-event-and-define-an-event-handler)
- [Test leadership management](#heading--test-leadership-management)

<a href="#heading--observe-the-leader-elected-event-and-define-an-event-handler"><h2 id="heading--observe-the-leader-elected-event-and-define-an-event-handler">Observe the leader-elected event and define an event handler</h3></a>

In the `src/charm.py` file, in the `__init__` function of your charm, set up an observer for the leader-elected event and pair that with an event handler. For example:

```
self.framework.observe(self.on.leader_elected, self._on_leader_elected)
```

> See more: [LeaderElectedEvent](https://ops.readthedocs.io/en/latest/#ops.LeaderElectedEvent)

Now, in the body of the charm definition, define the event handler. For example, updating a configuration file:

```
def _on_leader_elected(self, event: ops.LeaderElectedEvent):
    self.reconfigure(leader=self.unit)
```

> Examples: [Tempo reconfiguring ingress on leadership change](https://github.com/canonical/tempo-k8s-operator/blob/3f94027b6173f436968a4736a1f2d89a1f17b2e1/src/charm.py#L263), [Kubeflow Dashboard using a holistic handler to configure on leadership change and other events](https://github.com/canonical/kubeflow-dashboard-operator/blob/02caa736a6ea8986b8cba23b63c08a12aaedb86c/src/charm.py#L82)

Note that there is another leadership event, `leader-settings-changed`, but it
is deprecated. If non-leader units need to be notified about leadership changes,
do that by changing data in a peer relation.

Commonly, other event handlers will need to check for leadership. For example,
only the leader unit can change charm application secrets, so checks for
leadership are needed to guard against non-leaders. For example:

```python
if self.unit.is_leader():
    secret = self.model.get_secret(label="my-label")
    secret.set_content({"username": "user", "password": "pass"})
```

Note that Juju guarantees leadership for only 30 seconds after a `leader-elected`
event or an `is-leader` check. If the charm code may run longer, then extra
`is_leader()` calls should be made to ensure that the unit is still the leader.

<a href="#heading--test-leadership-management"><h2 id="heading--test-leadership-management">Test leadership management</h2></a>

> See first: [Get started with charm testing](/t/6894)

You'll want to add three levels of tests:

- [Write unit tests](#heading--write-unit-tests)
- [Write scenario tests](#heading--write-scenario-tests)
- [Write integration tests](#heading--write-integration-tests)

<a href="#heading--write-unit-tests"><h3 id="heading--write-unit-tests">Write unit tests</h3></a>

> See first: [How to write unit tests for a charm](/t/4461)

When using Harness for unit tests, use the `set_leader()` method to control whether the unit is the leader. For example, to verify that leadership change is handled correctly:

```python
@pytest.fixture()
def harness():
    yield ops.testing.Harness(MyCharm)
    harness.cleanup()


def test_new_leader(harness):
    # Before the test, the unit is not leader.
    harness.set_leader(False)
    harness.begin()
    # Simulate Juju electing the unit as leader.
    harness.set_leader(True)
    # Assert that it was handled correctly.
    assert ...


def test_leader_sets_secrets(harness):
    # The unit is the leader throughout the test, and no leader-elected event
    # is emitted.
    harness.set_leader(True)
    harness.begin()
    secret_id = harness.add_model_secret(APP_NAME, content={"secret": "sssh"})
    harness.update_config(secret_option=secret_id)
    # Assert that the config-changed handler set additional secret metadata:
    assert ...
```

> See more: [`ops.testing.Harness.set_leader`](https://ops.readthedocs.io/en/latest/#ops.testing.Harness.set_leader)

<a href="#heading--write-scenario-tests"><h3 id="heading--write-scenario-tests">Write scenario tests</h3></a>

> See first: [How to write scenario tests for a charm](/t/10585)

When using Scenario for unit tests, pass the leadership status to the `State`. For example:

```python
class MyCharm(ops.CharmBase):
    def __init__(self, framework):
        super().__init__(framework)
        framework.observe(self.on.start, self._on_start)

    def _on_start(self, _):
        if self.unit.is_leader():
            self.unit.status = ops.ActiveStatus('I rule')
        else:
            self.unit.status = ops.ActiveStatus('I am ruled')


@pytest.mark.parametrize('leader', (True, False))
def test_status_leader(leader):
    ctx = scenario.Context(MyCharm, meta={"name": "foo"})
    out = ctx.run('start', scenario.State(leader=leader))
    assert out.unit_status == ops.ActiveStatus('I rule' if leader else 'I am ruled')
```

<a href="#heading--write-integration-tests"><h3 id="heading--write-integration-tests">Write integration tests</h3></a>

> See first: [How to write integration tests for a charm](/t/12734)

Juju is in sole control over which unit is the leader, so leadership changes are
not usually tested with integration tests. If this is required, then the test
needs to remove the leader unit (machine charms) or run `juju_stop_unit` in the
charm container (Kubernetes charms). The test then needs to wait up to 60 seconds
for Juju to elect a new leader.

More commonly, an integration test might want to verify that leader and non-leader behaviour is
as expected. For example:

```python
async def get_leader_unit(ops_test, app, model=None):
    """Utility method to get the current leader unit."""
    leader_unit = None
    if model is None:
        model = ops_test.model
    for unit in model.applications[app].units:
        if await unit.is_leader_from_status():
            leader_unit = unit
            break

    return leader_unit
```

> Examples: [Zookeeper testing upgrades](https://github.com/canonical/zookeeper-operator/blob/106f9c2cd9408a172b0e93f741d8c9f860c4c38e/tests/integration/test_upgrade.py#L22), [postgresql testing password rotation action](https://github.com/canonical/postgresql-k8s-operator/blob/62645caa89fd499c8de9ac3e5e9598b2ed22d619/tests/integration/test_password_rotation.py#L38)

<br>

> <small>**Contributors:**@tmihoc, @tony-meyer</small>