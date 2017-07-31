# Multi-Node AWS Openshift Deployment 
OpenShift environment with the Service Catalog & Ansible Service Broker configured in AWS with multiple EC-2 instances

## Overview
These playbooks will:
  * Create a public VPC if it does not exist
  * Create a security group if it does not exist
  * Create a configurable number of instances with a specific Name if does not exist
  * Configure a hostname with as a CNAME record through Route53
  * Setup OpenShift via the [Openshift Ansible BYO playbook](https://github.com/openshift/openshift-ansible/blob/master/playbooks/byo/config.yml)
  * Install Service Catalog 
  * Install Ansible Service Broker
  * Configure cluster users with htpasswd authentication

## Pre-Reqs
  * Ansible needs to be installed so its source code is available to Python.
    * Check to see if Ansible modules are available to Python
      ```bash
      $ python -c "import ansible;print(ansible.__version__)"
      2.3.0.0
      ```
    * MacOS requires Ansible to be installed from `pip` and not `brew`
      ```bash
      $ python -c "import ansible;print(ansible.__version__)"
      Traceback (most recent call last):
      File "<string>", line 1, in <module>
      ImportError: No module named ansible

      brew uninstall ansible
      pip install ansible

      $ python -c "import ansible;print(ansible.__version__)"
      2.3.0.0
      ```
  * Install python dependencies (This is needed for python2. Use pip3 if using python3)
    * On Fedora and EL7 it is recommended that you use ansible in a python virtualenv.
     * This is due to a couple reasons:
       - boto rpms are not sufficiently new enough
       - pip is not sudo safe on Fedora and EL7 
     * To setup and activate a virtualenv do the following;
     ```
     sudo dnf install python-virtualenv #or EL7: sudo yum install python-virtualenv 
     virtualenv /tmp/ansible
     source /tmp/ansible/bin/activate
     pip install ansible
     ```
   * Continue with the next step:
    ```bash
    $ pip install boto boto3 six
    ```
  * Configure a SSH Key in your AWS EC-2 account for the given region
  * Create a hosted zone in Route53
  * Set these environment variables:
    ```bash
    AWS_ACCESS_KEY_ID
    AWS_SECRET_ACCESS_KEY
    AWS_SSH_PRIV_KEY_PATH  - Path to your private ssh key to use for the ec2 instances
    ```
## Configuration
### Default Configuration
The default configuration will install Openshift Enterprise v3.6 from the RH CDN Repository with 1 master, and 2 nodes.

### Node List
The node list can be specified by the [`ec2_hosts.yml`](ec2_hosts.yml) file. You may specify more than 2 nodes, but only 1 master node is supported at this time.

Also, a 'wildcard_dns_host' will need to be specified in the [ec2_hosts.yml](ec2_hosts.yml) file, which will be used to set the wildcard DNS record in Route53 for the cluster.  Currently, the `wildcard_dns_host` must equal the name of the master node

### Using the latest (nightly) built packages
If `enable_ops_mirror_repo` or the environment variable `ENABLE_OPS_MIRROR_REPO` is set to `true`, the scripts can be configured to use the latest built packages vs their official released counter parts.  

If these variables are set to `true`, you MUST have access to the `aos-ansible` private repository files locally on the machine where you're running the ansible playbooks from.  The repo path is defined in the `ops_mirror_dir` variable, which defaults to `/git/aos-ansible`.

### 'my_vars.yml' file
All configurable variables are contained in the [`all.yml`](../../ansible/group_vars/all.yml) under the [`group_vars`](../../ansible/group_vars/) folder. Any of these may be overwritten, if they're specified in the *my_vars.yml* file.

Review the example [`my_vars.yml.example`](../../config/my_vars.yml.example) file, and make a copy of it as `my_vars.yml` in the same location.  The following are some of the example values that you may want to have in your `my_vars.yml`
  * Use the latest packages
    ```bash
    enable_ops_mirror_repo: true
    enable_rh_cdn_repo: false
    ```
  * OPS Mirror repo path
    ```bash
    ops_mirror_dir: /path/to/aos-ansible
    ```
  * RHN Credentials 
    ```bash
    rhn_user: <rhn_username>
    rhn_passwd: <password>
    aos_pool_id: <OpenShift Container Platform subscription pool ID>
    ```
  * Specify the 'aws_custom_prefix'
    The 'aws_custom_prefix' is the string that names and tags all your AWS resources related to the cluster, and keeps them unique within a AWS user account.  Default value is your 'username'. However, ff you whish to deploy multiple cluster setups, changing the 'aws_custom_prefix' will allow you to create multiple cluster deployments, each with its own unique set of names.
    ```bash
    aws_custom_prefix: my_cluster1 # mycluster2, mycluster3, etc.
    ```

## Execute
  * Navigate to the [`config`](../../config) folder
    ```bash
    $ cd catasb/config
    ```
  * Edit the variables file [`ec2_env_vars`](../../config/ec2_env_vars)
    * Note the following and update:
      ```bash
      AWS_SSH_KEY_NAME="splice"
      TARGET_DNS_ZONE="ec2.dog8code.com"
      ```
      Needs to match a hosted zone entry in your Route53 account, we will create a subdomain under it for the ec2 instance
  * create a `my_vars.yml` as described [above](#configuration)
  
  * Create your Cluster
    ```bash
    $ ./create_cluster.sh
    ```
  * Open a Web Browser
    * Visit: `https://master.<AWS_CUSTOM_PREFIX>.ec2.dog8code.com:8443`
      * Accept the SSL certificate for the apiserver-service-catalog endpoint
      * Note: must accept the new SSL cert, each time you create your OpenShift environment
    * Visit: `https://<USERNAME>.ec2.dog8code.com:8443`
      * Where `<USERNAME>` is the value of `whoami` when you launched `run_setup_environment.sh`

### Cleanup
* To terminate the ec2 instance and cleanup the associated EBS volumes run the below
  ```bash
  $ ./terminate_instance.sh
  ```

### Tested with
  * ansible 2.3.0.0
    * Problems were seen using ansible 2.2 and lower