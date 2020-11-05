

# Setup

## Configure an OpenShift 4.5+ cluster.
Provision an OpenShift 4.5 or later cluster following the default instructions. The cluster should have at least 3 worker nodes with a total of 18 CPU and 80G of memory or `m5.xlarge` on AWS EC2.

The following example `install-config.yaml` was used to prepare the Hub cluster for the demo:

### install-config.yaml
```bash
apiVersion: v1
baseDomain: SPECIFY_YOUR_BASE_DOMAIN
compute:
- hyperthreading: Enabled
  name: worker
  platform:
    aws:
      zones:
      - us-east-1b
      - us-east-1c
      - us-east-1d
      type: m5.xlarge
      rootVolume:
        iops: 4000
        size: 250
        type: io1
  replicas: 3
controlPlane:
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
metadata:
  name: SPECIFY_YOUR_CLUSTER_NAME
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineCIDR: 10.0.0.0/16
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: us-east-1
    userTags:
      owner: SPECIFY_YOUR_EMAIL_ADDRESS
      purpose: demo
publish: External
pullSecret: 'SPECIFY_YOUR_PULL_SECRET'
sshKey: |
  ssh-rsa SPECIFY_YOUR_SSH_RSA_PUBLIC_KEY

```
## Deploy the RHACM product hub.

