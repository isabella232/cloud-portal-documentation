Packaging Applications for the |project_name|
===============================================

.. note:: This document is still a **work in progress**. The |project_name| itself is still in very active development,
          and while we strive to keep this document as in sync as possible there might be minor delays. Any comment or feedback is very appreciated!


The EBI Cloud Portal has been built to provide an App-Store-like
experience when deploying applications to internal and public cloud
providers, independently from their complexity. This goal is achieved
exploiting two open source tools, |terraform| and |ansible|. While |terraform| makes very
easy to create complex virtual infrastructures, Ansible shines in configuration
management to turn a set of independent VMs into a, for example, batch system. This
approach, called `Infrastructure as Code <https://en.wikipedia.org/wiki/Infrastructure_as_Code>`_,
allows to reproducibily deploy virtual resources in one or multiple cloud providers
in an highly automated way. The |project_name| builds on top of these two tools
to provide a resilient use web interface allowing users with little to
no knowledge of IT management to easily self-provision the infrastructure they require.
The process of wrapping infrastucture and workloads in an App that will be then
understood and deployed by the |project_name| is called *packaging*.

The tools
--------------

As previously mentioned, the deployment capabilities of the |project_name|
are based on two Open Source tools, |terraform| and |ansible|. Let’s
explore them a little bit further, then!

Terraform
~~~~~~~~~

|terraform| allows defining the virtual infrastructure an application requires to
run in an easily understandable declarative template written in HCL
(`HashiCorp Configuration Language <https://www.terraform.io/docs/configuration/syntax.html>`_).
VMs, networks, firewalls and storage volumes can easily be defined in a single or multiple files, leaving to
|terraform| the burden to understand dependencies between all these
resources and the order in which they must be created. Due to the
intrinsic differences existing between different cloud providers,
|terraform| is currently unable to provide a single template that is then
mapped onto different cloud-specific templates. For this reason, the person in charge of
packaging applications will need to define a |terraform| for each cloud
provider he or she intends to support the application for. However, this
usually comes down to a very reasonable mapping exercise between each
cloud provider object names. Albeit this may be seen as a downside of
Terraform, on the other hand, it allows exploiting many of the
vendor-specific features and services that wouldn’t otherwise be
possible to access. At the time of writing, |terraform| supports the major
public cloud providers (`AWS <https://aws.amazon.com/>`_, `Google Cloud Platform <https://cloud.google.com/>`_,
`Microsoft Azure <https://azure.microsoft.com/en-gb/>`_, and `many
more <https://www.terraform.io/docs/providers/index.html>`_) as well
as `OpenStack <https://www.openstack.org/>`_. There are (as always) other cloud
orchestrators that are able to deliver similar functionalities, but they are
usually bound to a single platform (i.e. AWS `CloudFormation <https://aws.amazon.com/cloudformation/>`_
or OpenStack `Heat <https://docs.openstack.org/heat/latest/>`_).

The Terraform lifecycle
^^^^^^^^^^^^^^^^^^^^^^^

|terraform| is based on a declarative language, which allows you to define
the desired layout of the infrastructure you want to provision in the
cloud. The state of each |terraform| deployment is tracked in what is
called a *state* file, which is basically a list of all the
resources |terraform| has deployed in the previous run. This behaviour
allows the modification of a deployment editing the template as well the
redeployment of resources that are no more available. The life-cycle of
a |terraform| deployment can be divided into mainly three steps: planned,
deployed and destroyed.

Planning
^^^^^^^^

Depending on the initial state being an empty or a partially provisioned
environment, the operations |terraform| will need to perform will be
different. For this reason, the software allows listing all the tasks
that will be carried out in the following run, comparing the desired
state defined in the template and the state file and coming up with a
*plan* that you can revise. This is obtained simply running terraform
plan from within the folder containing the |terraform| template. Keep in
mind that if the state file reports that some components are already
deployed, |terraform| will check if it’s still in place and adjust the
plan, if need.

