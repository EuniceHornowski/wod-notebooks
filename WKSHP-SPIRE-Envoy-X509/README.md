![HPEDEVlogo](Pictures/hpe-dev-logo.png)

# Welcome to the Hack Shack
Powered by [HPE DEV Team](https://hpedev.io)

<p align="center">
  <img src="Pictures/hackshackdisco.png">
  
</p>

# Workshops-on-Demand

# Creating a Zero Trust Model for Microservices Architectures with SPIRE and Envoy 

This workshop will get you started with SPIRE and Envoy Secret Directory Service (SDS) to create a **Zero Trust** security model within a microservices architecture in Kubernetes. A Zero Trust security model uses cryptographic identities for authenticating every system and application services, and allows these services to establish a two-way (mutual) authentication and communicate securily in the environment. 

SPIRE is the production-ready implementation of the Secure Production Identity Framework for Everyone (SPIFFE) open-source specifications for implementing a Zero Trust security model through an Identity control plane and for establishing trust among services in dynamic and heterogeneous environments. The heart of these specifications is the one that defines short lived cryptographic identity documents called **SVIDs** (SPIFFE Verifiable Identity Document). Services can retrieve their service identities through the SPIRE **Workload API** and then use these identity documents when authenticating to other services, for example when establishing end-to-end mutual TLS (mTLS) encrypted connections and communicate securely, thus creating a Zero Trust security model.

It may be challenging for organizations to modify their applications to use the SPIRE Workload API directly. **Envoy sidecar proxy** can be used to connect to the SPIRE Workload API, retrieve the service identities, and encrypt and authenticate traffic on behalf of the applications. Envoy is a self contained process that is designed to run alongside every application service.

In this workshop, you'll go through the deployment of three containerized applications (2 frontend services and 1 backend service) in a Kubernetes cluster and the configuration of **Envoy sidecar proxies** sitting in front of these applications. The Envoy proxies will be configured to use the SPIRE Agent SDS implementation and establish secure communication through mTLS connections on behalf of the applications. You'll configure the existing SPIRE system to provide service identity to Envoy proxy services in the form of X.509 certificates (aka X.509 SVIDs) that will be consumed by Envoy proxies, so the applications can communicate securely.

A SPIRE deployment is composed of a SPIRE Server and a SPIRE Agent installed on each Kubernetes worker node on which a service is running: 

**SPIRE Server:** This component acts as a signing authority for identities issued to a set of services via agents. It is responsible for generating and signing all X.509 certificates (SVIDs) in the entire system. It also maintains a registry of service identities and the conditions that must be verified in order for those identities to be issued.

**SPIRE Agent:** This component is responsible for delivering identities to the workload services. It seats on each Kubernetes worker nodes and it is responsible for serving the *Workload API* and for providing identified services with their certificate SVIDs. By default, SPIRE Agent acts as SDS Provider for Envoy.

>**Important Note:**
_For this hands-on workshop, the SPIRE components (Server and Agents) have already been deployed in a Kubernetes cluster managed by **HPE Ezmeral Runtime Enterprise** (formerly known as HPE Ezmeral Container Platform). For more information on how to get a SPIRE Server and SPIRE Agent running in a Kubernetes cluster, check out [here](https://spiffe.io/docs/latest/try/getting-started-k8s/)._ 


# Authors: [Florian Buehr](mailto:florian.buehr@hpe.com); [Denis Choukroun](mailto:denis.choukroun@hpe.com)

## Handouts
You can freely copy the Jupyter Notebooks, including their output, in order to practice back at your office at your own pace, leveraging a local installation of Jupyter Notebook on your laptop.

- You install the Jupyter Notebook application from [here](https://jupyter.org/install). 
- A Beginners Guide is also available [here](https://jupyter-notebook-beginner-guide.readthedocs.io/en/latest/what_is_jupyter.html)


Enjoy the labs ! :-)


## Definitions
Before diving into the heart of the hands-on exercises, we believe it is important you understand some of the SPIFFE/SPIRE and Envoy concepts and terminologies:

- ***Envoy:*** Envoy is a popular open-source service proxy that is widely used to provide abstracted, secure, authenticated and encrypted communication between services. Envoy is a self contained process that is designed to run alongside every application service. All of the Envoy proxies form a communication mesh in wich each application sends and receives messages to and from **localhost** and is unaware of the network topology.

- ***Envoy Secret Discovery Service (SDS):*** One component of this configuration system is the Secret Discovery Service protocol or SDS. Envoy uses SDS to retrieve and maintain updated “secrets” from SDS providers. In the context of SPIRE, these secrets are the service identifier, the certificate document, and the trusted CA certificate(s). The SPIRE Agents act as the Envoy SDS Provider and issue these elements to Envoy proxy. The Envoy proxy uses them to provide mutual TLS authentication on behalf of applications. 

- ***SPIRE Server:*** This component acts as a signing authority for service identities (the SPIFFE ID service identifier and the certificate SVID) issued to services via SPIRE agents. It is responsible for generating and signing all certificates (known as ***SVIDs***) in the entire system, and creating the ***Trust Bundle*** (a set of trusted CA certificates). It also maintains a registry of system and service identities, and the conditions that must be verified in order for those identities (i.e.: the SPIFFE ID and the SVIDs) to be issued.

- ***SPIRE Agent:*** This component seats on each Kubernetes worker nodes and it is responsible for serving the ***Workloads API*** to services running on Kubernetes worker nodes and for providing the identified services with their service identities (the SPIFFE ID and cryptographic identity document (SVID)). The SPIRE Agent can be configured as an SDS provider for Envoy, allowing it to directly provide Envoy with the elements (SPIFFE ID service identifier, the X.509 certificate SVID, and the Trust Bundle) it needs to provide TLS authentication. The SPIRE Agent will also take care of re-generating the short-lived keys and certificates as required. 

- ***SPIFFE ID:*** A service identifier that uniquely identifies a service in SPIRE environment.

- ***SPIFFE Verifiable Identity Document (SVID):*** A SVID is the document with which a service proves its identity to a resource. A SVID contains a single SPIFFE ID which represents the identity of a service presenting it. It encodes the SPIFFE ID in a cryptographically-verifiable document in one of two supported formats: an X.509 certificate or a JWT token. The cryptographic properties of the SVID allow a particular service to prove its identity and that its certificate SVID is authentic and that it belongs to the service presenting it.

- ***Trust Bundle:*** The set of CA certificates that a service can use to verify a certificate SVID presented by another service. The Trust Bundle is generated by the SPIRE Server and distributed to the SPIRE Agents.

- ***Workload API:*** The SPIFFE Workload API, exposed by the SPIRE agent, is the method through which services obtain their service identities (SPIFFE ID and certificate SVIDs) and Trust Bundle. A service calls the Workload API to request its service identity and obtain the Trust Bundle.

## Workflow
 
To illustrate how SPIRE can be used for application identity management and allow services to establish secure communication through mTLS connections, we create a simple scenario with three services. One service will be the **backend** application that is a simple nginx instance serving static data, with Envoy as a sidecar proxy to handle X.509 certificates (SVIDs), certificate rotation and encrypted mTLS communication on behalf of the backend service. On the other side, we run two instances of the Symbank demo banking application acting as the **frontend** services, with Envoy as a sidecar proxy to also handle X.509 certificate (SVIDs), certificate rotation and encrypted mTLS communication on behalf of frontend services. The Symbank frontend services send HTTP requests to the nginx backend to get the user account details. The Envoy proxies run in the same POD as the Frontend and Backend PODs, encrypt and authenticate traffic. 


<p align="center">
  <img src="Pictures/SPIFFE-Envoy-with-X509-SVIDs-v4.png">

As shown in the diagram, to use SPIRE to establish mTLS connection between workloads requires the following:
   
* A SPIRE Server (already deployed in the Kubernetes cluster),
* A SPIRE Agent already deployed on each K8s worker node and that also acts as SDS Provider for Envoy. The agent issues identity documents, through the Workload API, to Envoy proxies on application service's behalf.
* The application services (containerized application running on the Kubernetes cluster) along with their Envoy sidecar proxy,
* Creation of registration entries on SPIRE Server to identify services and issue the service identifier (SPIFFE ID) and X.509 certificates (SVIDs) for the services,
* Configuration of Envoy sidecar proxies sitting in front of the backend and frontend services, so they request their service identities to SPIRE and establish encrypted mTLS connections on each application service's behalf. 

In this hands-on technical workshop you will:

* Deploy the application services (Backend, Frontend services) with Envoy as proxy on Kubernetes cluster managed by HPE Ezmeral Runtime Enterprise.
* Create registration entries on the SPIRE Server for the Envoy sidecar proxy instances sitting in front of the application services.
* Explore the anatomy of Envoy sidecar proxies sitting in front of the backend and frontend services.
* Test successful X.509 authentication for mTLS connection between the application services through the Envoy proxies.
* Configure an Envoy RBAC HTTP filter policy and observe how RBAC policy can be used to allow granulr access to information from the backend.

>**Note:** _In this workshop, you will all share the resources of a Kubernetes cluster running on HPE Ezmeral Runtime Enterprise with SPIRE components (Server and Agents) already deployed. To learn the fundamentals of SPIFFE and SPIRE, feel free to register to the hands-on workshop [SPIFFE - SPIRE 101 – An introduction to SPIFFE server and SPIRE agent security concepts](https://hackshack.hpedev.io/workshop/27)._
    
### Lab 1: Authenticate as tenant user to HPE Ezmeral Runtime Enterprise
In this first lab, you will connect to the HPE Ezmeral Runtime Enterprise REST API endpoint and retrieve an authentication session token to be used for fetching the KubeConfig file you will need to interact with the Kubernetes cluster available for your tenant.
    
* [Lab 1](1-WKSHP-SPIRE-Envoy-X509-Get-Kubeconfig.ipynb)

### Lab 2: Deploy application services and configure their Envoy sidecar proxy
In this second lab, you will deploy the application services and configure Envoy sidecar proxies and SPIRE to consume X.509 certificates provided by SPIRE, and test successful X.509 authentication for encrypted mTLS connection between the application services through the Envoy proxies.
    
* [Lab 2](2-WKSHP-SPIRE-Envoy-X509-Deploy-Workloads.ipynb)
    
# Thank you!
![grommet.JPG](Pictures/grommet.jpg)