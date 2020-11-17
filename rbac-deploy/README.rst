Deploy RBAC authorization for Confluent Platform
================================================

In this workflow scenario, you'll set up secure Confluent Platform clusters with
SASL_PLAIN authentication, RBAC authorization, and inter-component TLS.

Before you begin this tutorial:

* Clone the Tutorial repo.

To complete this scenario, you'll follow these steps:

#. Set up a Kubernetes cluster for this tutorial.

#. Deploy Confluent Operator.

#. Deploy Confluent Platform.

#. Tear down Confluent Platform.

=========================
Deploy Confluent Operator
=========================

The assumption is that you’ve set up Early Access credentials following `the
instruction
<https://github.com/confluentinc/operator-earlyaccess/blob/master/README.rst>`__.

#. Install Confluent Operator using Helm:

   ::

     helm upgrade --install operator confluentinc/confluent-operator
  
#. Check that the Confluent Operator pod comes up and is running:

   ::
     
     kubectl get pods

===============
Deploy OpenLDAP
===============

This repo includes a Helm chart for OpenLdap (https://github.com/osixia/docker-openldap) 
The chart values.yaml includes the set of principal definitions that are needed for Confluent Platform to run with RBAC.

#. Deploy OpenLdap

   ::

     helm upgrade --install -f ./openldap/ldaps-rbac.yaml test-ldap ./openldap --namespace confluent

#. Validate that OpenLDAP is running. Log in to the pod

   ::

     kubectl exec -it ldap-0 bash -n confluent

#. Run LDAP search command

   ::

     ldapsearch -LLL -x -H ldap://ldap.operator.svc.cluster.local:389 -b 'dc=test,dc=com' -D "cn=mds,dc=test,dc=com" -w 'Developer!'

============================
Deploy configuration secrets
============================

You'll use Kubernetes secrets to provide credential configurations.

With Kubernetes secrets, credential management (defining, configuring, updating)
can be done outside of the Confluent Operator. You define the configuration
secret, and then tell Confluent Operator where to find the configuration.

In this tutorial, you will deploy a secure Zookeeper, Kafka and Control Center,
and the rest of Confluent Platform components as shown below:

.. figure:: ../images/confluent-platform-security-deployment.png
   :width: 200px
   
To support the above deployment scenario, you need to provide the following
credentials:

* Root Certificate Authority to auto-generate certificates

* Authentication credentials for Zookeeper, Kafka, and Control Center

Provide a Root Certificate Authority
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Confluent Operator provides auto-generated certificates for Confluent Platform
components to use for inter-component TLS. You'll need to generate and provide a
Root Certificate Authority (CA).

#. Generate a CA pair to use in this tutorial:

   ::

     openssl genrsa -out $TUTORIAL_HOME/ca-key.pem 2048
    
   ::

     openssl req -new -key $TUTORIAL_HOME/ca-key.pem -x509 \
       -days 1000 \
       -out $TUTORIAL_HOME/ca.pem \
       -subj "/C=US/ST=CA/L=MountainView/O=Confluent/OU=Opeator/CN=TestCA"

#. Create a Kuebernetes secret for inter-component TLS:

   ::

     kubectl create secret tls ca-pair-sslcerts -n confluent \
       --cert=$TUTORIAL_HOME/ca.pem \
       --key=$TUTORIAL_HOME/ca-key.pem
  
Provide authentication credentials
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Create a Kubernetes secret object for Zookeeper, Kafka, and Control Center. This
secret object contains file based properties. These files are in the format that
each respective Confluent component requires for authentication credentials.

   ::

     kubectl create secret generic credential -n confluent \
      --from-file=plain-users.json=$TUTORIAL_HOME/creds-kafka-sasl-users.json \
      --from-file=digest-users.json=$TUTORIAL_HOME/creds-zookeeper-sasl-digest-users.json \
      --from-file=digest.txt=$TUTORIAL_HOME/creds-kafka-zookeeper-credentials.txt \
      --from-file=plain.txt=$TUTORIAL_HOME/creds-client-kafka-sasl-user.txt \
      --from-file=basic.txt=$TUTORIAL_HOME/creds-control-center-users.txt \
      --from-file=ldap.txt=$TUTORIAL_HOME/ldap.txt

In this tutorial, we use one credential for authenticating all client and server
communication to Kafka brokers. In production scenarios, you'll want to specify
different credentials for each of them.

Create Kubernetes secret objects for MDS.

   ::

     kubectl create secret generic mds-token --from-file=mdsPublicKey.pem=$TUTORIAL_HOME/mds-publickey.txt --from-file=mdsTokenKeyPair.pem=$TUTORIAL_HOME/mds-tokenkeypair.txt
  
   ::

     kubectl create secret generic mds-client --from-file=bearer.txt=$TUTORIAL_HOME/bearer.txt

=========================
Deploy Confluent Platform
=========================

#. Deploy Confluent Platform with the above configuration:

   ::

     kubectl apply -f $TUTORIAL_HOME/confluent-platform-rbac-secure.yaml

#. Check that all Confluent Platform resources are deployed:

   ::
   
     kubectl get confluent

#. Get the status of any component. For example, to check Control Center:

   ::
   
     kubectl describe controlcenter

=======================
Configure Role Bindings
=======================

#. Set up port forwarding

   ::
   
     kubectl -n confluent port-forward kafka-0 8090:8091

#. Set up DNS access from your local machine

   ::
   
     vi /etc/hosts
     # Add Kafka URL <> localhost mapping
     127.0.0.1	kafka.confluent.svc.cluster.local

#. MDS commands

   ::
   
     confluent login --url https://kafka.confluent.svc.cluster.local:8090 --ca-cert-path $TUTORIAL_HOME/ca.pem
     # Log in with superuser credentials - user: kafka and pass: kafka-secret

#. Get Kafka cluster id

   ::
   
     curl -ik https://kafka.confluent.svc.cluster.local:8090/v1/metadata/id 
     # Take the id value and set an environment variable as below:
     export KAFKA_ID=____

#. Create Control Center Role Binding

   ::
     confluent iam rolebinding create \
     --principal User:c3 \
     --role SystemAdmin \
     --kafka-cluster-id $KAFKA_ID

========
Validate
========

Validate in Control Center
^^^^^^^^^^^^^^^^^^^^^^^^^^

Use Control Center to monitor the Confluent Platform, and see the created topic
and data.

#. Set up port forwarding to Control Center web UI from local machine:

   ::

     kubectl port-forward controlcenter-0 9021:9021

#. Browse to Control Center and log in as admin with the ``Developer1`` password:

   ::
   
     https://localhost:9021

=========
Tear down
=========

::

  kubectl delete -f confluent-platform-rbac-secure.yaml

::

  kubectl delete secret mds-token
  kubectl delete secret mds-client

::

  kubectl delete secret credential

::

  kubectl delete secret ca-pair-sslcerts

::

  helm delete operator
  
::
  
  kubectl delete ns confluent