Deploying *(a.k.a. applying)*
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Applying is the operation that deploys a |terraform| template to a cloud
provider. |terraform| will read the template and the state file (if any)
figuring out which operations must be carried out to reach convergence,
and then start applying them. This process may take a while, depending
on the extent of the required changes and their mutual dependencies, but
can usually greatly speeded up increasing the *parallelism*, the number
of objects will act on at the same time. Once the deployment is
complete, |terraform| will output any defined output in the template and
exit.

Destroying
^^^^^^^^^^

After his honourable service, your infrastructure is ready to be torn
down or destroyed, following Terraform’s nomenclature. Not violating
dependencies is an important factor to consider here, as this might
cause errors in the destroy process (i.e. removing a subnetwork while
instances are still hooked into it). |terraform| wraps all this into an
easy to use a single command.

Ansible
~~~~~~~

While |terraform| provides some features to configure (or *localise*) VMs
after they’re launched, this is limited to uploading bash scripts or a
run single command through SSH. Configuring and orchestrating complex
deployments do require a fully fledged configuration management system.
Countless different software are available to solve this problem, each
of them having its own strong and weak points. We’ve eventually chosen
|ansible| as the configuration management system for the EBI Portal
deployments due to its very easy learning curve and to the fact it
doesn’t require any agent on the VMs since it only needs an SSH
connection to work. On top of that, it’s very easy YAML-based syntax can
usually be learned and put into use in a matter of hours, not days.

After a set of resources is created by |terraform|, |ansible| can take over
by applying all the required configuration changes (i.e. install
packages or update configuration files). |ansible| simplifies the process
with a small trade-off: It is good enough to manage and maintain simple
to moderately complex infrastructures because it employs a declarative
approach to configuration statements. However, it’s important to note
that the choice of supporting only |ansible|-based deployments from the
portal doesn’t imply that applications themselves are forced to use this
tool: it’s in fact quite easy to use |ansible| to bootstrap a Salt server
(or a Puppet master) that is then used by other VMs to configure
themselves.

.. note::
          Since version 2.0 |ansible| has added several modules to provision
          virtual infrastucture as |terraform| does. However, |terraform| still
          provides clear advantages, such as dependencies resolution, state
          tracking, and a much wider range of supported clouds. For these reasons,
          it still represents our preferred choice.

.. warning::
          While there's nothing to stop you from using Ansible to provision the
          virtual infrastucture required by your Application, doing so will prevent
          the |project_name| from tracking resource consumption as this feature
          relies on inspecting the |terraform| state file.


Linking Terraform and Ansible
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

|terraform| outputs the final state of the deployment in a state file.
However, |ansible| relies on an inventory file to know to IP addresses of
the VMs it needs to talk with and their logical grouping. To bridge this
gap, the |project_name| supports
`terraform-inventory <https://github.com/adammck/terraform-inventory>`_,
a small GO app that is able to parse a |terraform| state file and output
its content as an |ansible| inventory.

The EBI Cloud Portal packaging structure
----------------------------------------

Cloud providers
~~~~~~~~~~~~~~~

The Portal relies on a homogeneous labelling of Cloud Providers to
match, for example, deployments with credentials and the cloud-specific
code that must be executed each time. We strongly suggest following the
labelling schema below to take full advantage of all the features the
portal offers.

+-------------------------+--------+
| Cloud Provider          | Label  |
+=========================+========+
| Amazon Web Services     | AWS    |
+-------------------------+--------+
| Google Compute Platform | GCP    |
+-------------------------+--------+
| Microsoft Azure         | AZURE  |
+-------------------------+--------+
| OpenStack               | OSTACK |
+-------------------------+--------+

The general structure
~~~~~~~~~~~~~~~~~~~~~

