# Configuring OpenShift node as an IPv6 router in an O-RAN LLS-C1 configuration

## Author
Javier Peña (@fj_pena).

## Introduction

When building a 5G network, operators can choose from different configurations to combine the components that form the solution. We are going to discuss the requirements of the LLS-C1 configuration, when used in a pure IPv6 environment. In that case, if the Distributed Unit is running on Red Hat OpenShift Container Platform, we may need to add some components to the solution if we want to allow the Radio Unit to communicate with the rest of the network.

## LLS-C1 configuration

The O-RAN Alliance defines several synchronization modes for the fronthaul network. In this post, we will be discussing the LLS-C1 configuration (see section 11.2.2.2 of the O-RAN.WG4.CUS.0-v09.00 document), depicted by the following diagram:


<img src="https://raw.githubusercontent.com/javierpena/blogposts/main/radvd/images/LLS-C1 Configuration.png">

This configuration is a simple topology for network timing, where the Radio Unit (O-RU), connected to the 5G antenna, is directly connected to the Distributed Unit (O-DU) via a point-to-point network cable. When using IPv6, if the Radio Unit needs to connect to a system other than the Distributed Unit, that DU will need to act as an IPv6 router. Since those Distributed Units can be running on a container platform like Red Hat OpenShift Container Platform, we need to make sure the node running the O-DU will be able to act as an IPv6 router for the Radio Unit.

## IPv6 addressing

There are several ways to dynamically configure an IPv6 address for a host, namely SLAAC (defined by RFC 4862), stateless DHCPv6 (defined by RFC 3736, updated by RFC 8415) and stateful DHCPv6 (RFC 8415). 

A common requirement for all methods is that the IPv6 router must send a Router Advertisement (RA) message, specifying the network prefix for its subnet, the default gateway, and some optional flags. Unlike in IPv4, this means that we need two separate components to provide all addressing options to a node if using DHCP:

- The DHCPv6 server
- A router service providing RAs

In Red Hat Enterprise Linux, RAs are provided by the radvd package, as described in the [official documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/managing_networking_infrastructure_services/providing-dhcp-services_networking-infrastructure-services#the-differences-when-using-dhcpd-for-dhcpv4-and-dhcpv6_providing-dhcp-services). However, this package is not available as part of the CoreOS distribution, so we need to find a way to run it if we need to use our OpenShift node as a router for this kind of IPv6 environment.

## Providing router advertisements in Red Hat OpenShift Container Platform

To use radvd in a worker node, we suggest running it as a pod inside the OCP cluster. To achieve this, first we need to create a container image including it. Then, we can create a container with the appropriate configuration and privileges, required to access the network cards on the node. See the diagram below.

<img src="https://raw.githubusercontent.com/javierpena/blogposts/main/radvd/images/Using radvd in a container.png">

### Create the container image

1. On a node running Red Hat Enterprise Linux with a valid subscription, create the following files:
   - Containerfile
     ```
     FROM registry.access.redhat.com/ubi8/ubi:latest

     RUN dnf install -y radvd && \
        mkdir /etc/radvd && chmod 755 /etc/radvd && \
        rm -rf /var/cache/{yum,dnf}
     ADD radvd.sh /
     RUN chmod 755 /radvd.sh

     CMD ["/radvd.sh"]
     ```

   - radvd.sh
     ```
     #!/bin/bash

     /usr/sbin/radvd -C /etc/radvd/radvd.conf -p /run/radvd.pid -n -m stderr -u radvd
     ```

2. Build and push the container to a container registry. If you are uploading the container image to your own registry, make sure to replace “quay.io/user/radvd:2.17-15” with the appropriate URI.

    ```
    $ podman build -t radvd:2.17-15 .
    $ podman push radvd:2.17-15 quay.io/user/radvd:2.17-15
    ```

### Run the container

1. Create a configmap file with the radvd configuration file (this is just an example, and needs to be adapted to the specific environment, like network card or IPv6 prefix).

    ```
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: radvd-conf
    data:
      radvd.conf: |
      interface ens4f0
      {
            AdvSendAdvert on;
            AdvManagedFlag on;
            AdvOtherConfigFlag on;
            MinRtrAdvInterval 30;
            MaxRtrAdvInterval 100;
            prefix 2600:42:7:15::/64
            {
              AdvOnLink on;
              AdvAutonomous on;
              AdvRouterAddr off;
            };
          };
    ```

2. Create the deployment YAML file, which also includes creation of a service account and assignment of the required permissions to the service account.

   Please note that this is assuming we have a Single-Node OpenShift (SNO) environment. If you have a different environment, you may want to use a different resource, such as a DaemonSet with a hostname selector.

   Make sure to replace “quay.io/user/radvd:2.17-15” with the appropriate URI for the container image.
    ```
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: radvd-sa
    automountServiceAccountToken: false
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: system:openshift:scc:privileged
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: system:openshift:scc:privileged
    subjects:
    - kind: ServiceAccount
      name: radvd-sa
      namespace: radvd
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
      app: radvd
      name: radvd
    spec:
      replicas: 1
      selector:
      matchLabels:
        app: radvd
      template:
      metadata:
            labels:
              app: radvd
      spec:
            serviceAccountName: radvd-sa
            hostNetwork: true
          containers:
            - image: quay.io/user/radvd:2.7-15
              imagePullPolicy: IfNotPresent
              name: radvd
              volumeMounts:
              - name: config-volume
                  mountPath: /etc/radvd
              resources:
              limits:
                  cpu: 0.5
                  memory: 512Mi
              securityContext:
              capabilities:
                  add:
                    - NET_RAW
              restartPolicy: Always
        volumes:
          - name: config-volume
            configMap:
              name: radvd-conf
    ```

    Note that the deployment spec has two special requirements:

    - The NET_RAW capability for the pod, required by radvd
    - Host network access, so the radvd process can access the network cards in the OpenShift node.

3. Create the configmap and deployment resources from the YAML files:

    ```
    $ oc apply -f configmap.yml
    $ oc apply -f deployment.yml
    ```


## Summary and conclusions

While OpenShift nodes are not meant to act as routers, we can configure them as such in certain situations. For an Open RAN LLS-C1 5G environment, we can set up radvd running on a container to provide IPv6 router advertisements, if required by the environment.


## Acknowledgements

- John Williams and Lazhar Halleb, for their reviews and input on this post.
