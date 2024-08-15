**Contents:**

- [Set the charm version](#heading--set-the-version-of-the-charm)
- [Test setting the charm version](#heading--test-setting-the-charm-version)
- [Set the workload version](#heading--set-the-version-of-the-workload)
- [Test setting the workload version](#heading--test-setting-the-workload-version)

<a href="#heading--set-the-version-of-the-charm"><h2 id="heading--set-the-version-of-the-charm">Set the charm version</h3></a>

Charms can specify a version of the charm itself, to allow tracking the
installed version of the charm back to the source tree that it was built from.

To set the version, include a file called `version` (no extension) at the top
level of the charm (the same folder as the `charmcraft.yaml` file). The content
of the file is the version string, which is typically a major.minor.patch style
verison or a version control hash identifier.

For example, using the hash of the latest HEAD commit as the version:

```shell
$ git rev-parse HEAD > version
$ ls
lib      src    tox.ini charmcraft.yaml  LICENSE  requirements.txt  tests  version
$ cat version
0522e1fd009dac78adb3d0652d91a1e8ff7982ae
```

<note>
Normally, your publishing workflow would take care of updating this file, so
that it will automatically match the revision that you're publishing.
</note>

Juju admins using your charm can find this information with `juju status` in the
YAML or JSON formats in the `applications.<app name>.charm-version` field. If
there is no version, the key will not be present in the status output.

Note that this is disinct from the charm **revision**, which is set when
uploading a charm to CharmHub (or when deploying/refreshing for local charms).

> See more: [Juju | Revision](https://juju.is/docs/sdk/revision)

<a href="#heading--test-setting-the-charm-version"><h2 id="heading--test-setting-the-charm-version">Test setting the charm version</h2></a>

> See first: [Get started with charm testing](https://juju.is/docs/sdk/get-started-with-charm-testing)

Since the version isn't set by the charm code itself, you'll want to test that
the version is correctly set with an integration test, rather than also having a
unit test.

> See first: [How to write integration tests for a charm](/t/12734)

To verify that setting the charm version works correctly in an integration test, get the status
of the model, and check the `workload_version` attribute of the unit. For example:

```python
async def test_build_and_deploy(ops_test: OpsTest):
    # Build and deploy charm from local source folder
    charm = await ops_test.build_charm(".")

    # Deploy the charm and wait for active/idle status
    await asyncio.gather(
        ops_test.model.deploy(charm, application_name=APP_NAME),
        ops_test.model.wait_for_idle(
            apps=[APP_NAME], status="active", raise_on_blocked=True, timeout=1000
        ),
    )

    # Verify that the charm version has been set.
    status = await ops_test.model.get_status()
    version = status.applications[APP_NAME].charm_version
    expected_version = subprocess.check_output(["git", "rev-parse", "HEAD"]).decode("utf8")
    assert version == expected_version
```

<!---
No "see more" link: this is not currently documented in the pylibjuju docs.
-->

<a href="#heading--set-the-version-of-the-workload"><h2 id="heading--set-the-version-of-the-workload">Set the workload version</h3></a>

Most applications modelled by charms will also have a version. Each application
will have its own versioning scheme, and its own way of accessing that
information. To make things easier for Juju admins, the charm should expose the
workload version through Juju - it's visible in `juju status` in the application
table in the "Version" column in the tabular view, or in the
`applications.<app name>.version` field in JSON or YAML format.

<note>
If the charm has not set the workload version, then the field will not be
present in JSON or YAML format, and if the version string is too long or
contains particular characters then it will not be displayed in the tabular
format.
</note>

The workload version is often set in one of the early event handlers, such as
`start`. If you don't already have a `start` handler, in the `src/charm.py`
file, in the `__init__` function of your charm, set up an observer for the
`start` event and pair that with an event handler. For example:

```python
self.framework.observe(self.on.start, self._on_start)
```

> See more: `[`ops.StartEvent`](https://ops.readthedocs.io/en/latest/#ops.StartEvent)

Now, in the body of the charm definition, define the event handler. Typically,
the workload version is retrieved from the workload itself, with a subprocess
(machine charms) or Pebble exec (Kubernetes charms) call or HTTP request. For
example:

```python
def _on_start(self, event: ops.StartEvent):
    # The workload exposes the version via HTTP at /version
    version = requests.get("http://localhost:8000/version").text
    self.unit.set_workload_version(version)
```

> see more: [`ops.Unit.set_workload_version`](https://ops.readthedocs.io/en/latest/#ops.Unit.set_workload_version)

> Examples: [jenkins-k8s sets the workload version after getting it from the Jenkins package](https://github.com/canonical/jenkins-k8s-operator/blob/29e9b652714bd8314198965c41a60f5755dd381c/src/charm.py#L115), [discourse-k8s sets the workload version after getting it via an exec call](https://github.com/canonical/discourse-k8s-operator/blob/f523b29f909c69da7b9510b581dfcc2309698222/src/charm.py#L581), [synapse sets the workload version after getting it via an API call](https://github.com/canonical/synapse-operator/blob/778bcd414644c922373d542a304be14866835516/src/charm.py#L265)

<a href="#heading--test-setting-the-workload-version"><h2 id="heading--test-setting-the-workload-version">Test setting the workload version</h2></a>

> See first: [Get started with charm testing](https://juju.is/docs/sdk/get-started-with-charm-testing)

You'll want to add three levels of tests:

- [Write unit tests](#heading--write-unit-tests)
- [Write scenario tests](#heading--write-scenario-tests)
- [Write integration tests](#heading--write-integration-tests)

<a href="#heading--write-unit-tests"><h3 id="heading--write-unit-tests">Write unit tests</h3></a>

> See first: [How to write unit tests for a charm](/t/4461)

When using Harness for unit tests, use the `get_workload_version()` method to
get the version that the charm set:

```python
@pytest.fixture()
def harness():
    yield ops.testing.Harness(MyCharm)
    harness.cleanup()

def test_start(harness):
    # Our charm gets the workload version by running a command in the container,
    # so mock that out.
    harness.handle_exec("webserver", ["/bin/server", "--version"], result="1.2\n")
    # begin_with_initial_hooks will trigger the 'start' event, and we expect
    # the charm's 'start' handler to set the workload version.
    harness.begin_with_initial_hooks()
    assert harness.get_workload_version() == "1.2"
```

> Examples: [grafana-k8s checking the workload version](https://github.com/canonical/grafana-k8s-operator/blob/1c80f746f8edeae6fd23ddf31eed45f5b88c06b4/tests/unit/test_charm.py#L283) (and the [earlier mocking](https://github.com/canonical/grafana-k8s-operator/blob/1c80f746f8edeae6fd23ddf31eed45f5b88c06b4/tests/unit/test_charm.py#L127)), [sdcore-webui checks both that the version is set when it is available, and not set when not](https://github.com/canonical/sdcore-webui-k8s-operator/blob/1a66ad3f623d665657d04ad556139439f4733a28/tests/unit/test_charm.py#L447)

> See more: [`ops.testing.Harness.get_workload_version`](https://ops.readthedocs.io/en/latest/#ops.testing.Harness.get_workload_version)

<a href="#heading--write-scenario-tests"><h3 id="heading--write-scenario-tests">Write scenario tests</h3></a>

> See first: [How to write scenario tests for a charm](/t/10585)

When using Scenario for unit tests, retrieve the workload version from the `State`. For example:

```python
def test_workload_version_is_set(leader):
    ctx = scenario.Context(MyCharm, meta={"name": "foo"})
    # Our charm gets the workload version by running a command in the container,
    # so mock that out.
    container = scenario.Container(
        "webserver",
        exec_mock={("/bin/server", "--version"): scenario.ExecOutput(stdout="1.2\n")},
    )
    out = ctx.run('start', scenario.State(containers=[container]))
    assert out.workload_version == "1.2"
```

<a href="#heading--write-integration-tests"><h3 id="heading--write-integration-tests">Write integration tests</h3></a>

> See first: [How to write integration tests for a charm](/t/12734)

To verify that setting the workload version works correctly in an integration test, get the status
of the model, and check the `workload_version` attribute of the unit. For example:

```python
async def test_build_and_deploy(ops_test: OpsTest):
    # Build and deploy charm from local source folder
    charm = await ops_test.build_charm(".")

    # Deploy the charm and wait for active/idle status
    await asyncio.gather(
        ops_test.model.deploy(charm, application_name=APP_NAME),
        ops_test.model.wait_for_idle(
            apps=[APP_NAME], status="active", raise_on_blocked=True, timeout=1000
        ),
    )

    # Verify that the workload version has been set.
    status = await ops_test.model.get_status()
    version = status.applications[APP_NAME].units[f"{APP_NAME}/0"].workload_version
    # We'll need to update this version every time we upgrade to a new workload
    # version. If the workload has an API or some other way of getting the
    # version, the test should get it from there and use that to compare to the
    # unit setting.
    assert version == "3.14"
```

> Examples: [synapse checking that the unit's workload version matches the one reported by the server](https://github.com/canonical/synapse-operator/blob/778bcd414644c922373d542a304be14866835516/tests/integration/test_charm.py#L139)

<!---
No "see more" link: this is not currently documented in the pylibjuju docs.
-->

<br>

> <small>**Contributors:**@tmihoc, @tony-meyer</small>