Here’s the general structure of a repository hosting a packaged
application for the portal: ::

   ├ .gitignore
   ├ README.md
   ├ aws
   │ ├ ansible -> ../gcp/ansible/
   │ ├ deploy.sh
   │ ├ destroy.sh
   │ ├ state.sh
   │ └ terraform
   ├ gcp
   │ ├ ansible
   │ ├ deploy.sh
   │ ├ destroy.sh
   │ ├ state.sh
   │ └ terraform
   ├ manifest.json
   └ ostack
     ├ ansible -> ../gcp/ansible/
     ├ deploy.sh
     ├ destroy.sh
     ├ state.sh
     ├ terraform
     └ volume_parser.py


As you can see, there’s a file ``manifest.json`` at the root of it, and
then folders storing code for each cloud provider. In this particular
repo, the |ansible| code is shared among the cloud providers via symlinks,
but this is not a strict requirement. Being fully honest, there’s hardly
strict requirements at all in the way the Portal consumes applications!
Let’s have a more in-depth look, then!

Where to store your code
~~~~~~~~~~~~~~~~~~~~~~~~

The code defining an application for the EBI Cloud Portal must be
tracked within a git repository publicly clonable over the internet.
This is a fundamental requirement, as the way the Portal imports
applications in its own registry is cloning such repositories.

Out the many ways, we may have supported this, we eventually chose git
repository as this allows to easy to track code changes, keep dev and
production deployments separated in different branches, and provides a
well-established approach to final users to customize their own
deployments forking the original repository.

The manifest file
~~~~~~~~~~~~~~~~~

Each repository defining an application must contain a *manifest* at its
root describing it. This file will be parsed by the EBI Cloud Portal
when importing the application to populate the Registry fields. Here’s
an example of the manifest file defining an OpenLava cluster
application:

::

    {
        "applicationName":"OpenLava cluster",
        "contactEmail":"dario@ebi.ac.uk",
        "about":"An OpenLava cluster for AWS, GCP and OpenStack",
        "version": "0.1",
        "inputs": ["nodes"],
        "outputs": ["MASTER_IP"],
        "volumes": ["DATA_DISK_ID"],
        "cloudProviders": [
            { "cloudProvider":"AWS", "path":"aws", "inputs": [ "vpc_id" ] },
            { "cloudProvider":"OSTACK", "path":"ostack" },
            { "cloudProvider":"GCP", "path":"gcp"}
       ]
    }

Many of the fields are self-explanatory, but let’s walk through them
anyway:

-  ``applicationName`` The name that will be shown in the Portal
   registry for this application
-  ``contactEmail`` The email address of the person or group maintaining
   the application
-  ``about`` A (*very*) brief description of what the application does
-  ``version`` The current version of the application
-  ``inputs`` An array of strings defining the inputs required by the
   application, in this particular case the number of nodes to be
   deployed in our OpenLava cluster. Input fields will be shown by the
   Portal to allow users to customize the deployment behaviour. All the
   values will then be injected as environment variables when deploying,
   making them accessible to |terraform| and |ansible|.

Any ``input`` name defined in the manifest will be injected into the
environment as ``TF_VAR_input``. Since |terraform| automatically imports
environment variables with the ``TF_VAR_`` prefix and maps them to its
own internal variables (removing the prefix), this allows to easily wire
up the deployment with user inputs. Using our OpenLava deployment as an
example, the portal will show an input field named ``nodes``, and inject
the value entered by the user in the environment variable
``TF_VAR_nodes``, that is the read by |terraform| and mapped to its
internal variable ``nodes``. Should an |ansible| playbook need to access
the same input value, it must look for the ``TF_VAR_input`` environment
variable, as no automatic mapping is available in |ansible|.

outputs
^^^^^^^


A very common use case when deploying infrastructure to the cloud is the
need to show back to the user some information resulting from the
deployment itself, as for example the external IP address of a batch
system master node. The portal will scan the output of the Terraform
state file looking for the strings defined in this JSON array, and
display the result to the user.

volumes
^^^^^^^

