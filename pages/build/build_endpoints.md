---
title: Spine Endpoints
keywords: build, directory, endpoints
tags: [design]
sidebar: overview_sidebar
permalink: build_endpoints.html
summary: "The Spine Directory Service provides details of all systems endpoints registered with the Spine"
---

{% include important.html content="All information provided below is indicative and subject to on-going review." %}

# Spine Accredited Systems and Endpoints #

Every system that connects to the Spine has one or more "Accredited System" entry in the Spine Directory, identified by an Accredited System Identifier (ASID).

This ASID is unique to a system deployed in a specific organisation, so the same application deployed into three NHS organisations would typically be represented as three unique ASIDs.

In addition, each endpoint that is registered with the Spine has one or more "interaction IDs" associated with the endpoint. These identify the types of Spine interaction that the endpoint has been assured to perform. These are used to control what functionality can be called by connecting systems - calls to interactions for which the accredited system has not been assured will be blocked.

## Terminology ##

| **MHS** | Message Handling Server. A middleware that handles messaging to/from Spine. |
| **ASID** | Accredited System Identifer. A unique number allocated to a system on accreditation for connection to Spine. |
| **MHS Endpoint** | An endpoint registered with Spine for use with multiple systems via a MHS. Each system has its own ASID. |
| **CMA Endpoint** | Combined MHS and Accredited System Endpoint. An endpoint registered with Spine for a single system. |

## Consuming Spine Services ##

A consuming system will interact with SDS in order to resolve the FHIR Endpoint Server Root URL to be used when constructing the request to be made to the Spine service it wishes to call (or another external endpoint when sending a brokered call through the [Spine Secure Proxy]()).

This is a two step process, as follows:

1. Lookup the Accredited System ID (ASID)
2. Lookup the Message Handling System (MHS)

Once the MHS record has been retrieved the fully qualified domain name (FQDN) and full endpoint of the FHIR server can be retrieved from returned attributes of the MHS record.

Systems SHOULD cache SDS query results giving details of consuming system, endpoints and endpoint capability on a per session basis.

Consuming systems SHALL NOT cache and re-use consuming system, endpoint information derived from SDS across multiple patient encounters or practitioner usage sessions. Each new patient encounter will result in new lookups to ascertain the most up-to-date consuming system, endpoint and endpoint capability.


### Step 1: Accredited System ID (ASID) Lookup ###

#### Lookup for National Services ####

When the client wants to make a call to a national Spine service (e.g. NRLS, Visitors and Migrants, etc), the client SHALL query SDS to find the appropriate Accredited System ID (ASID) as follows:

- Accredited System type
	- objectClass = `nhsAs`
- Organisational code
	- nhsIDCode = XXXXXXX
- Interaction ID
	- nhsAsSvcIA = *[interactionId]* of the API operation required.

```bash
ldapsearch -x -H ldaps://ldap.vn03.national.ncrs.nhs.uk –b "ou=services, o=nhs" 
	"(&(nhsIDCode=[odsCode]) (objectClass=nhsAS)(nhsAsSvcIA=[interactionId]))" 
	uniqueIdentifier nhsMhsPartyKey
```

The ASID will be returned in the uniqueIdentifier attribute which is returned from the ldaps query above.

#### Lookup for Brokered call to External Services ####

When the client wants to query an external service brokered through the Spine Security Proxy (e.g. a GP Connect API), the client SHALL use an organisation ODS code for the target organisation to lookup the Accredited System ID (ASID) as follows:

- Accredited System type
	- objectClass = `nhsAs`
- Organisational code
	- nhsIDCode = *[odsCode]* of the target organisation (e.g. GP practice).
- Interaction ID
	- nhsAsSvcIA = *[interactionId]* of the API operation required.

```bash
ldapsearch -x -H ldaps://ldap.vn03.national.ncrs.nhs.uk –b "ou=services, o=nhs" 
	"(&(nhsIDCode=[odsCode]) (objectClass=nhsAS)(nhsAsSvcIA=[interactionId]))" 
	uniqueIdentifier nhsMhsPartyKey
```

The ASID will be returned in the uniqueIdentifier attribute which is returned from the ldaps query above.

Note that ldaps is used to establish a TLS session rather than the StartTLS option. Also note that once the TLS session is established, SASL authentication is not used by SDS and is therefore disabled through the -x option.

Please refer to the specification of the specific FHIR API you are using for details of the interaction ID to use:

- [GP Connect operation guidance](https://developer.nhs.uk/apis/gpconnect/development_fhir_operation_guidance.html) for details of the GPConnect interactionId appropriate for your use case.


#### Step 2: Message Handling System (MHS) Lookup ####

Clients SHALL lookup the FHIR endpoint from the MHS record using the Party Key retrieved in step 1, as follows:

- Message Handling System type
	- objectClass = `nhsMHS`
- MHS Party Key
	- nhsMHSPartyKey = *[partyKey]* as retrieved from the nhsMhsPartyKey attribute in step 1
- MHS Interaction ID
	- nhsMhsSvcIA = *[interactionId]* of the GP Connect API operation required(?)


```bash
ldapsearch -x -H ldaps://ldap.vn03.national.ncrs.nhs.uk -b "ou=services, o=nhs" 
	"(&(nhsMhsPartyKey=[partyKey]) (objectClass=nhsMhs) (nhsMhsSvcIA=[interactionId]))" 
	nhsMhsEndPoint
```

The FHIR endpoint URL of the message handling system can then be extracted from the `nhsMhsEndPoint` attribute of the MHS record. The attribute nhsMhsFQDN could also be retrieved in the above query to retrieve the FQDN of the endpoint, though this can be extracted from the nhsMhsEndPoint.

#### ldapsearch configuration ####

SDS requires TLS Mutual Authentication. It is therefore necessary to configure ldapsearch in the examples above with the certificates necessary to verify the authenticity of the SDS LDAP server, and also to enable SDS to verify the spine endpoint making the LDAP request:

1. Root and SubCA spine development certificates available from Assurance Support
2. Obtain a client certificate by submitting a certificate signing request for your development endpoint to Assurance Support

##### Server certificate setup #####
For the examples above, ldapsearch should be configured to find the RootCA and SubCA certificates using the TLS_CACERT option in the ldap.conf file. This should point to a file, in PEM format, which contains both root and subca certificates ensuring that the root certificate is placed after the subCA certificae. The LDAPCONF environment vairable can be used to define the location of the ldap.conf 

##### Client certificate setup #####
The client certificate and encrypted private key should be defined in the .ldaprc file using the following directives.

`
TLS_CERT C:\mydir\cert.pem
TLS_KEY C:\mydir\key.pem
`

The location of the .ldaprc file can be defined using the LDAPRC environment variable.

Please contact [Assurance Support service desk](mailto:sa.servicedesk@nhs.net) for certificates and details of the ldap server for your environment.

### Worked example of the endpoint lookup process ###

**Given**
A consuming system which needs to get the HTML View of a patient record at the patient's registered practice. The consuming system has the following information about the patient:
- NHS Number
- A set of demographic details about the patient

**When**
The consuming system interacts with GP Connect

**Then** 
The following steps MUST be followed:


#### Step 0. PDS Trace (pre-requisite step)

The Consuming system is responsible for [performing a PDS Trace](integration_personal_demographic_service.html) to both verify the identity of the patient and retrieve the ODS code of the patient's registered primary care practice. 

For this example, NHS Number 9000000084 with demographic details Mr Anthony Tester, 19 Ficticious Avenue, Testtown returns the ODS code T99999


#### Step 1. Accredited System Lookup on SDS

The ASID and Party Key is now looked up on SDS. The example below uses ldapsearch:

	
	ldapsearch -x -H ldaps://ldap.vn03.national.ncrs.nhs.uk –b "ou=services, o=nhs" 
	"(&(nhsIDCode=T99999) (objectClass=nhsAS)(nhsAsSvcIA=urn:nhs:names:services:gpconnect:fhir:operation:gpc.getcarerecord))" 
	uniqueIdentifier nhsMhsPartyKey
	
This query should return a single matching accredited system object from SDS, the ASID being found in the uniqueIdentifier attribute. In the case, ldapsearch returns the following results:


	999999999999, Services, nhs
	dn: uniqueIdentifier=9999999999,ou=Services,o=nhs
	uniqueIdentifier: 999999999999
	nhsMhsPartyKey: T99999-9999999

	# search result
	search: 1
	result: 0 Success

	
#### Step 2: MHS lookup on SDS to determine FHIR endpoint Server Root URL

Using the Party Key retrieved from Step 1, and the same interaction ID, the following ldapsearch query is executed:

	ldapsearch -x -H ldaps://ldap.vn03.national.ncrs.nhs.uk -b "ou=services, o=nhs" 
	"(&(nhsMhsPartyKey=T99999-9999999) (objectClass=nhsMhs) (nhsMhsSvcIA=urn:nhs:names:services:gpconnect:fhir:operation:gpc.getcarerecord))" 
	nhsMhsEndPoint nhsMHSFQDN
	

This query should again return a single endpoint. In this case, the ldapquery returns the following results:

	# 472b35d4641b76454b13, Services, nhs
	dn: uniqueIdentifier=472b35d4641b76454b13,ou=Services,o=nhs
	nhsMhsEndPoint: https://pcs.thirdparty.nhs.uk/T99999/DSTU2/1
	nhsMHSFQDN: pcs.thirdparty.nhs.uk

	# search result
	search: 2
	result: 0 Success
	


#### Step 3: Consumer constructs full GP Connect request URL to be sent to the Spine Security Proxy

The format of the full URL which the consuming sytem is responsible for constructing is as follows:

`https://[URL of Spine Security Proxy]/[Provider Server Root URL]/[FHIR request]`

The value returned in the nhsMhsEndPoint attribute in Step 2 should be treated as the FHIR Server Root URL at the provider system.

In this example, to issue a GetCareRecord request, the following request would be made:

`POST https://testspineproxy.nhs.domain.uk/https://pcs.thirdparty.nhs.uk/T99999/DSTU2/1/Patient/$gpc.getcarerecord`



### Provider System Viewpoint ###

The provider system is responsible for populating SDS with the necessary information to enable reliable endpoint lookup by consuming systems.

In order to ensure that endpoint lookup is reliable, the following guidelines must be followed for First of Type implementations:

**1. Format of Server Root URL**

The *Server Root URL* for a given ASID SHALL be defined in the nhsMhsEndPoint attribute of the MHS record (i.e. the ldap object of type nhsMhs). This URL SHALL be in the format described in the [API Versioning](development_general_api_guidance.html#fhir-api-versioning) guidance.




**2. For First of Type interactions, CMA type endpoints only will be used**

A CMA type endpoint refers to an endpoint which is a combined MHS sytem and Accredited System endpoint. There will be a 1-1 mapping between an Accredited System (uniquely identified by an ASID) and a Message Handling System (MSH). A single MHS record SHALL be associated with a given ASID and interaction ID.

**3. nhsMhsEndPoint attribute SHALL contain the FHIR Server Root URL only**

The nhsMHsEndPoint attribute SHALL contain the *FHIR Server Root URL*. It is the responsibility of the consuming system to construct the FHIR operation or RESTful resource request which will be postfixed to this base URL.

I.e. an example of a FHIR server root URL for a GetCareRecord interaction at practice GP0001 would be

`https://provider.thirdparty.nhs.uk/GP0001/DSTU2/1`

Note that the "Patient/$gpc.getcarerecord" is not added.

In line with this, provider systems SHOULD perform checks that the FHIR request received is a reasonable means to request the resource in view given the specificed interaction. 


**4. Practice routing identifier to be included in FHIR Server Root URL**

As described in the [API Versioning](development_general_api_guidance.html#fhir-api-versioning) guidance, a routing identifier SHALL be placed in the FHIR Servder Root URL. This routing identifier may be the ODS code of the practice, or another logical identifier which acheives reliable routing of the request to the patient's registered practice data store. It is expected that the FHIR server business logic will extract the routing identifier.

In line with this, HTTP headers SHALL NOT be used to provide this organisation routing.

**5. Practice specific ODS codes to be used**

ODS codes which refer to Principle Clinical Systems as a single entity SHALL NOT be used to provide routing. Practice specific ODS codes SHALL be used for routing purposes in the FHIR Server Root URL found in the NhsMhsEndPoint attribute of the MHS record.


**6. The FHIR Server Root URL SHALL contain the FHIR version name**

The FHIR Server Root URL defined in the nhsMhsEndPoint attribute SHALL contain the FHIR version name as described in the [API Versioning](development_general_api_guidance.html#fhir-api-versioning) guidance. This will enable versioning of provider API by FHIR version. 

In line with this, provider systems SHALL NOT version through the use of HTTP headers.


**7. FHIR version SHALL match version found in FHIR conformance statement**

The FHIR version as returned in a [CapabilityStatement](foundations_use_case_get_the_fhir_capability_statement.html) from the FHIR server which services the FHIR request SHALL match the FHIR version given in the FHIR Server Root URL.

**8. FHIR Server Root URLs associated with a given product set SHALL use same FHIR Version**

Where a provider moves in future to later version of FHIR, it will be necessary to define a new product set to accommodate the set of interactions provided by this. FHIR Server Root URLs defined for a specific product set SHALL all reference the same FHIR version. This ensures that FHIR resources references returned in FHIR responses are locally resolvable. 

For example, all interactions associated with the Appointment Management capability in a given product set must refer to the same FHIR server, so that the resource references for "Read Appointment" and "Amend" appointment would be locally resolvable to the same resource on the same FHIR Server. 


**9. Acceptable use of ASID information in HTTP Headers**

Source and destination ASID information is passed to the provider system from the Spine Security Proxy. Providers SHALL use this information for audit and debugging purposes only, and SHALL NOT use these headers to perform routing or lookups. 

It is the responsibility of the Spine Security Proxy SSP to perform lookups to determine consumer accreditation status. Routing shall be carried out as described above through practice specific ODS codes present in the FHIR Server Root URL. 




