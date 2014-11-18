.. _docs-cloud:

Amazon Web Services
-------------------

`Amazon Web Services (AWS) <https://aws.amazon.com/>`_ provides a flexible cloud
based environment for running analyses. Cloud approaches offer the ability to
perform analyses at scale with no investment in local hardware. They also offer
full programmatic control over the environment, allowing us to automate the
entire setup, run and teardown process.

`bcbio-vm <https://github.com/chapmanb/bcbio-nextgen-vm>`_ provides a wrapper
around bcbio-nextgen that automates interaction with AWS and `Docker
<https://www.docker.com/>`_. ``bcbio_vm.py`` also cleans up the command line
usage to make it more intuitive so provides the same functionality as
``bcbio_nextgen.py`` but with additional tools.

bcbio uses `Elasticluster <https://github.com/gc3-uzh-ch/elasticluster>`_,
to build a cluster on AWS with an optional Lustre shared filesystem.

AWS support is still a work in progress and the current approach only supports
GRCh37 analyses. We welcome feedback and will continue to improve.

Local setup
===========

``bcbio_vm.py`` provides the automation to start up and administer remote bcbio
runs on AWS. This only requires the python wrapper code, not any of the Docker
containers or biological data, which will all get installed on AWS. The easier
way to install is using `conda`_ with an isolated Python::

    wget http://repo.continuum.io/miniconda/Miniconda-latest-Linux-x86_64.sh
    bash Miniconda-latest-Linux-x86_64.sh -b -p ~/install/bcbio-vm/anaconda
    ~/install/bcbio-vm/anaconda/bin/conda install --yes -c https://conda.binstar.org/bcbio bcbio-nextgen-vm
    ln -s ~/install/bcbio-vm/anaconda/bin/bcbio_vm.py /usr/local/bin/bcbio_vm.py

.. _conda: http://conda.pydata.org/

Data preparation
================

The easiest way to organize an AWS project is using an `S3 bucket
<http://aws.amazon.com/s3/>`_. Create a unique bucket for your analysis and
upload fastq, BAM and, optionally, a region BED file. You can do this using the
`AWS S3 web console <https://console.aws.amazon.com/s3/>`_,
`the AWS cli client <http://aws.amazon.com/cli/>`_ or specialized tools
like `gof3r <https://github.com/rlmcpherson/s3gof3r>`_.

You will also need a template file describing the type of run to do and a CSV
file mapping samples in the bucket to names and any other metadata. See the
:ref:`automated-sample-config` docs for more details about these files. Also
upload both of these files to S3.

With that in place, prepare and upload the final configuration to S3 with::

    bcbio_vm.py template s3://your-project/template.yaml s3://your-project/name.csv

This will find the input files in the ``s3://your-project`` bucket, associate
fastq and BAM files with the right samples, and add a found BED files as
``variant_regions`` in the configuration. It will then upload the final
configuration back to S3 as ``s3://your-project/name.yaml``, which you can run
directly from a bcbio cluster on AWS.

Extra software
~~~~~~~~~~~~~~

We're not able to automatically install some useful tools in pre-built docker
containers due to licensing restrictions. Variant calling with GATK requires a
manual download from the `GATK download`_ site for academic users.  Appistry
provides `a distribution of GATK for commercial users`_. Commercial users also
need a license for somatic calling with muTect. To make these jars available,
upload them to the S3 bucket in a ``jars`` directory. bcbio will automatically
include the correct GATK and muTect directives during your run.  You can also
manually specify the path to the jars using the global ``resources`` section
of your input sample YAML file::

    resources:
      gatk:
        jar: s3://bcbio-syn3-eval/jars/GenomeAnalysisTK.jar

.. _GATK download: http://www.broadinstitute.org/gatk/download
.. _a distribution of GATK for commercial users: http://www.appistry.com/gatk

AWS setup
=========

The first time running bcbio on AWS you'll need to setup permissions, VPCs and
local configuration files. We provide commands to automate all these steps and once
finished, they can be re-used for subsequent runs. To start you'll need to have
an account at Amazon and your Access Key ID and Secret Key ID from the
`AWS security credentials page
<https://console.aws.amazon.com/iam/home?#security_credential>`_. These can be
`IAM credentials <https://aws.amazon.com/iam/getting-started/>`_ instead of root
credentials as long as they have administrator privileges. Make them available
to bcbio using the standard environmental variables::

  export AWS_ACCESS_KEY_ID=your_access_key
  export AWS_SECRET_ACCESS_KEY=your_secret_key