Sometimes, a deployment requires attaching a previously defined volume.
For example, some data may be staged in via a GridFTP server on a
particular volume, that is then re-attached to an NFS server serving a
batch system. The EBI Cloud Portal allows to completely separate the
volumes lifecycle from the lifecycle of applications. Adding a volume
name (i.e. ``DATA_DISK_ID`` in our previous example) to volumes
automatically displays on the deployment card a drop-down menu listing
all the volumes deployed through the portal. The id of the selected
volume (as provided by the cloud provider, not the portal internal id)
is then injected into the deployment process as an environment variable
(i.e. ``TF_VAR_DATA_DISK_ID`` in this case).

cloudProviders
^^^^^^^^^^^^^^


This is where the magic happens! This JSON array contains a dictionary
(an hash table, following JSON nomenclature) for each cloud provider the
application supports. The structure is as follows:

::

    { "cloudProvider":"AWS", "path":"aws", "inputs": [ "vpc_id" ] }

``cloudProvider`` specifies which cloud provider the dictionary is
providing information for, ``path`` is the path to the folder where the
code to deploy to the given cloud provider is located, while inputs is
an optional JSON array defining cloud-specific inputs (in our example,
the ``vpc_id`` to use on AWS).

The ``cloudProvider`` value is also used to pick the right credentials
when deploying. At the time of writing, the portal simply looks among
the defined credentials and picks the one tagged with the same string
(``AWS`` in this case), so it is important to follow the labelling
schema previously mentioned.

How to organise your code in the git repository
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Separate each cloud provider
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As you’ve learned in the section about the manifest file, the code to
deal with each cloud provider must be kept in a separate folder. Even
though the naming of these folders is currently left to each individual
author, we suggest sticking to the following schema in the “**Cloud
providers**” section above.

Following this convention ensures that the repository will be more
easily understood by other developers and make the credential matching
more reliable.

Separate Terraform and Ansible
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As for the cloud providers, we suggest keeping separate the Terraform
and |ansible| codebases as this improves much more the readability and
maintainability of the repository. Also, it allows for some tricks like
sharing the same |ansible| code among different cloud providers (symlinks
are good!) or using git
`submodules <https://git-scm.com/book/en/v2/Git-Tools-Submodules>`__ to
share code between several deployments.

Deployment scripts
^^^^^^^^^^^^^^^^^^

The EBI Cloud portal is currently unable to directly execute Terraform
and |ansible| commands, but exploits bash scripts to perform the
deployments. Even if some work is currently in progress to move away
from that, this is likely to remain the paradigm the portal will follow
in the close future. Three deployment scripts are required for each
cloud provider: deploy.sh, destroy.sh, state.sh. These must be saved in
the root folder of each cloud provider deployment.

Special variables
*****************

Aside from the environment variables needed to authenticate against the
APIs of the cloud providers, the portal will automatically inject some
variables referring to the paths where data can be stored and retrieved,
along with the deployment id. Currently, the complete list is as
follows:

+-----------------------------------+-----------------------------------+
| Environment variable              | Value                             |
+===================================+===================================+
| PORTAL_APP_REPO_FOLDER            | The path where the application    |
|                                   | code is stored (e.g. the cloned   |
|                                   | repo). Only available in the      |
|                                   | deploy and destroy phase, not     |
|                                   | when checking the state running   |
|                                   | state.sh                          |
+-----------------------------------+-----------------------------------+
| PORTAL_DEPLOYMENTS_ROOT           | The path to the root of the       |
|                                   | folder storing all the deployment |
|                                   | folders                           |
+-----------------------------------+-----------------------------------+
| PORTAL_DEPLOYMENT_REFERENCE       | The ID assigned to the deployment |
|                                   | by the portal                     |
+-----------------------------------+-----------------------------------+

A very common use case for these variables is to place the Terraform
output in the folder belonging to your deployment: this path can be
easily obtained joining ``PORTAL_DEPLOYMENTS_ROOT`` and
``PORTAL_DEPLOYMENT_REFERENCE``, e.g.

::

    "$PORTAL_DEPLOYMENTS_ROOT'/'$PORTAL_DEPLOYMENT_REFERENCE'/terraform.tfstate'"

deploy.sh
*********

