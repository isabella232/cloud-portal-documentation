Packaging Applications for the |project_name|
===============================================

.. note:: This document is still a **work in progress**. The |project_name| itself is still in very active development,
          and while we strive to keep this document as in sync as possible there might be minor delays. Any comment or feedback is very appreciated!


The |project_name| has been built to provide an App-Store-like
experience when deploying applications to private as well as public clouds,
independently from their complexity. Leveraging multiple open source tools (see :ref:`portal-tools`), the
Portal can orchestrate complex virtual infrastructures ranging from batch
systems to *Pipeline-as-a-service* scenarios, where the limit is only set by the
application developer. This approach, called `Infrastructure as Code <https://en.wikipedia.org/wiki/Infrastructure_as_Code>`_,
allows to reproducibily deploy virtual resources in one or multiple cloud providers
in an highly automated way. The |project_name| builds on top of these tools
to provide a resilient REST API and a web interface allowing users with little to
no knowledge of IT management to easily self-provision the infrastructure they require.
The process of wrapping infrastucture and workloads in an App that will be then
understood and deployed by the |project_name| is called *packaging*.

.. _portal-tools:

The tools
--------------

While the |project_name| can easily fall in the category of the **Cloud Orchestrators**,
one of its streghts is taking advantage of widely adopted Open Source tools to
deploy and configure the infrastructure it is required to manage. Adopting this
approach limits the amount of efforts that are required to, for example, stay on
top of all the changes in the Cloud APIs that providers expose to access their services,
to focus on its own very mission: support IT staff as well as researchers in easily
deploy the infrastructure they require for their needs.

As previously mentioned, the |project_name| takes advantage of two open source
tools to orchestrate infrastructure, namely |terraform| and |ansible|. Let’s
explore them a little bit further, then!

Terraform
~~~~~~~~~

|terraform| allows to define the virtual infrastructure an application requires to
run in an easily understandable declarative template written in HCL
(`HashiCorp Configuration Language <https://www.terraform.io/docs/configuration/syntax.html>`_).
VMs, networks, firewalls and storage volumes can easily be defined in a single or multiple files, leaving to
|terraform| to understand dependencies between all these
resources and thus the order in which they must be created. Cloud providers
offering are, however, too diverse to allow a single template to be automatically
deployed across several of them. |terraform| doesn't try to hide these differences,
For this reason, the person in charge of packaging applications will need to define
a |terraform| template for each cloud provider he or she intends to support the
application for. However, this usually comes down to a very reasonable mapping
exercise between each cloud provider object names. This can be seen as a downside of
|terraform| but, on the other hand, it allows to take advantage many of the
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
resources |terraform| has deployed in the previous run. Comparing the state file
with the *desired* state defined in the templates, |terraform| can compute their
*diff* and incrementally change a deployment, i.e. increasing the number of nodes
in a cluster, or recover from failures.


Planning
^^^^^^^^

Depending on the initial state being an empty or a partially provisioned
environment, the operations |terraform| will need to perform will be
different. For this reason, the software allows listing all the tasks
that will be carried out in the following run, comparing the desired
state defined in the template and the state file and defining a
*plan* that you can revise. This is obtained simply running ``terraform plan``
within the folder containing the |terraform| template. Keep in
mind that if the state file reports that some components are already
deployed, |terraform| will check if they are still in place and adjust the
plan accordingly.

Applying
^^^^^^^^

``terraform apply`` is the operation that deploys a |terraform| template to a cloud
provider. |terraform| will read the template and the state file (if any)
figuring out which operations must be carried out to reach convergence,
and apply them. This process may take a while, depending
on the extent of the required changes and their dependencies, but
can usually be greatly speeded up increasing the *parallelism*, or the number
of objects |terraform| will act on at the same time.
Once the deployment is complete, |terraform| will print out any output defined in
the template and exit.

Destroying
^^^^^^^^^^

After its honourable service, your infrastructure is ready to be torn
down or destroyed, following |terraform|’s nomenclature. Not violating
dependencies is an important factor to consider here, as this might
cause errors in the destroy process (i.e. removing a subnetwork while
instances are still hooked into it). |terraform| wraps all this into an
easy to use a single command: ``terraform destroy``.

Ansible
~~~~~~~