With this in place, two commands setup your elasticluster and AWS environment to
run a bcbio cluster. The first creates public/private keys, a bcbio IAM user,
and sets up your elasticluster config in ``~/.bcbio/elasticluster/config``::

  bcbio_vm.py aws iam

The second configures a VPC to host bcbio::

  bcbio_vm.py aws vpc

Running a cluster
=================

Following this setup, you're ready to run a bcbio cluster on AWS. We start
from a standard Ubuntu AMI, installing all software for bcbio and the cluster as
part of the boot process.

The ``~/.bcbio/elasticluster/config`` file defines the number of compute nodes
to start. If you set up your AWS configuration manually, the bcbio-vm GitHub
repository has the `latest example configuration
<https://github.com/chapmanb/bcbio-nextgen-vm/blob/master/elasticluster/config>`_.

To start a cluster with a SLURM manager front end node and 2 compute nodes::

    [cluster/bcbio]
    setup_provider=ansible-slurm
    frontend_nodes=1
    compute_nodes=2
    flavor=r3.8xlarge

    [cluster/bcbio/frontend]
    flavor=m3.large
    root_volume_size=100
    root_volume_type=gp2

To start a single machine without a cluster to compute directly on::

    [cluster/bcbio]
    setup_provider=ansible
    frontend_nodes=1
    compute_nodes=0
    flavor=m3.large

    [cluster/bcbio/frontend]
    flavor=m3.2xlarge
    root_volume_size=100
    root_volume_type=gp2

Adjust the number of nodes, machine size flavors and root volume size as
desired. Elasticluster mounts the frontend root volume across all machines using
NFS. At scale, you can replace this with a Lustre shared filesystem. See below
for details on launching and attaching this to a cluster.

Once customized, start the cluster with::

    bcbio_vm.py elasticluster start bcbio

The cluster will take five to ten minutes to start. Once running,
install the bcbio wrapper code, Dockerized tools and system configuration
with::

    bcbio_vm.py aws bcbio bootstrap

Running Lustre
==============

Elasticluster mounts the cluster frontend root volume ``/home`` directory as a
NFS share available across all of the worker machines. You can use this as a
processing directory for smaller runs but for larger runs will need a
distributed file system. bcbio supports using `Intel Cloud Edition for Lustre (ICEL) <https://wiki.hpdd.intel.com/display/PUB/Intel+Cloud+Edition+for+Lustre*+Software>`_
to set up a Lustre scratch filesystem on AWS.

- Subscribe to `ICEL in the Amazon Marketplace
  <https://aws.amazon.com/marketplace/pp/B00GK6D19A>`_.

- By default, the Lustre filesystem will be 2TB and will be accessible to
  all hosts in the VPC. Creation takes about ten minutes and can happen in
  parallel while elasticluster sets up the cluster. Start the stack::

    bcbio_vm.py aws icel create

- Once the ICEL stack and elasticluster cluster are both running, mount the
  filesystem on the cluster::

    bcbio_vm.py aws icel mount

- The cluster instances will reboot with the Lustre filesystem mounted.

Running an analysis
===================

To run the analysis, connect to the head node with::

    bcbio_vm.py elasticluster ssh bcbio

If you started a single machine without a cluster run with::

    mkdir ~/run/your-project
    cd !$ && mkdir work && cd work
    bcbio_vm.py run -n 8 s3://your-project/name.yaml

Where the ``-n`` argument should be the number of cores on the machine.

To run on a full cluster with a Lustre filesystem::

    sudo mkdir /scratch/cancer-dream-syn3-exome
    sudo chown ubuntu !$
    cd !$ && mkdir work && cd work
    bcbio_vm.py ipythonprep s3://your-project/name.yaml \
                            slurm cloud -r 'mincores=30' -r 'timelimit=2-00:00:00' -n 60
    sbatch bcbio_submit.sh

Where 30 is the cores per node on the worker machines (minus 2 to account for
the base bcbio_vm script and IPython controller) and 60 is the total number of
cores across all the worker nodes.

On successful completion, bcbio uploads the results of the analysis back into your s3
bucket as ``s3://your-project/final``. You can now cleanup the cluster and
Lustre filesystem.

Shutting down
=============

The bcbio Elasticluster and Lustre integration can spin up a lot of AWS
resources. You'll be paying for these by the hour so you want to clean them up
when you finish running your analysis. To stop the cluster::

    bcbio_vm.py elasticluster stop bcbio

To remove the Lustre stack::

    bcbio_vm.py aws icel stop

Double check that all instances have been properly stopped by looking in the AWS
console.