This script takes care of deploying the application, and usually
consists of at least a |terraform| and an |ansible| call. Here’s a snippet of
the current deploy.sh for a GridFTP server on GCP:

::

    #!/usr/bin/env bash
    set -e
    # Provisions a GridFTP instance in GCP
    # For details about expected inputs and outputs, refer to https://github.com/EMBL-EBI-TSI/gridftp-server
    # The script assumes that env vars for authentication with GCP are present.
    export TF_VAR_name="$(awk -v var="$PORTAL_DEPLOYMENT_REFERENCE" 'BEGIN {print tolower(var)}')"
    export KEY_PATH="${HOME}/.ssh/demo-key.pem"

    # Launch provisioning of the VM
    terraform apply --state=$PORTAL_DEPLOYMENTS_ROOT'/'$PORTAL_DEPLOYMENT_REFERENCE'/terraform.tfstate' $PORTAL_APP_REPO_FOLDER'/gcp/terraform'

    # Start local ssh-agent
    eval "$(ssh-agent -s)"
    ssh-add $KEY_PATH &> /dev/null

    # Get ansible roles
    cd gcp/ansible || exit
    ansible-galaxy install -r requirements.yml

    # Run Ansible
    TF_STATE=$PORTAL_DEPLOYMENTS_ROOT'/'$PORTAL_DEPLOYMENT_REFERENCE'/terraform.tfstate' ansible-playbook -i /usr/local/bin/terraform-inventory -u centos -b --tags live deployment.yml > ansible.log 2>&1

    # Kill local ssh-agent
    eval "$(ssh-agent -k)

As you can see, there are a few additional things going on here rather
than two simple |terraform| and |ansible| calls. Again, let’s go
step-by-step!

::

    #!/usr/bin/env bash
    set -e
    # Provisions a GridFTP instance in GCP
    # For details about expected inputs and outputs, refer to https://github.com/EMBL-EBI-TSI/gridftp-server
    # The script assumes that env vars for authentication with GCP are present.
    export TF_VAR_name="$(awk -v var="$PORTAL_DEPLOYMENT_REFERENCE" 'BEGIN {print tolower(var)}')"
    export KEY_PATH="${HOME}/.ssh/demo-key.pem"

