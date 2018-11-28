---
title: Environments
keywords: test, environments
tags: [test]
sidebar: overview_sidebar
permalink: test_environments.html
summary: "The Environments page covers the various Spine environments available for test and production use"
---

# Reference Implementations #

Some APIs on Spine (including some clinical system APIs brokered through SSP) are also available for initial testing and experimentation through publicly available reference implementations. These are not hosted in the same environment as the Spine, so are not guaranteed to be 100% identical to the APIs exposed on Spine, but can provide a useful place to do initial experimentation and testing during development. The URLs for these reference implementations, where they exist, should be available from the relevant system-specific API specification.

# What Spine environments are available? #

In addition to the live environment, Spine also provides a number of "path to live" environments:

- Development
- Integration
- Deployment
- Training
- Non-Functional Testing
- Opentest

The full details of these environments, including the endpoint URLs for all the services in each environment can be found in the "authority service names" documents for each environment. These documents can be downloaded from the [Assurance Support Website](http://www.assurancesupport.digital.nhs.uk/){:target="_blank"} (note: an N3 connection required to access this site).

The above site also holds copies of the parent Root and SubCA certificates which you'll need to verify the SSL certificates registered against for Spine services (all endpoint certificates are children of these parent certificates).

Please contact [Platforms support desk](mailto:platforms.supportdesk@nhs.net) if you can't find the information required, or need help with obtaining an endpoint certificate to access a Spine environment.

# What is Opentest? #

Opentest is NHS Digital's open-access network for developing and testing healthcare applications. The Opentest network consists of a hosted component, and a VPN through which users connect. The network is not N3/HSCN connected and so is accessible to users without access to those networks.

More details are avaliable [here](https://digital.nhs.uk/spine/opentest)

