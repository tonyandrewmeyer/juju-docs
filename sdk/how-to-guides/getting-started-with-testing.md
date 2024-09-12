Testing charm code is an essential part of charming. Here we will see how to get started with it. We will look at the templates we have available and the frameworks we can use to write good unit, integration, and functional tests.


Prerequisites:
- knowledge of testing in general
- knowledge of juju and charms
- knowledge of the juju models and events, esp. the data involved in a charm's lifecycle (e.g. see [Talking to a workload control flow from A to Z](/t/6161)

What you will learn:
- What are the starting points for adding tests to a charm?
- What do you typically want to test in a charm?
- How can you do that?
  - What can you unit-test?
     - How to effectively use the Harness.
  - What can you only integration-test?
     - What integration testing frameworks are there?
  - Where can you apply functional testing?
- How to automate this in a CI pipeline.


# Templates

The two most popular ways to set up a charm project are via `charmcraft init` and by using the [`template-operator`](https://github.com/canonical/template-operator) template repo. 
[note]
Currently, so-called charmcraft profiles are being developed -- these will consolidate some standard templates and make them available via `charmcraft init --profile <profile name>`.
[/note]

## Charmcraft
The template you get with `charmcraft init` is rather compact. The testing infrastructure consists of a `tests/test_charm.py` file, and a `run_tests` shell script that contains:
```bash
                                                      
if [ -z "$VIRTUAL_ENV" -a -d venv/ ]; then            
    . venv/bin/activate                               
fi                                                    
                                                      
if [ -z "$PYTHONPATH" ]; then                         
    export PYTHONPATH="lib:src"                       
else                                                  
    export PYTHONPATH="lib:src:$PYTHONPATH"           
fi                                                    
                                                      
flake8                                                
coverage run --branch --source=src -m unittest -v "$@"
coverage report -m                                    
```
In words, this will run:
 - `flake8` linter checks.
 - `unittest` with coverage in `/tests/`.

## Template-operator
The `template-operator` template provides a more robust starting point for most charming needs. 
It comes with:
 - example unit tests (`/tests/unit/test_charm.py`) 
 - example integration tests (`/tests/integration/test_charm.py`)
 - instead of a `run_tests` script, a `tox.ini` file that contains pre-set environments for:
    - formatting (`black`)
    - linting (`flake8`, `isort`)
    - unit (`pytest`)
    - integration (`pytest`, `pytest-operator`)

So `template-operator` contains all of `charmcraft init`'s template tooling, and more. It also contains github actions specifications that will configure GH to run the whole suite on each PR to `master/main` (more on that later).

[note]`template-operator` is in practice a k8s charm template. Its example code is that of a kubernetes charm, and its CI will run the integration tests on `microk8s`. If you're developing a machine charm, you are going to have to change all that.[/note]

In the rest of this tutorial we will assume that our project is set up with `template-operator`.

# Unit Testing


## A charm as an input -> output function
In production, a charm is a stateless (modulo stored state) object that comes to life when the juju agent decides to execute it with a given context (we call that an `event`). 
The "inputs" of a charm run are therefore:

 - the event context
 - charm configuration
 - relation data
 - stored state

Only the event context is guaranteed to be present. The other input sources are optional, but typically a charm will have at least some config and a few integrations adding to its inputs.

The charm code executes and typically produces side-effects aimed at its workload (i.e. it writes files to a disk, runs commands on a system, or reconfigures a process) or at other charms it integrates with (i.e. it writes relation data). We call this 'operating' a workload, and that is what a charm is meant to do.
The ways in which a charm operates can be roughly categorized as:
 
- system operations (e.g. kill a process, restart a service, write a file, emit to a syslog server, make a HTTP request)
- cloud operations (e.g. deploy a kubernetes service, launch a VM)
- workload operations (e.g. send a request to a local server, write a config file)
- juju operations (write relation data)

If the charm is a machine charm, workload operation calls can be done directly, while if we're talking about a k8s charm, they will likely be mediated by [Pebble](https://github.com/canonical/pebble).

"juju operations" are the most 'meta' of them all: they do not affect the workload in and of itself, but they share data which is meant to affect the operation of *other* charms that this charm is related to.

> For the bigger picture of a charm's lifecycle, you can read [this](/t/6161) and [that](/t/5938).

## What we are testing when we unit-test.  
A 'live', deployed juju application will have access to all the inputs we discussed above, including environment variables, host system access, et cetera. Unittests will typically want to mock all that and focus on mapping inputs to expected outputs. Any combination of the input sources we mentioned above can result in any combination of operations.
A few examples of increasing complexity of scenarios we may want to unittest:

 - if this event occurs, assert that the charm emits that system call
 - if this event occurs, given this config, assert that the charm writes to the filesystem a config file with this expected content
 - if this event occurs, given this relation data and that config value, assert that that system call is made and this relation data is written (to another relation)
 
> You will notice that the starting point is typically always an event. A charm doesn't do anything unless it's being run, and it is only run when an event occurs. So there is *always* an event context to be mocked. This has important consequences for the unit-testing framework, as we will see below.

## The harness.  
In the charming world, unit testing means using the [Harness](https://ops.readthedocs.io/en/latest/#ops.testing.Harness).

The harness is the 'mocker' for most inputs and outputs you will need. Where a live charm would gather its input through context vars and calls to the juju api (by running the hook tools), a charm under unit-test will gather data via a mocked backend managed by the harness. Where a live charm would produce output by writing files to a filesystem, the harness exposes a mock filesystem the charm will be able to interact with without knowing the difference. More specific outputs, however, will be need to be mocked individually

A typical test with the harness will look like this:
 
- set up:
  - set up the charm and its metadata
  - set up the harness
  - mock any 'output' callable that you know would misfire or break (e.g. a system call -- you don't want a unittest to reboot your laptop)
  - configure the charm
  - mock any relation data
 - **mock an event**
 - get the output
 - run assertions on the output

> Obviously, other flows are possible, e.g. where you unittest individual charm methods without going through the whole event context setup, but this is the characteristic one.

### Understanding the harness
When you instantiate a `Harness` object, the charm instance does not exist yet. 
Just like in a live charm, it is possible that when the charm is executed for the first time, the juju model already has given it storage, relations, some config, or leadership. This delay is meant to give us a chance to simulate this in our test setup. You create the harness, then you prepare the 'initial state' of the model mock, then you finally initialize the charm and simulate one or more events.

There are two ways to initialize a harnessed charm.
When a charm is deployed, it goes through the Setup phase (for a description of the phases, see [a charm's life](5938)). That is a fixed sequence of events. The harness has a method, `begin_with_initial_hooks()`, that runs said sequence.
Alternatively, you can initialize the charm by calling `begin()`. This will instantiate the charm without firing any Setup phase event.

After the Setup phase, the charm goes into Operation. To test operation-phase-related events, the harness provides some methods to simulate the most common scenarios. For example: 
 
 - the cloud admin changes the charm config: `harness.update_config`
 - the cloud admin relates this charm to some other: `harness.add_relation`
 - a remote unit joins in a relation (e.g. because the cloud admin has scaled up a remote charm): `harness.add_relation_unit`
 - a remote unit touches its relation data: `harness.update_relation_data`
 - the cloud admin removes a relation: `harness.remove_relation`
 - a resource is attached/detached: `harness.attach_storage`/`harness.detach_storage`
 - a container becomes ready: `harness.container_pebble_ready`

Therefore, one typically will not have to manually `.emit()` events, but can rely on the harness' utilities and focus on the higher level abstractions that they expose.

#### A note on the Harness' double role: realistic backend mock and allmighty simulator
The Harness plays a double role in unittests, one that can cause some confusion to starters. On the one hand, the Harness provides a mock backend for the charm to talk with. If the charm-under-test wants to know what data is in a relation's databag, it will ask the Harness for it. Since a real, deployed charm has to play by juju's rules and cannot, for example, write a remote unit's databag, if the charm attempts to do so via the Harness, the Harness will not allow it and raise the same error the charm would meet when deployed.

If you, as tester, want to mock the remote unit's databag, so that when the charm attempts to *read* it, it will give some data back, then we want the Harness to not fail and tell you the remote databag is read-only.
This is achieved by checking, from `ops` code, of whether we are in an event context or not.

When a live charm executes it is, invariably, because an event has occurred. A harnessed charm, however, can be kept alive while multiple events are ran on it, its methods are called in arbitrary orders, and the object is otherwise freely manipulated. The harness achieves event-runtime behaviour consistency with a 'live' charm by controlling the event context.

- When the event context is unset, the harness operates in '**allmighty**' mode and we assume that everything is allowed: all databags can be read/written at will.
- When the event context is set, the harness operates in '**strict**' mode and the backend will behave in a realistic way, raising errors on forbidden calls.

Therefore, if the Harness knows that an event is being fired, it will be able to set the strict mode and the code will be validated as it should. But if you bypass the Harness by manually calling an event handler, for example, then the code will run with superpowers and the test may miss some bugs.


```python
class MyCharm(CharmBase):
     def _on_event(self, event):
         """This event handler attempts to write a remote app's databag and should therefore fail."""
         relation = self.get_relation()
         relation.data[relation.app]['foo'] = 'bar'
         
def test_allmighty():
     harness = Harness(MyCharm)
     harness.begin()
     harness.charm._on_event(None)  # no exception raised
     
def test_strict():
    harness = Harness(MyCharm)
    harness.begin()
    with pytest.raises(Exception):
        harness.charm.on.event.emit()   # will raise an exception

    with pytest.raises(Exception):
        # The same will happen if instead of emitting the event directly we 'cause' it to be emitted by the harness. E.g:
        harness.begin_with_initial_hooks()  
        # or whatever we need to do to trigger `event`
```

> We are currently working on how to expose these knobs to the test environment, so that one can tell the Harness to temporarily pretend we are (or we are not) in a certain event context. [see this](https://github.com/canonical/operator/pull/795)

### Writing a test

The typical way in which we want to structure a test is:
 - configure the required inputs
 - mock any function or system call you need to
 - initialize the charm
 - fire some event OR use one of the harness methods to trigger a predefined event sequence
 - assert some output matches what is expected, or some function is called with the expected parameters, etc...

This is how a simple test might look like:
```python
from charm import MyCharm
from ops.testing import Harness

def test_pebble_ready_writes_config_file():
    """Test that on pebble-ready, a config file is written"""
    harness: Harness[MyCharm] = Harness(MyCharm)
    # If you want to mock charm config:
    harness.update_config({'foo': 'bar'})
    # if you want to mock charm leadership:
    harness.set_leader(True)

    # if you want to mock relation data:
    relation_ID = harness.add_relation('relation-name', 'remote-app-name')
    harness.add_relation_unit(relation_ID, 'remote-app-name/0')
    harness.update_relation_data(relation_ID, 'remote-app-name/0', {'baz': 'qux'})
    
    # we are done setting up the inputs.

    harness.begin()
    charm = harness.charm  # this is a MyCharm instance
    
    # this will fire a `workload-pebble-ready` event.
    harness.container_pebble_ready("workload")

    # suppose that MyCharm has dropped a YAML config file via Pebble.push()
    container = charm.unit.get_container("workload")
    file = "/opt/workload/path_to_config_file.yaml"
    config = yaml.safe_load(container.pull(file).read())
    assert config[0]['foo']['bar'] == 'baz'  # or whatever
``` 

[note] 
Don't forget, in the setup part, to specify the charm's metadata. A charm's available configuration depends on its `config.yaml` metadata file, and the relations it can have depend on its `metadata.yaml` file, and so on. 
You need to pass yaml-encoded strings containing the necessary data to the `Harness` constructor in order for the harness to be aware of what config is possible and which relations and actions are available.
[/note]

[note]
An important difference between a harnessed charm  and a 'live', deployed charm is that the harness holds on to the charm instance between events, while a deployed charm garbage-collects the charm instance between hooks.
So if your charm were to set some states in, say, instance attributes, and rely on it on subsequent event handling loops, the unit tests based on the harness would not be able to catch that mistake. An integration test would.
[/note]

# Integration Testing
Where unit testing focuses on black-box method-by-method verification, integration testing focuses on the big picture. Typically integration tests check that the charm does not break (i.e. blocks with status `blocked/error`) when a (mocked) cloud admin performs certain operations. These operations are scripted by using, in order of abstraction:
 -  shell commands against [the `juju` cli](https://juju.is/docs/olm/juju-cli-commands)
 - [`python-libjuju`](https://github.com/juju/python-libjuju), wrapping juju api calls
   - [`pytest-operator`](https://github.com/charmed-kubernetes/pytest-operator), a `pytest` plugin wrapping `python-libjuju`
   - [`zaza`](https://zaza.readthedocs.io/en/latest/index.html), a testing-framework-agnostic wrapper on top of `python-libjuju` 

Things you typically want to test with integration tests:
 - The charm can be packed (i.e. `charmcraft pack` does not error)
 - The charm can be deployed (i.e. `juju deploy ./packed_charm.charm` deploys an application that reaches `active` or `waiting` within a reasonable time frame)

These are 'smoke tests' that should always be present. The following are non-smokey, proper integration tests.
- The charm can be related to other applications without erroring
  - and the relation has the expected effect on the charm's operation logic
- The charm can be configured
  - and the config has the expected effect on the charm's operation logic
- The actions supported by the charm can be executed
  - and return the expected results
- Given any combination of the above relations, configs, etc, the charm 'works', i.e. the workload it operates does whatever it is supposed to do.

## Testing with `pytest-operator`
In `template-operator`, the `tox.ini` file contains the following lines:
```shell
[testenv:integration]
description = Run integration tests
deps =
    pytest
    juju
    pytest-operator>=0.13.0
    -r{toxinidir}/requirements.txt
commands =
    pytest -v --tb native --ignore={[vars]tst_path}unit --log-cli-level=INFO -s {posargs}
```
So you can run `tox -e integration` and the integration tests will run, provided that you have a cloud available on your local juju client. In other words, you should be able to open a shell and execute `juju add-model foo`.

The entry point for all `pytest-operator` tests is the `ops_test` fixture.
The fixture is a module-scoped context which, on entry, adds to juju a randomly-named new model and destroys it on exit.
All tests in that module, and all interactions with the `ops_test` object, will take place against that model.

[note]
If you run `tox -e integration --model foo`, `ops_test` will not generate a random new model, but instead will connect to an existing model called `foo` and the tests will be run there. This means that multiple modules will all be executed in the same model, which can give headaches in CIs, where the `--model` argument is often used to provide a predictable model name for logging collection purposes. This is the case for example in `template-operator`.
[/note]

> *Pro tip*: you can prevent `ops_test` from tearing down the model on exit by passing the `--keep-models` argument. This is useful when the tests fail and the logs don't provide a sufficient post-mortem and a real live autopsy is required.

Detailed documentation of how to use `ops_test` and `pytest-operator` is out of scope for this document. However, this is how a typical integration test could look like (yes, `pytest-operator` has an async API):

```python
async def test_operation(ops_test: OpsTest):
    # tweak the config
    app: Application = ops_test.model.applications.get("tester")
    await app.set_config({"my-key": "my-value"})
    
    # add another charm and relate them
    await ops_test.model.deploy('other-app')
    await ops_test.model.relate('tester:endpoint1', 'other-charm:endpoint2')
    
    # scale it up
    await app.add_unit(2)
    
    # remove another app
    await ops_test.model.remove_application('yet-another-app')
    
    # run an action on a unit
    unit: Unit = app.units[1]
    action = await unit.run('my-action')
    assert action.results == <foo>
    
    # what this means depends on the workload
    assert charm_operates_correctly()  
```

`ops_test` has, of course, API for all inverse operations: remove an app, scale it down, remove a relation...

A good integration testing suite will check that the charm continues to operate as expected whenever possible, by combining these simple elements.


# Functional Testing
Some charms represent their workload by means of an object-oriented wrapper, which mediates between operator code and the implementation of operation logic.
In such cases, it can be useful to add a third category of tests, namely functional tests, that black-box test that workload wrapper without worrying about the substrate it runs on (the charm, the cloud, the machine or pod...).
For an example charm adopting this strategy, see [parca-operator](https://github.com/jnsgruk/parca-operator).


Nowadays, the preferred tool to do functional testing is [scenario](https://github.com/canonical/ops-scenario).
See [this tutorial](/t/10585) for a good start.

# CI

Typically, you want the tests to be ran automatically against any PR into your repository's main branch, and sometimes, to trigger a new release whenever that succeeds. CD is out of scope for this article, but we will look at how to set up a basic CI.

`template-operator` once again has the essential plumbing already in place. 

Let's look at `.github/workflows/ci.yaml`, it contains three jobs: `lint`, `unit-test` and `integration-test-microk8s`. 
```yaml
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Install dependencies
        run: python3 -m pip install tox
      - name: Run linters
        run: tox -e lint
```
`unit-test` is very similar, and both are rather self-explanatory. They work closely together with `tox.ini`, which does most of the job of specifying what it means to run a given test suite.

`integration-test-microk8s` is a bit more complex, because in order to run itests we need to deploy a kubernetes cluster.
That is achieved by this step:

```yaml
      - name: Setup operator environment
        uses: charmed-kubernetes/actions-operator@main
        with:
          provider: microk8s
```
This will install microk8s on the test runner and `pytest-operator` will be able to connect to it.

> You can find more actions, advanced documentation and use cases in [charming-actions](https://github.com/canonical/charming-actions)

Secondly, the test runner step passes `--model testing` to `tox` to, as we mentioned above, later be able to use the logdump action on the model to gather debug logs.
```yaml
      - name: Run integration tests
        run: tox -e integration -- --model testing
      - name: Dump logs
        uses: canonical/charm-logdump-action@main
        if: failure()
        with:
          app: <name of the app being tested>
          model: testing
```

# Conclusion
We have examined all angles one might take when testing a charm, and given a brief overview of the most popular frameworks for implementing unit and integration tests, all the way to how one would link them up with a CI system to make sure the repository remains clean and tested.