While |terraform| provides some features to configure (or *localise*) VMs
after they’re launched, this is limited to uploading bash scripts or
run bash commands through SSH. Configuring and orchestrating complex
deployments usually requires a fully fledged configuration management system.
Countless different software are available to solve this problem, each
of them having its own strong and weak points. We’ve eventually chosen
|ansible| as the configuration management system for the |project_name|
deployments for several reasons:

- configuration is written in `YAML <https://en.wikipedia.org/wiki/YAML>`_,
  an *easy-to-read* and *easy-to-write* language.
- the learning curve is very gentle, and most bash scripts can be easily mapped
  (and improved!) in Ansible tasks.
- it doesn't require any agent on the target VMs, only SSH access.

After a set of resources is created by |terraform|, |ansible| can take over
and apply the configuration changes (i.e. install packages, update configuration
files and so on).

While |ansible| represents our choice in all the deployment situations, it doesn’t
imply that applications themselves are forced to use this tool. For example, it
would be quite easy to to use |ansible| to bootstrap a Salt server
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

Of course, developers are not bound to use this method to connect |terraform|
and |ansible|. Solutions such as the `Terraform Ansible Provisioner <https://github.com/jonmorehouse/terraform-provisioner-ansible>`_
or even custom scripts are viable options, depending on the needs of the App
developer.

The |project_name| packaging structure
----------------------------------------

The |project_name| has been designed to provide as much flexibility as possible
when dealing with Apps development. However, some conventions need to be followed
while designing your App in order for it to work properly and to take advantage
of all the features we provide.

.. _cloud-providers:

Cloud providers
~~~~~~~~~~~~~~~

The Cloud world can be, as the name says, very *cloudy*. However, the |project_name|
needs to be absolutely sure of which cloud provider an application can be deployed
to ensure it's providing the right set of configurations to the final user of your
App. For this reason, the |project_name| relies on a homogeneous labelling of
Cloud Providers in the Apps definition as well as in the REST API and the web
application. You *must* follow this convention:

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

If the Cloud Provider you want to write an App for isn't listed here, please get
in touch with us - we'll be happy to add it to the list!

Where to store your code
~~~~~~~~~~~~~~~~~~~~~~~~

First things first, where do you need to store your code?

The code defining an application for the |project_name| must be
tracked within a Git repository publicly clonable over the internet.
This is a **fundamental** requirement, as the way the Portal imports
applications in your Registry is cloning such repositories.

Adopting Git as our main delivery mechanisms allowed us to easily track code
changes, keep ``dev`` and ``prod`` deployments separated in different branches,
and provides a well-established approach to final users to further customise
deployments above what initially foreseen by the App developer simply forking
the original repository.

The general structure
~~~~~~~~~~~~~~~~~~~~~

Apps, especially those supporting multiple cloud providers, can consist of a
reasonable number of lines of code scattered across multiple files and written
in several languages. It is thus important to keep some logical order in the
codebase to help other users - and yourself in a few months! - understand how
your application has been defined and operates. From the |project_name| perspective,
there are a few requirements that must be satisfied when writing your app, and
we'll cover those in the next sections.


Separate Cloud Providers
^^^^^^^^^^^^^^^^^^^^^^^^
The code used to deploy to each cloud provider - being it |terraform|, |ansible|
or anything else you require - must be stored in a dedicated folder. The names
of these folders are currently not subject to any restriction, but we suggest
to give them meaningful names (such as those suggested in the :ref:`cloud-providers`
section above).

Following this convention ensures that the repository will be more
easily understood by other developers and help configuration matching.


Separate Terraform and Ansible
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As for the Cloud Providers, we suggest keeping separate the |terraform|
and |ansible| codebases as this improves the readability and
maintainability of the repository. Also, it allows for some tricks like
sharing the same |ansible| code among different cloud providers (symlinks
are good!) or using git
`submodules <https://git-scm.com/book/en/v2/Git-Tools-Submodules>`__ to
share code between several deployments.


Manifest file
^^^^^^^^^^^^^^

The manifest file is a file containing a ``JSON`` dict providing a description of the application
parsed by the |project_name| when loading it. You can find more information
on its structure and the mandatory fields in the :ref:`manifest-file`
section below.