This initial block defines the `shebang <https://en.wikipedia.org/wiki/Shebang_(Unix)>`_
for the script (``#!/usr/bin/env bash``) and forces the bash script to exit
immediately if any command exits with a non-zero status (``set -e``).
Then, it exports two environment variables: ``TF_VAR_name`` and
``KEY_PATH``. The first will automatically be picked up by |terraform| and
mapped to its internal variable name, eventually causing each resource
to be named after the deployment ID (more on this in the next session),
while the second allows defining the path to the SSH key to be used to
access the VMs.

::

    # Launch provisioning of the VM
    terraform apply --state=$PORTAL_DEPLOYMENTS_ROOT'/'$PORTAL_DEPLOYMENT_REFERENCE'/terraform.tfstate' $PORTAL_APP_REPO_FOLDER'/gcp/terraform'

This block is quite obvious: deploy the defined |terraform| template to
the cloud provider. Keep in mind that the Portal will have already
injected the needed credentials in the deployment environment, so you
don’t need to care about that.

::

    # Start local ssh-agent
    eval "$(ssh-agent -s)"
    ssh-add $KEY_PATH &> /dev/null

    # Get ansible roles
    cd gcp/ansible || exit
    ansible-galaxy install -r requirements.yml

    # Run Ansible
    TF_STATE=$PORTAL_DEPLOYMENTS_ROOT'/'$PORTAL_DEPLOYMENT_REFERENCE'/terraform.tfstate' ansible-playbook -i /usr/local/bin/terraform-inventory -u centos -b --tags live deployment.yml

    # Kill local ssh-agent
    eval "$(ssh-agent -k)"

This block deals with everything that is needed by |ansible| to work. When
the Portal launches the deployment script, a new
`ssh-agent <https://en.wikipedia.org/wiki/Ssh-agent>`__ is spawned and
the SSH key to access the VMs is pre-loaded. Then, ansible-galaxy is
used to pull all the requirements for the playbook to run (keep in mind
that only public repositories will be clonable). Next step, invoking
|ansible| itself. It’s not a very plain invocation, though:

-  prefixing the command with ``TF_STATE=...`` tells terraform-inventory
   where to look for the |terraform| state file

-  ``-i /usr/local/bin/terraform-inventory`` tells |ansible| to use
   terraform-inventory to create the inventory on the flight. Keep in
   mind that |ansible| supports as arguments of the ``-i`` flag both text
   files containing an inventory and *executables returning an
   inventory.*

-  ``-u centos -b`` force |ansible| to use the user centos over ssh and to
   execute commands with ``sudo`` (b =
   `become <http://docs.ansible.com/ansible/become.html>`__)

The last step is to kill the previously spawned ssh-agent. Deployment
(hopefully) done!

destroy.sh
**********

This script is executed by the EBI Cloud Portal to destroy an
application. It usually consists of a single |terraform| call to destroy
the provisioned infrastructure. Here’s an example, again from a GridFTP
server.

::

    #!/usr/bin/env bash
    set -e
    # Destroys a GridFTP deployment in GCP
    # For details about expected inputs and outputs, refer to: https://github.com/EMBL-EBI-TSI/gridftp-server
    # The script assumes that env vars for authentication with GCP are already present.

    # Export input variable in the bash environment
    export TF_VAR_name="$(awk -v var="$PORTAL_DEPLOYMENT_REFERENCE" 'BEGIN {print tolower(var)}')"

    # Destroy everything
    terraform destroy --force --state=$PORTAL_DEPLOYMENTS_ROOT'/'$PORTAL_DEPLOYMENT_REFERENCE'/terraform.tfstate' $PORTAL_APP_REPO_FOLDER'/gcp/terraform'

Nothing fancy, right?

state.sh
********

This script is executed by the Portal immediately after the deployment
to grab an updated picture of all the deployed resources. It’s basically
a wrapper around the |terraform| state command. Here’s the usual example!

::

    #!/usr/bin/env bash
    set -e
    # Get the status of a GridFTP deployment in GCP
    # For details about expected inputs and outputs, refer to https://github.com/EMBL-EBI-TSI/gridftp-server
    # The script assumes that env vars for authentication with GCP are present.

    # Query Terraform state file
    terraform show $PORTAL_DEPLOYMENTS_ROOT'/'$PORTAL_DEPLOYMENT_REFERENCE'/terraform.tfstate'

Auxiliary scripts
*****************

Depending on the particular needs of each application, you might need
auxiliary scripts to carry out the deployment successfully. These can
currently be added to any folder within the repo and invoked via the
bash scripts. We suggest placing the outputs of these commands (if any)
in the deployment folder.

Cloud credentials
~~~~~~~~~~~~~~~~~

At the moment, the EBI Cloud Portal supports credentials for all cloud
providers, as long as these can be provided to |terraform| injecting a
properly defined environment variable. A user can provide multiple
credentials for different cloud providers, but we currently support a
*single* set of credentials for each of them.

Each set of credentials is defined in the portal by three fields:

-  ``Credential name`` A name for the credentials set

-  ``Cloud provider`` The cloud provider to which this set of cloud
   credentials refers to. Please refer to the labelling schema
   previously mentioned to pick the right label for the cloud provider.

-  ``Credentials fields`` This field contains a JSON array defining the
   credentials to be injected into the environment to allow |terraform| to
   authenticate with the cloud provider. Here’s an example of how an
   OpenStack array looks like:

::

    [
        {"key": "OS_AUTH_URL", "value":"https://someurl.com:5000/v2.0"},
        {"key": "OS_TENANT_NAME", "value": "tenant_name"},
        {"key": "OS_USERNAME", "value": "username"},
        {"key": "OS_PASSWORD", "value": "password"}
    ]

One caveat: since this code will be read by a Java appliance, remember
to escape any special character you may have in your credentials. A good
example of this is the private key used as part of the GCP
authentication: it contains some newline characters (``\n``) that will
need to be escaped (``\\n``).

Other configurations (moving towards a profile concept)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. warning:: This section is not in sync with the current state of the |project_name|
             and will be updated soon.

Sometimes injecting credentials are not enough. For example, GCP has the
concept of projects, which are a separate compartment in which a single
account can be divided into. |terraform| needs to know to which
compartment resources should be deployed, and this is usually done
specifying the project in the
`provider <https://www.terraform.io/docs/providers/google/>`__. As the
packaged application must be able to deploy itself in any project, this
should be provided as an input. However, inputs must be typed in, each
time the application is deployed! How can we fix this? Well, here’s the
trick: |terraform| can also read the project from a dedicated environment
variable: ``GOOGLE_PROJECT``. If we are planning to deploy always to the
same project, we can simply add another variable to the credentials JSON
array defining the ``GOOGLE_PROJECT`` environment variable, so that it
will always be injected when deploying. A whole range of similar
problems can be solved via this approach, i.e. feed the id of a shared
AWS VPC to the deployments. However, this is currently limited by the
fact that we only support a single set of credentials for each cloud
provider. Once this limitation will be removed, we’ll revisit the
concept of credential sets, possibly moving towards *cloud profiles*.

Testing locally
~~~~~~~~~~~~~~~

Especially at the beginning of the packaging process, it is very useful
to test deployments locally. Keep also in mind that if your application
fails to deploy in the portal, it might very hard to get a clear reason
why that happened without access to the logs (which of course are
*super-secret!*)

So, how to reproduce the Portal behaviour locally? First, you’ll need to
install a few dependencies:
|terraform|,|ansible| and
`terraform-inventory <https://github.com/adammck/terraform-inventory>`__
(click on the links to go to their respective “How-to install pages”).
Second, you need to replicate the deployment environment. As you should
have now understood, the only way the portal interacts with your
deployments at the moment is setting *environment variables*.
Reproducing this is very easy, thanks to
`source <https://en.wikipedia.org/wiki/Source_(command)>`__!

We can define a small file like this:

::

    #!/bin/bash
    # Define the three special env vars
    export PORTAL_DEPLOYMENTS_ROOT="absolute/path/to/repo"
    export PORTAL_DEPLOYMENT_REFERENCE="test_deployment"
    export PORTAL_APP_REPO_FOLDER="."

    # Define the volume id of the volume to be linked to our deployment
    export TF_VAR_DATA_DISK_ID="vol-fb65c979"

And then simply run ``source filename``. This will inject all the
variables defined in the file into the bash environment (keep in mind
that you’ll need to run again the command if you move to another
terminal). At the bare minimum, you’ll need to export the three portal
special variables (``PORTAL_DEPLOYMENTS_ROOT``,
``PORTAL_DEPLOYMENT_REFERENCE`` and ``PORTAL_APP_REPO_FOLDER``) plus one
variable for each input your application needs (remember to prepend it
with ``TF_VAR_``).

Similarly, you need to source the credentials for the cloud provider you
want to interact with. OpenStack allows you to download a pre-populated
script to be sourced from its web interface, Horizon (exactly in “Access
& Security” tab -> “API Access” subtab -> “Download OpenStack RC file”).

For AWS you’ll need to create your own script to be sourced, here’s an
example:

::

    #!/bin/bash
    export AWS_ACCESS_KEY_ID="your_access_key_id"
    export AWS_SECRET_ACCESS_KEY="you_secret_access_key"

Finally, for GCP you’ll need to download a JSON file from the `Google
Developers Console <https://console.developers.google.com/>`__. Here’s
the process step-by-step, as defined by the |terraform| documentation for
the `GCP provider <https://www.terraform.io/docs/providers/google/>`__:

1. Log into the Google Developers Console and select a project.

2. The API Manager view should be selected, click on “Credentials” on
   the left, then “Create credentials”, and finally “Service account
   key”.

3. Select “Compute Engine default service account” in the “Service
   account” drop-down, and select “JSON” as the key type.

4. Clicking “Create” will download your credentials.

Once you have the file, you can easily define a one-line script to load
its content in the appropriate env vars, as follows:

::

    #!/bin/bash
    export GOOGLE_CREDENTIALS="`cat path/to/the/json/file.json`"

At this point, invoking the various deployment scripts from the root of
your repository (i.e. ./gcp/deploy.sh) should just work. **Happy
packaging!**

Portal usage
------------

.. warning:: This section is deprecated.
             Please refer to the :ref:`using-the-portal` section instead.

Configuring repositories
~~~~~~~~~~~~~~~~~~~~~~~~

Before deployments can be made, a user first has to configure portal
repositories. These are application definitions which are later used for
deployment purposes.

Steps:

1. User logs into EBI Cloud Portal.

2. On the Dashboard, the user clicks on “Search Repositories” or selects
   “Repository” from the left pane menu.

3. Click on the “+” button on the right side of the screen.

4. On the “Add application screen” user pastes public repo URL and
   clicks “Add”.

5. The application is added to your repository.

The deployment process overview
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

So, eventually, what are the steps the portal takes every time it needs
to deploy or destroy an application? And, how the ``deploy.sh`` and
``destroy.sh`` scripts links into that?

Deployment
~~~~~~~~~~

Here’s a step-by-step list of every operation the cloud portal performs
to deploy an application:

-  The user clicks the “Deploy” button on an application in the Portal,
   after providing all the required inputs and selecting the cloud
   provider. The web app sends the request to the API.

-  The selected cloud provider is matched with the credentials in the
   user profile. If a match is found, they are injected in the
   deployment environment. If not match is found, the process exits with
   an error that is reported back to the web app.

-  Input variables and the volume IDs, if present, are injected into the
   environment.

-  The cloud-specific deploy.sh script is executed (e.g.
   /root/gcp/deploy.sh).

   Internally, the ``deploy.sh`` script executes these steps:

   1. Runs |terraform| to provision the resources according to the
      pre-defined template

   2. Runs |ansible| to apply the configuration on the provisioned VMs. An
      |ansible| inventory is produced on the fly by terraform-inventory
      starting from the |terraform| state file to feed |ansible| with the
      IPs of the machine it needs to talk to, along with their logical
      grouping

-  If the deployment script exits with a non-zero status (it fails), the
   information is sent back to the web app and the process stops. If the
   deployment script exits with a zero, the process continues

-  Executes the cloud-specific ``state.sh`` script, and looks for the
   outputs defined in the manifest (if any)

-  Reports the outputs (if any) back to the web app

Destroy
~~~~~~~

The destroy phase is usually much easier - in many cases it only
consists of a single |terraform| call to tear down the resources. But how
this works from the portal perspective?

Again, here’s the list!

-  The user clicks the Destroy button on the web application. A request
   to the API is fired to tear down the deployment

-  Credentials for the cloud provider hosting the deployment are
   injected into the environment. If not match is found among the
   credentials into the user profile, the process exits with an error
   that is reported back to the web app.

-  Input variables and the volume IDs, if present, are injected into the
   environment.

-  The cloud-specific ``destroy.sh`` script is executed

   Internally, the ``destroy.sh`` script executes a single step:

   1. Runs |terraform| to destroy the resources, as they’re reported in
      the state file

   In some cases destroying a deployment may require some preliminary
   steps, e.g. power the VMs off in advance with |ansible|. These needs
   can simply be fulfilled by, for example, using a separate |ansible|
   playbook to be executed before invoking Terraform. It is however
   imperative that all the unneeded resources are removed at the of the
   process, as users will not be able to remove them at a later time.

-  If the destroy script exits with a non-zero status (it fails), an
   error is displayed by the web app and the process stops. On the
   contrary, if the destroy script exits with a zero (success), the
   deployment is removed by the web app and the process concludes.
