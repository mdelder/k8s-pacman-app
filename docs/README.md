

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

## Deploy Ansible Tower and the Ansible Resource Operator

Detailed instructions on setting up the Ansible Tower container-based installation method are available at [docs.ansible.com](https://docs.ansible.com/ansible-tower/3.7.1/html/administration/openshift_configuration.html). The following steps capture the specific actions taken to prepare the demo captured in this repository.

1. Apply the policies under `hack/manifests/policies`.

  ```bash
  cd hack/manifests
  oc apply -f ansible-tower-policies-subscription.yaml
  ```

  The result of these policies will setup the (a) Ansible Resource Operator, (b) prepare the Ansible Tower `Project` named `tower` and (c) create a `PeristentVolumeClaim` using the default storage class to support Ansible Tower's database (PostgresSQL).

  After a few moments, verify the resources were correctly applied on your Hub cluster. You can view the Policies `policy-ansible-tower-prep` and `policy-auth-provider` are Compliant from your RHACM web console (under "Govern Risk"). You can also verify the resources that were created as follows:

  ```bash
  oc get subs.operators --all-namespaces
  NAMESPACE                 NAME                        PACKAGE                       SOURCE                CHANNEL
  open-cluster-management   acm-operator-subscription   advanced-cluster-management   acm-custom-registry   release-2.1
  tower-resource-operator   awx-resource-operator       awx-resource-operator         redhat-operators      release-0.1

  oc get pvc -n tower
  NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  postgresql   Bound    pvc-1554a179-0947-4a65-9af0-81c5f2d8b476   5Gi        RWO            gp2            3d20h
  ```
2. Download the Ansible Tower installer release from the [available releases](https://releases.ansible.com/ansible-tower/setup_openshift/). Extract the archive into a working directory.

3. Configure the inventory for your OpenShift cluster. The `inventory` file should be placed directly under the folder for the archive (e.g. `ansible-tower-openshift-setup-3.7.2/inventory`).

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

4. Update the default task image that is used to run the defined Jobs. Because the Jobs use additional modules, we need to ensure that various python module dependencies are available.

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

5. Run the installer.

    ```bash
    ./setup_openshift.sh
    ```

6. Launch the Tower web console.

    ```bash
    open https://$(oc get route -n tower ansible-tower-web-svc -ojsonpath='{.status.ingress[0].host}')
    ```

    Login with the user and password that you specified in the `inventory` file above. You must then choose your license for Tower. If you have a Red Hat user identity, you can login and choose the 60-day evaluation license.

7. Optionally, cusomtize the `hack/manifests/ansible-tower-console-link.yaml` for your own cluster. Then apply the file (NOTE: You must update the URL within the file before this will work in your cluster).

    ```bash
    cd hack/manifests
    oc apply ansible-tower-console-link.yaml
    ```

    After applying the `ConsoleLink`, refresh your OpenShift web console and view the shortcut to your Ansible Tower under the "Applications" drop down in the header.

## Configure projects for ServiceNow and F5 Cloud DNS Load Balancer.

The example application uses the F5 Cloud DNS Load Balancer service and ServiceNow to demonstrate Ansible automation.

The following steps assume that you have:

  - **Created a developer instance of ServiceNow.** If you need a developer instance of ServiceNow, follow the directions at https://developer.servicenow.com/.
  - **Created an account with the F5 Cloud DNS Load Balancer service.** If you need to create an account with F5 Cloud DNS Load Balancer SaaS, you can do this through the [AWS Marketplace](https://aws.amazon.com/marketplace/pp/F5-Networks-F5-DNS-Load-Balancer-Cloud-Service/B07W3P8HM4).
  - **Delegated your global domain to the F5 DNS Nameservers**. Created an `NS` delegating DNS record with the global domain that you will use with F5. You can do this via Route53 or your DNS provider. The [F5 DNS Load Balancer Cloud Service FAQ](https://clouddocs.f5.com/cloud-services/latest/f5-cloud-services-GSLB-FAQ.html) answers questions related to this prerequisite.

Once you have the credentials for the services above, you can configure the Ansible Tower instance that you deployed above with the two relevant Ansible Projects that provide the Job Templates that will be executed as part of the prehook and posthooks that run when the application is placed or removed on a cluster.

1. Create a file named `tower_cli.cfg` under `hack/tower-setup` with the following contents:

    ```bash
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

2. Create a file named `credentials.yml` under `hack/tower-setup/group_vars` with the following contents:

    ```yaml
    # User and password for the F5 CloudServices account.
    f5aas_username: SPECIFY_YOUR_F5_USERNAME
    f5aas_password: SPECIFY_YOUR_F5_PASSWORD

    # Credentials for ServiceNow
    snow_username: admin
    snow_password: SPECIFY_YOUR_SERVICENOW_USERNAME
    # Specify your ServiceNow developer instance ID.
    snow_instance: devXXXXX
    ```

3. You may need to install required python libraries.

    ```bash
    # Optionally specify the correct version of python required by Ansible
    # Of course, you must update the PYTHON var specific to your environment
    export PYTHON="/usr/local/Cellar/ansible/2.9.13/libexec/bin/python3.8"
    $PYTHON -m pip install --upgrade ansible-tower-cli
    ```

4. Run the playbook

    ```bash
    # Optionally specify the correct version of python required by Ansible
    # Of course, you must update the PYTHON var specific to your environment
    export PYTHON="/usr/local/Cellar/ansible/2.9.13/libexec/bin/python3.8"
    ansible-playbook -e ansible_python_interpreter="$PYTHON" tower-setup.yml
    ```

## Configure toweraccess Secret and create Ansible Tower token

From Ansible Tower, create an authorization token. The authorization token will be used in a `Secret` that the application will reference to invoke the Ansible Tower Jobs.

- Login to the Ansible Tower instance.

    ```bash
    open https://$(oc get route -n tower ansible-tower-web-svc -ojsonpath='{.status.ingress[0].host}')
    ```

- Click on the "admin" user in the header.
- Click on "Tokens".
- Click on the "+"
- Set the scope to "Write"
- Click "Save" and **BE SURE** to copy and save the value of the token.
- Create a file named `hack/manifests/toweraccess-secret.yaml` with the following contents:

```yaml
apiVersion: v1
stringData:
  host: ansible-tower-web-svc-tower.apps.SPECIFY_YOUR_CLUSTER_NAME.SPECIFY_YOUR_BASE_DOMAIN
  token: SPECIFY_YOUR_ANSIBLE_TOWER_ADMIN_TOKEN
kind: Secret
metadata:
  name: toweraccess
  namespace: pacman-app
type: Opaque
```

## Deploy the pacman-app example to your cluster

- Create the `Project` for the application and apply the `Secret`:
```bash
oc new-project pacman-app
oc apply -f hack/manifests/toweraccess-secret.yaml
```
- Now you can create the `pacman-app` from the New Application wizard in the RHACM web console.

# References

- [github.com/open-cluster-management/deploy](https://github.com/open-cluster-management/deploy)
- [Ansible Tower containerized install method](https://releases.ansible.com/ansible-tower/setup_openshift/)
- https://docs.ansible.com/ansible/latest/collections/awx/awx/tower_project_module.html
- https://github.com/ansible/awx
- https://github.com/ansible/awx/tree/devel/awx_collection#running
- https://developer.servicenow.com/
- [AWS Marketplace](https://aws.amazon.com/marketplace/pp/F5-Networks-F5-DNS-Load-Balancer-Cloud-Service/B07W3P8HM4)