Deployment scripts
^^^^^^^^^^^^^^^^^^

When deploying or destroying an Application, the |project_name| doesn't directly
execute |terraform| or |ansible|, but executes the ``deploy.sh`` and ``destroy.sh``
scripts that it expects to find in each folder dealing with a cloud provider deployment.
A third script, ``state.sh``, is executed after the deployment succeeds to capture
a snapshot of the deployed infrastucture. More details on how these scripts
should be coded are available in the :ref:`deployment-scripts` section.


Auxiliary scripts
^^^^^^^^^^^^^^^^^

There might be situations requiring additional scripts or tools to carry out the
deployment successfully. Feel free to add them to a folder within the repo, either
in a cloud provider-specific folder if it's needed only by a single cloud provider or
in a generic folder in the root of the repository if you need it in all clouds.


The final structure
^^^^^^^^^^^^^^^^^^^

Putting everything together, here's how a repository hosting a packaged App looks
like: ::

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


.. _manifest-file:

The manifest file
~~~~~~~~~~~~~~~~~


Each repository defining an application must contain a ``JSON`` file, called a
*manifest* fine, at the root of the repo. This file is parsed by the |project_name|
when adding an application to the Registry to extract things such as application name,
version, contact email of the maintainer, and so on. Here’s
an example of the manifest file defining a ``Generic server instance`` App supporting
both ``AWS`` and ``OSTACK``:

::

  {
    "applicationName": "Generic server instance",
    "contactEmail": "somebody@ebi.ac.uk",
    "about": "A base virtual machine instance",
    "version": "0.6",
    "cloudProviders": [
      {
        "cloudProvider": "AWS",
        "path": "aws",
        "inputs": [
          "instance_type"
        ]
      },
      {
        "cloudProvider": "OSTACK",
        "path": "ostack",
        "inputs": [
          "flavor_name"
        ]
      }
    ],
    "deploymentParameters": [
      "network_name",
      "floatingip_pool",
      "subnet_id"
    ],
    "inputs": [
      "disk_image"
    ],
    "outputs": [
      "external_ip"
    ],
    "volumes": [
    ]
  }

Nothing too difficult, hopefully! The manifest is logically divided in two *parts*:
one dealing with the general description of the application, and one dealing with
configurations that are specific to a cloud provider. Let's start from the general one
first


Cloud provider independent bits
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This part of the manifest deals with all the information that is cloud provider
*independent*, such as name of the App, maintainer, version, as well as inputs and outputs.
While many of the fields are self-explanatory, here's a run down of all of them:


applicationName (Required)
  The name of the application packaged in the git repo.

  As users can have many applications
  in their Registries, going for a descriptive name is a good approach (``some server`` isn't
  going to get you far!).

contactEmail (Required)
  The email address of the person (or group) in charge of maintaining the Application
  and provide support for it. *Mandatory*

about (Required)
  A one-line description on what the Application does. *Mandatory*

  This will be displayed below the title in the App card within the Repository.

version (Required)
  The current version of the application. This is also
  displayed in the App card in the Repository.

.. _manifest-deploymentParameters:

deploymentParameters (Optional)
  A list of the Deployment Parameters for this app.

  Deployment parameters are all those parameter that *do not* change between
  deployments, but are *cloud provider* or *tenancy* specific. For example, the
  name (or id) of the external network in an Openstack cloud depends on the cloud
  itself, but is always the same when deploying to a given cloud. It thus makes
  sense to separate these parameters from deployment-dependent parameters (see
  :ref:`inputs <manifest-inputs>` for those) to save the user the hassle to type them every time.


  Variables defined here will be injected by the |project_name| in the deployment
  environment prepended with the suffix ``TF_VAR_`` to allow |terraform| to use
  them `directly <https://www.terraform.io/docs/configuration/variables.html#environment-variables.>`_.
  Values for the ``deploymentParameters`` variables are sourced at deployment time
  from the :ref:`Deployment Parameters` referenced in the :ref:`configuration <Configuration>`
  selected by the user.

.. _manifest-inputs:

