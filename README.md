<!-- NOTE: This file is generated from skewer.yaml.  Do not edit it directly. -->

# Skupper Hello World using Ansible

[![main](https://github.com/ssorj/skupper-example-ansible/actions/workflows/main.yaml/badge.svg)](https://github.com/ssorj/skupper-example-ansible/actions/workflows/main.yaml)

This example is part of a [suite of examples][examples] showing the
different ways you can use [Skupper][website] to connect services
across cloud providers, data centers, and edge sites.

[website]: https://skupper.io/
[examples]: https://skupper.io/examples/index.html

#### Contents

* [Prerequisites](#prerequisites)
* [Step 1: Install the Skupper command-line tool](#step-1-install-the-skupper-command-line-tool)
* [Step 2: Install the Skupper Ansible collection](#step-2-install-the-skupper-ansible-collection)
* [Step 3: Set up your clusters](#step-3-set-up-your-clusters)
* [Step 4: Inspect the inventory file](#step-4-inspect-the-inventory-file)
* [Step 5: Run the setup playbook](#step-5-run-the-setup-playbook)
* [Step 6: Access the frontend](#step-6-access-the-frontend)
* [Step 7: Run the teardown playbook](#step-7-run-the-teardown-playbook)
* [Next steps](#next-steps)
* [About this example](#about-this-example)

## Prerequisites

* The `kubectl` command-line tool, version 1.15 or later
  ([installation guide][install-kubectl])

* Access to at least one Kubernetes cluster, from [any provider you
  choose][kube-providers]

[install-kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[kube-providers]: https://skupper.io/start/kubernetes.html

## Step 1: Install the Skupper command-line tool

This example uses the Skupper command-line tool to deploy Skupper.
You need to install the `skupper` command only once for each
development environment.

On Linux or Mac, you can use the install script (inspect it
[here][install-script]) to download and extract the command:

~~~ shell
curl https://skupper.io/install.sh | sh
~~~

The script installs the command under your home directory.  It
prompts you to add the command to your path if necessary.

For Windows and other installation options, see [Installing
Skupper][install-docs].

[install-script]: https://github.com/skupperproject/skupper-website/blob/main/input/install.sh
[install-docs]: https://skupper.io/install/

## Step 2: Install the Skupper Ansible collection

Use the `ansible-galaxy` command to install the
`skupper.network` collection.

_**Terminal:**_

~~~ shell
ansible-galaxy collection install skupper.network
~~~

## Step 3: Set up your clusters

This example uses two clusters.  The clusters are accessed using
two kubeconfig files:

~~~
<project-dir>/ansible/kubeconfigs/east
<project-dir>/ansible/kubeconfigs/west
~~~

For each kubeconfig, set the `KUBECONFIG` environment variable
to the file path and run the login command for your cluster.
This updates the kubeconfig with the required credentials.

**Note:** The cluster login procedure varies by provider.  See
the documentation for yours:

* [Minikube](https://skupper.io/start/minikube.html#cluster-access)
* [Amazon Elastic Kubernetes Service (EKS)](https://skupper.io/start/eks.html#cluster-access)
* [Azure Kubernetes Service (AKS)](https://skupper.io/start/aks.html#cluster-access)
* [Google Kubernetes Engine (GKE)](https://skupper.io/start/gke.html#cluster-access)
* [IBM Kubernetes Service](https://skupper.io/start/ibmks.html#cluster-access)
* [OpenShift](https://skupper.io/start/openshift.html#cluster-access)

_**Terminal:**_

~~~ shell
export KUBECONFIG=<project-dir>/ansible/kubeconfigs/east
# Enter your provider-specific login command for cluster 1
export KUBECONFIG=<project-dir>/ansible/kubeconfigs/west
# Enter your provider-specific login command for cluster 2
~~~

## Step 4: Inspect the inventory file

Before we start running commands, let's examine the inventory
file.  It is here that we can define Skupper sites, links, and
exposed services.

[ansible/inventory.yml](ansible/inventory.yml):

~~~ yaml
all:
  vars:
    ansible_connection: local
  hosts:
    east:
      kubeconfig: "{{ inventory_dir }}/kubeconfigs/east"
      namespace: east
      links:
        - host: west
      services:
        backend:
          ports:
            - 8080
          targets:
            - type: deployment
              name: backend
    west:
      kubeconfig: "{{ inventory_dir }}/kubeconfigs/west"
      namespace: west
~~~

Our example has two sites, East and West, enumerated under
`hosts`.

The `links` attribute on host `east` defines a link from East to West.

The `services` attribute on host `east` exposes the backend on
East so the frontend in West can access it.

The playbooks that follow use this inventory data to set up and
tear down the Skupper network.

For more information about inventory files, see
[X][ansible-inventory] and [Y][skupper-inventory].

[ansible-inventory]: https://docs.ansible.com/ansible/latest/getting_started/get_started_inventory.html
[skupper-inventory]: https://mit.edu/

## Step 5: Run the setup playbook

Now let's look at the setup playbook.

[ansible/setup.yml](ansible/setup.yml):

~~~ yaml
- hosts: east
  tasks:
    - command: "kubectl apply -f {{ playbook_dir }}/kubernetes/east.yaml"

- hosts: west
  tasks:
    - command: "kubectl apply -f {{ playbook_dir }}/kubernetes/west.yaml"

- hosts: all
  collections:
    - skupper.network
  tasks:
    - import_role:
        name: skupper
~~~

The two `kubectl` tasks deploy our example application.

The last task is to use the `skupper` role from the
`skupper.network` collection to deploy the Skupper network.

Use the `ansible-playbook` command to run the playbook:

_**Terminal:**_

~~~ shell
ansible-playbook -i ansible/inventory.yml ansible/setup.yml
~~~

_Sample output:_

~~~ console
$ ansible-playbook -i ansible/inventory.yml ansible/setup.yml
[...]

PLAY RECAP *********************************************************************************************
east             : ok=34   changed=13   unreachable=0    failed=0    skipped=69   rescued=0    ignored=0
west             : ok=34   changed=12   unreachable=0    failed=0    skipped=69   rescued=0    ignored=0
~~~

## Step 6: Access the frontend

In order to use and test the application, we need external access
to the frontend.

Use `kubectl port-forward` to make the frontend available at
`localhost:8080`.

_**Terminal:**_

~~~ shell
export KUBECONFIG=$PWD/ansible/kubeconfigs/west
kubectl port-forward deployment/frontend 8080:8080
~~~

You can now access the web interface by navigating to
[http://localhost:8080](http://localhost:8080) in your browser.

## Step 7: Run the teardown playbook

To clean everything up, run the teardown playbook.

[ansible/teardown.yml](ansible/teardown.yml):

~~~ yaml
- hosts: all
  collections:
    - skupper.network
  tasks:
  - import_role:
      name: skupper_delete

- hosts: east
  tasks:
    - command: "kubectl delete -f {{ playbook_dir }}/kubernetes/east.yaml"

- hosts: west
  tasks:
    - command: "kubectl delete -f {{ playbook_dir }}/kubernetes/west.yaml"
~~~

The `skupper_delete` role from the `skupper.network` collection
removes all the Skupper resources.

_**Terminal:**_

~~~ shell
ansible-playbook -i ansible/inventory.yml ansible/teardown.yml
~~~

_Sample output:_

~~~ console
$ ansible-playbook -i ansible/inventory.yml ansible/teardown.yml
[...]

PLAY RECAP *********************************************************************************************
east             : ok=9    changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
west             : ok=9    changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
~~~

## Next steps

Check out the other [examples][examples] on the Skupper website.

## About this example

This example was produced using [Skewer][skewer], a library for
documenting and testing Skupper examples.

[skewer]: https://github.com/skupperproject/skewer

Skewer provides utility functions for generating the README and
running the example steps.  Use the `./plano` command in the project
root to see what is available.

To quickly stand up the example using Minikube, try the `./plano demo`
command.