The RHACM product can be deployed directly from the Red Hat Operator Catalog or by using the instructions documented at [github.com/open-cluster-management/deploy](https://github.com/open-cluster-management/deploy).

## Deploy Ansible Tower.

Detailed instructions on setting up the Ansible Tower container-based installation method are available at [docs.ansible.com](https://docs.ansible.com/ansible-tower/3.7.1/html/administration/openshift_configuration.html). The following steps capture the specific actions taken to prepare the demo captured in this repository.

1. Download the Ansible Tower installer release from the [available releases](https://releases.ansible.com/ansible-tower/setup_openshift/).

2. Create the `ansible-tower` project if necessary (the installer will create the project if the user credential has sufficient permissions):

    ```bash
    oc new-project ansible-tower
    ```

3. Configure a PersistentVolumeClaim against your OpenShift cluster (and ensure the correct name is reflected in your `inventory` file):

    ```yaml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
    name: postgresql
    namespace: ansible-tower
    spec:
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
        storage: 5Gi
    storageClassName: gp2
    volumeMode: Filesystem
    ```

4. Configure the inventory for your OpenShift cluster.

    The following example `inventory` can be placed under the root directory of the extracted release archive. Be sure to override:

    1. `SPECIFY_YOUR_OWN_PASSWORD`: Choose a strong password of at least 16 characters.
    2. `SPECIFY_YOUR_CLUSTER_ADDRESS`: Provide the correct hostname of the API server for your OpenShift cluster.
    3. `SPECIFY_YOUR_OPENSHIFT_CREDENTIALS`: The password for your OpenShift cluster admin user. Be sure to also override `kubeadmin` if you have defined an alternate administrative user.

    Example inventory:

    ```yaml
    localhost ansible_connection=local ansible_python_interpreter="/usr/bin/env python"
    [all:vars]
    # This will create or update a default admin (superuser) account in Tower
    admin_user=admin
    admin_password='SPECIFY_YOUR_OWN_PASSWORD'
    # Tower Secret key
    # It's *very* important that this stay the same between upgrades or you will lose
    # the ability to decrypt your credentials
    secret_key='SPECIFY_YOUR_OWN_PASSWORD'
    # Database Settings
    # =================
    # Set pg_hostname if you have an external postgres server, otherwise
    # a new postgres service will be created
    # pg_hostname=postgresql
    # If using an external database, provide your existing credentials.
    # If you choose to use the provided containerized Postgres depolyment, these
    # values will be used when provisioning the database.
    pg_username='admin'
    pg_password='SPECIFY_YOUR_OWN_PASSWORD'
    pg_database='tower'
    pg_port=5432
    pg_sslmode='prefer'  # set to 'verify-full' for client-side enforced SSL
    # Note: The user running this installer will need cluster-admin privileges.
    # Tower's job execution container requires running in privileged mode,
    # and a service account must be created for auto peer-discovery to work.
    # Deploy into Openshift
    # =====================
    openshift_host=https://api.SPECIFY_YOUR_CLUSTER_ADDRESS:6443
    openshift_skip_tls_verify=true
    openshift_project=tower
    openshift_user=kubeadmin
    openshift_password=SPECIFY_YOUR_OPENSHIFT_CREDENTIALS
    # If you don't want to hardcode a password here, just do:
    # ./setup_openshift.sh -e openshift_token=$TOKEN
    # Skip this section if you BYO database. This is only used when you want the
    # installer to deploy a containerized Postgres deployment inside of your
    # OpenShift cluster. This is only recommended if you have experience storing and
    # managing persistent data in containerized environments.
    #
    #
    # Name of a PVC you've already provisioned for database:
    openshift_pg_pvc_name=postgresql
    #
    # Or... use an emptyDir volume for the OpenShift Postgres pod.
    # Useful for demos or testing purposes.
    # openshift_pg_emptydir=true
    # Deploy into Vanilla Kubernetes
    # ==============================
    # kubernetes_context=test-cluster
    # kubernetes_namespace=ansible-tower
    ```

5. Update the default task image that is used to run the defined Jobs. Because the Jobs use additional modules, we need to ensure that various python module dependencies are available.

    In `group_vars/all`, update the following key:
    ```yaml
    kubernetes_task_image: quay.io/mdelder/ansible-tower-task
    ```

    You can build this image and consume it from your own registry by building the `Dockerfile.taskimage` under `hack/tower-setup/container_task_image`.

    OPTIONAL: Build the task image and publish to your own registry. Use the correct Ansible version based on the release you downloaded above.
    ```bash
    cd hack/tower-setup/container_task_image
    docker build -t quay.io/YOUR_USERID/ansible-tower-task:3.7.2 -f Dockerfile.taskimage
    docker push quay.io/YOUR_USERID/ansible-tower-task:3.7.2
    ```

6. Run the installer.

    ```bash
    ./setup_openshift.sh
    ```

7. Launch the Tower web console.

    ```bash
    open https://$(oc get route -n tower ansible-tower-web-svc -ojsonpath='{.status.ingress[0].host}')
    ```

    Login with the user and password that you specified in the `inventory` file above. You must then choose your license for Tower. If you have a Red Hat user identity, you can login and choose the 60-day evaluation license.


## Configure projects for ServiceNow and F5 Cloud DNS Load Balancer.

1. Create a file named `tower_cli.cfg` under `hack/tower-setup` with the following contents:

    ```
    [general]
    host = https://ansible-tower-web-svc-tower.apps.cluster.baseDomain
    verify_ssl = false
    #oauth_token = ALTERNATIVELY_USE_A_TOKEN
    username = admin
    password = SPECIFY_YOUR_OWN_PASSWORD
    ```

    If you're unsure of the host address for Tower, you can use `oc` to find the correct value:

    ```bash
    oc get route -n tower ansible-tower-web-svc -ojsonpath='{.status.ingress[0].host}'
    ```

2. You may need to install required python libraries.

    ```bash
    export PYTHON="/usr/local/Cellar/ansible/2.9.13/libexec/bin/python3.8"
    $PYTHON -m pip install --upgrade ansible-tower-cli
    ```

Run the playbook

    export PYTHON="/usr/local/Cellar/ansible/2.9.13/libexec/bin/python3.8"

    ansible-playbook -e ansible_python_interpreter="$PYTHON" tower-setup.yml

## Deploy AWX Resource Operator.


References

https://docs.ansible.com/ansible/latest/collections/awx/awx/tower_project_module.html
https://github.com/ansible/awx
https://github.com/ansible/awx/tree/devel/awx_collection#running