inputs (Optional)
  A list of the inputs required by the Application.

  In this particular case the `disk_image` (also called **image name**) to be
  used when creating the virtual machine. Inputs should preferred over
  :ref:`deploymentParameters <manifest-deploymentParameters>` when their value needs to *change*
  at each deployment.
  In our case, the base disk will be different each time the user wants to deploy
  a different OS (CentOS, Ubuntu, BioLinux,...) so it makes sense to keep it as input.

  Input fields will be shown by the |project_name| for each of the `inputs`
  defined in the manifest to to allow users to customise the deployment behaviour.
  As for the :ref:`deploymentParameters <manifest-deploymentParameters>`, all the values will
  be injected as environment variables with the ``TF_VAR`` prefix.


outputs (Optional)
  A list of the outputs the Application wants to show to the user.

  A very common use case when deploying infrastructure to the cloud is the need
  to show back to the user some information resulting from the deployment itself,
  for example the external IP address of the VM that has just been deployed.

  The |project_name| will scan the output of the Terraform state file looking
  for the strings defined in this ``JSON`` array, and display the result to the user.

.. _manifest-volumes:

volumes (Optional)
  A list of the volumes the Application requires to work.

  Sometimes, a deployment requires attaching a previously defined volume.
  For example, some data may be staged in via a GridFTP server on a
  particular volume, that is then re-attached to an NFS server serving a
  batch system. The |project_name| allows to completely separate the
  volumes lifecycle from the lifecycle of applications. Adding a volume
  name (i.e. ``DATA_DISK_ID``) to volumes automatically displays a drop-down menu
  listing all the volumes deployed through the |project_name| on the deployment card.
  The id of the selected volume (as provided by the cloud provider,
  not the portal internal id!) is then injected into the deployment process as
  an environment variable (i.e. ``TF_VAR_DATA_DISK_ID`` in our example).

.. warning::
    Variables defined in :ref:`deploymentParameters <manifest-deploymentParameters>`,
    :ref:`inputs <manifest-inputs>` and :ref:`volumes <manifest-volumes>` will be injected by the
    |project_name| in the deployment environment prepended with the suffix ``TF_VAR_``
    to allow |terraform| to use them `directly <https://www.terraform.io/docs/configuration/variables.html#environment-variables.>`_.
    Keep this in mind when you're using these variables in Ansible!


Defining supported cloud providers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Each App can support one or more cloud providers, and this is defined by the
``cloudProviders`` list in the :ref:`manifest file <manifest-file>`. This key
is *required* in each manifest, and supported provider should be declared adding
a dictionary (or hash table, following the ``JSON`` nomenclature) to the ``cloudProviders``
list with the following schema:

::

  {
    "cloudProvider": "AWS",
    "path": "aws",
    "inputs": [
      "instance_type"
    ]
  }

Allowed keys in this dictionary are:

cloudProvider (Required)
  Specifies which cloud provider the dictionary specifies support for.

  This values is used to filter the :ref:`configurations <Configuration>` a user
  can pick when deploying this applications. It's thus *required* to follow the
  :ref:`nomenclature <cloud-providers>` defined earlier for this filtering to
  work as expected.

path (Required)
  Specifies the path to the folder containing the deployment code for the
  specified cloud provider.

  There is no restriction on the name these folders can have, and this is the
  very reason why this key exists, but for the sake of understandability we
  warmly suggest to use the string defined in our :ref:`nomenclature <cloud-providers>`
  for Cloud Providers in lowercase.

inputs (Optional)
  Specifies cloud provider specific inputs.

  These inputs will be only shown when the users decides to deploy the App in
  this cloud provider. The |project_name| will merge them with the
  :ref:`generic inputs <manifest-inputs>` and ask the user to provide values
  during the deployment process.

.. note::
    At the time of writing, the |project_name| doesn't support cloud provider
    specific :ref:`deploymentParameters <manifest-deploymentParameters>`.


Variables precedence
^^^^^^^^^^^^^^^^^^^^

If the same variable is defined both as a :ref:`deployment parameter <manifest-deploymentParameters>`
and as an :ref:`input <manifest-inputs>` (both generic or cloud-specific), **inputs**
will always take precedence. This allows to override what defined in a
:ref:`Deployment Parameters` on ad-hoc basis. However, this approach is *not* recommended
as it obscures the flow of information in your App.

.. _deployment-scripts:

Deployment scripts
~~~~~~~~~~~~~~~~~~

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
