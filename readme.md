
<!-- Install doctoc with `npm install -g doctoc`  then `doctoc README.md --github` -->

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Overview](#overview)
- [Reader Prerequisites](#reader-prerequisites)
- [Setup](#setup)
- [Usage](#usage)
  - [Boilerplate](#boilerplate)
  - [Introduction to Selectors](#introduction-to-selectors)
  - [Accessing API objects as Python Objects](#accessing-api-objects-as-python-objects)
  - [Making changes to APIObjects](#making-changes-to-apiobjects)
  - [Other examples:](#other-examples)
- [Environment Variables](#environment-variables)
  - [Defaults when invoking `oc`](#defaults-when-invoking-oc)
  - [Master timeout](#master-timeout)
  - [SSH Client Host](#ssh-client-host)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview
The [openshift-python](https://www.github.com/jupierce/openshift-python) library aims to provide a readable, concise, comprehensive, and fluent
API for rich interactions with an [OpenShift](https://www.openshift.com) cluster. Unlike other clients, this library exclusively uses the command
line tool (oc) to achieve the interactions. This approach comes with important benefits and disadvantages when compared
to other client libraries.

Pros:
- No additional software needs to be installed on the cluster. If oc is available on a box you can ssh to, you are ready to use
this API.
- Virtually always up-to-date. The version of oc installed on a cluster is always compatible with the version that cluster
is running. We say "virtually" here because it is possible that oc may deprecate or dramatically change aspects of the
command-line interface. Thankfully, such changes are extremely rare.
- No special cases for dynamic / custom resources. Every resource is treated as dynamic.
- Low learning curve. If you can use oc, you can understand this library quickly.

Cons:
- This API is not intended to implement something as complex as a controller. For example, we haven't even implemented
watch functionality. This library is intended to dramatically simplify CLI interactions. If can't imagine accomplishing
the CLI for your use case, this API is probably not for you.
- If you care about whether a REST API returns a particular error code, this API is probably not for you. Since it
is based on the CLI, high level return codes are used to determine success or failure.

## Reader Prerequisites
* Familiarity with OpenShift [command line interface](https://docs.openshift.org/latest/cli_reference/basic_cli_operations.html)
is highly encouraged before exploring the API's features. The API leverages the [oc](https://docs.openshift.org/latest/cli_reference/index.html)
binary and, in many cases, passes method arguments directly on to the command line. This document cannot, therefore,
provide a complete description of all possible OpenShift interactions -- the user may need to reference
the CLI documentation to find the pass-through arguments a given interaction requires.

* A familiarity with Python is recommended.

## Setup
1. Git clone https://github.com/jupierce/openshift-python.git (or your fork).
2. Append ./packages to your PYTHONPATH environment variable (e.g. export PYTHONPATH=$(pwd)/packages:$PYTHONPATH).

## Usage

### Boilerplate
Any standard Python application should be able to use the API if it imports the openshift package. The simplest
possible way to begin using the API is login to your target cluster before running your first application.

For example:
1. oc cluster up
2. oc login -u system:admin

Can you run `oc get project` successfully from the command line? Then write your app!

```python
#!/usr/bin/python
import openshift as oc

print('Current project: {}'.format(oc.get_project_name()))
```

This approach also works if you are using the library within a Pod container -- the `oc` binary automatically
detects it is running in a container and the injected serviceaccount token/cacert.

It is best practice with the library to setup a tracking context for your application so that
you will be able to easily analyze what 'oc' invocations where made on your behalf and the result
of those operations. *Note that details about all 'oc' invocations performed within the context will
be stored within the tracker. Therefore, do not use a single tracker for a continuously running
process -- it will consume memory for every oc invocation.*

```python
with oc.tracker() as tracker:
    try:
        print('Current project: {}'.format(oc.get_project_name()))
        print('Current user: {}'.format(oc.whoami()))
    except:
        print('Error acquire details about project/user')
    
    # Print out details about the invocations made within this context.
    print tracker.get_result()
```

In this case, the tracker output would look something like:
```json
{
    "status": 0, 
    "operation": "tracker", 
    "actions": [
        {
            "status": 0, 
            "verb": "project", 
            "references": {}, 
            "in": null, 
            "out": "aos-cd\n", 
            "err": "", 
            "cmd": [
                "oc", 
                "project", 
                "-q"
            ], 
            "elapsed_time": 0.15344810485839844, 
            "internal": false, 
            "timeout": false, 
            "last_attempt": true
        }, 
        {
            "status": 0, 
            "verb": "whoami", 
            "references": {}, 
            "in": null, 
            "out": "aos-ci-jenkins\n", 
            "err": "", 
            "cmd": [
                "oc", 
                "whoami"
            ], 
            "elapsed_time": 0.6328380107879639, 
            "internal": false, 
            "timeout": false, 
            "last_attempt": true
        }
    ]
}
```

If you are unable to use a KUBECONFIG environment variable or need fine grained control over the 
server/credentials you communicate with for each invocation, use openshift-python contexts. 
Contexts can be nested and cause oc invocations within them to use the most recently established 
context information.

```python
with oc.api_server('https:///....'):  # use the specified api server for nested oc invocations.
    
    with oc.token('abc..'):  # --server=... --token=abc... will be included in inner oc invocations.
        print "Current project: " + oc.get_project_name()
    
    with oc.token('def..'):  # --server=... --token=def... will be included in inner oc invocations.
        print "Current project: " + oc.get_project_name()
```

Is your oc binary on a remote host? No problem. Easily remote all CLI interactions over SSH using the client_host
context. Before running this command, you will need to load your ssh agent up with a key
appropriate to the target client host.

```python
with openshift.client_host(hostname="my.cluster.com", username="root", auto_add_host=True):
    # oc invocations will take place on my.cluster.com host as the root user.
    print "Current project: " + oc.get_project_name()
```

Using this model, your Python script will run exactly where you launch it, but all oc invocations will
occur on the remote host.

### Introduction to Selectors
Selectors are a central concept used by the API to interact with collections
of OpenShift resources. As the name implies, a "selector" selects zero or
more resources on a server which satisfy a user specified criteria. An apt
metaphor for a selector might be a prepared SQL statement which can be
used again and again to select rows from a database.

```python
# Create a selector which selects all projects.
project_selector = oc.selector("projects")

# Print the qualified name (i.e. "kind/name") of each resource selected.
print "Project names: " + str(project_selector.names())

# Count the number of projects on the server.
print "Number of projects: " + str(project_selector.count_existing())

# Selectors can also be created with a list of names.
sa_selector = oc.selector(["serviceaccount/deployer", "serviceaccount/builder"])

# Performing an operation will act on all selected resources. In this case,
# both serviceaccounts are labeled.
sa_selector.label({"mylabel" : "myvalue"})

# Selectors can also select based on kind and labels.
sa_label_selector = oc.selector("sa", labels={"mylabel":"myvalue"})

# We should find the service accounts we just labeled.
print "Found labeled serviceaccounts: " + str(sa_label_selector.names())
```

The output should look something like this:

```
Project names: [u'projects/default', u'projects/kube-system', u'projects/myproject', u'projects/openshift', u'projects/openshift-infra', u'projects/temp-1495937701365', u'projects/temp-1495937860505', u'projects/temp-1495937908009']
Number of projects: 8
Found labeled serviceaccounts: [u'serviceaccounts/builder', u'serviceaccounts/deployer']
```

### Accessing API objects as Python Objects

Selectors allow you to perform "verb" level operations on a set of objects, but
what if you want to interact objects at a schema level?

```python
projects_sel = oc.selector("projects")

# .objects() will perform the selection and return a list of APIObjects
# which model the selected resources.
projects = projects_sel.objects()

print "Selected " + str(len(projects)) + " projects"

# Let's store one of the project APIObjects for easy access.
project = projects[0]

# The APIObject exposes methods providing simple access to metadata and common operations.
print( 'The project is: {}/{}'.format(project.kind(), project.name()) )
project.label({ 'mylabel': 'myvalue' })

# And the APIObject allow you to interact with an object's data via the 'model' attribute.
# The model is similar to a standard dict, but also allows dot notation to access elements
# of the structured data.
print( 'Annotations:\n{}\n.format( project.model.metadata.annotations ) )

# There is no need to perform the verbose 'in' checking you may be familiar with when
# exploring the model object. Accessing the model will always return a value. If the
# accessed field does not have a corresponding value in the underlying model, the
# singleton 'Missing' will be returned.

if project.model.metadata.annotations.myannotation is Missing:
    print("This object has not been annotated yet")

# If a field in the model contains special characters, use standard Python notation
# to access the key instead of dot notation.
if project.model.metadata.annotations['my_annotation]' is Missing:
    print("This object has not been annotated yet")

# For debugging, you can always see the state of the underlying model by printing the
# APIObject as JSON.
print('{}'.format(project.as_json())

# Or getting deep copy dict. Changes made to this dict will not affect the APIObject.
d = project.as_dict()

# Model objects also simplify looking through kubernetes style lists. For example, can_match
# returns True if the modeled list contains an object with the subset of attributes specified.
# If this example, we are checking if the a node's kubelet is reporting Ready:
oc.selector('node/alpha').object().model.status.conditions.can_match(
    {
        'type': 'Ready',
        'status': "True",
    }
)
```


### Making changes to APIObjects
```python
# APIObject exposes simple interfaces to delete and patch the resource it represents.
# But, more interestingly, you can make detailed changes to the model and apply those
# changes to the API.

project.model.metadata.labels['my_label'] = 'myvalue'
project.apply()

# If modifying the underlying API resources could be contentious, use the more robust
# modify_and_apply method which can retry the operation multiple times -- refreshing
# with the current object state between failures.

# First, define a function that will make changes to the model.
def make_model_change(apiobj):
    apiobj.model.data['somefile.yaml'] = 'wyxz'
    return True

# modify_and_apply will call the function and attempt to apply its changes to the model
# if it returns True. If the apply is rejected by the API, the function will pull
# the latest object content, call make_model_change again, and try the apply again
# up to the specified retry account.
configmap.modify_and_apply(make_model_change, retries=5)

```

### Other examples:

Running oc exec on a pod.
```python
    result = oc.selector('pod/alertmanager-main-0').object().execute(['cat'],
                                                                     container_name='alertmanager',
                                                                     stdin='stdin for cat')
    print(result.out())
```

## Environment Variables
To allow openshift-python applications to be portable between environments without needing to be modified, 
you can specify many default contexts in the environment. 

### Defaults when invoking `oc`
Establishing explicit contexts within an application will override these environment defaults.
- `OPENSHIFT_PYTHON_DEFAULT_OC_PATH` - default path to use when invoking `oc`
- `OPENSHIFT_PYTHON_DEFAULT_CONFIG_PATH` - default `--config` argument
- `OPENSHIFT_PYTHON_DEFAULT_API_SERVER` - default `--server` argument
- `OPENSHIFT_PYTHON_DEFAULT_CA_CERT_PATH` - default `--cacert` argument
- `OPENSHIFT_PYTHON_DEFAULT_PROJECT` - default `--namespace` argument
- `OPENSHIFT_PYTHON_DEFAULT_OC_LOGLEVEL` - default `--loglevel`
- `OPENSHIFT_PYTHON_DEFAULT_TLS_CHECK` - If `false`, always include `--insecure-skip-tls-verify`

### Master timeout
Defines an implicit outer timeout(..) context for the entire application. This allows you to ensure
that an application terminates within a reasonable time, even if the author of the application has
not included explicit timeout contexts. Like any `timeout` context, this value is not overridden
by subsequent `timeout` contexts within the application. It provides an upper bound for the entire
application's oc interactions.

- `OPENSHIFT_PYTHON_MASTER_TIMEOUT` 

### SSH Client Host
In some cases, it is desirable to run an openshift-python application using a local `oc` binary and 
in other cases, the `oc` binary resides on a remote client. Encoding this decision in the application
itself is unnecessary.

Simply wrap you application in a `client_host` context without arguments. This will try to pull 
client host information from environment variables if they are present. If they are not present,
the application will execute on the local host.

For example, the following application will ssh to `OPENSHIFT_PYTHON_DEFAULT_SSH_HOSTNAME` if it is defined
in the environment. Otherwise, `oc` interactions will be executed on the host running the python application.

```python
with oc.client_host():  # if OPENSHIFT_PYTHON_DEFAULT_SSH_HOSTNAME if not defined in the environment, this is a no-op
    print( 'Found nodes: {}'.format(oc.selector('nodes').qnames()) ) 
```

- `OPENSHIFT_PYTHON_DEFAULT_SSH_HOSTNAME` - The hostname on which the `oc` binary resides
- `OPENSHIFT_PYTHON_DEFAULT_SSH_USERNAME` - Username to use for the ssh connection (optional)
- `OPENSHIFT_PYTHON_DEFAULT_SSH_PORT` - SSH port to use (optional; defaults to 22)
- `OPENSHIFT_PYTHON_DEFAULT_SSH_AUTO_ADD` - Whether to automatically trust news hosts