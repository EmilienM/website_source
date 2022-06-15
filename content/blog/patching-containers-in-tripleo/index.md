---
title: Patching containers in TripleO
author: Emilien
type: post
date: 2022-02-15T19:25:37+00:00
url: /blog/patching-containers-in-tripleo/
featured_image: /static/images/surgery-tools.jpg
categories:
  - openstack
  - containers
  - buildah
  - tripleo

---
Read this post to learn more how to update a container in TripleO on a live system.

<!--more-->

**Note: this might sound _surgery_ but this is I think the clean options to patch container images in TripleO.**

Your TripleO cloud is running and you want to update an rpm in one or multiple containers?

TripleO provides some CLI to build new container images with the rpms that you want. This procedure is also documented [here][1].

In this particular example, we will update the **python3-networking-ovn** rpm on **octavia_api**.

1. You need a host to build the image:

The easiest place is the Undercloud or the Standalone node, where Buildah and tripleoclient are installed. We'll build the image from that host.

2. Put your rpms in a directory:

e.g. in **/tmp/rpms**

3. Export the OpenStack admin credentials: e.g.:

```
export OS_CLOUD=standalone
```

4. Login to the registry (when using OSP):

```
podman login registry.redhat.io
```

5. Build the new container image for octavia_api:

```
openstack tripleo container image hotfix \
    --image registry.redhat.io/rhosp-rhel8/openstack-octavia-api:16.2 \
    --rpms-path /tmp/hotfix \
    --tag 16.2-customfix \
```
 

You should see the new image by running **buildah images**.

Now you'll need to push the image to a registry (yours, or TripleO registry): e.g. :

```
buildah push registry.redhat.io/rhosp-rhel8/openstack-octavia-api:16.2-16.2-customfix docker://quay.io/emilien/openstack-octavia-api:16.2-customfix
```

Now, there are two methods for deploying that new image.

  * Run the deploy command again, after updating the **ContainerOctaviaApiImage** parameter in TripleO environment
  * Run the following steps:

You need to figure out what's the TripleO step where Octavia is deployed (it's step 4), by looking on the host in **/var/lib/tripleo-config/container-startup-config** and grep for **octavia_api**.

Now, go on the host where you want to use that new image (in the case of Standalone, it's the same host where you built the image) and create an Ansible playbook with this content (e.g. paunch.yaml):

```
- hosts: localhost
  become: true
  vars:
    service_name: octavia_api
  tasks:
  - name: Stop and clean the old container
    command: systemctl stop {{ service name }} && podman rm {{ service_name }}
  - name: Start containers for step 1
    paunch:
      config: /var/lib/tripleo-config/container-startup-config/step_4/hashed-{{ service_name }}.json
      config_overrides:
        octavia_api:
          image: quay.io/emilien/openstack-octavia-api:16.2-customfix"
      config_id: tripleo_step4
      cleanup: false
      action: apply
```

Change the content for your needs (different step, image, etc).

Run Ansible with:

```
ansible-playbook paunch.yaml
```

Your container is now running with your custom image (check with **podman inspect**).

For more details or help, check out the TripleO manuals or ask for help on IRC #tripleo (OFTC now).

 [1]: https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/deployment/container_image_prepare.html#building-hotfixed-containers
