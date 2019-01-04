[![FIWARE Banner](https://fiware.github.io/tutorials.Administrating-XACML/img/fiware.png)](https://www.fiware.org/developers)

[![FIWARE Security](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/security.svg)](https://www.fiware.org/developers/catalogue/)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Administrating-XACML.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://nexus.lab.fiware.org/repository/raw/public/badges/stackoverflow/fiware.svg)](https://stackoverflow.com/questions/tagged/fiware)
<br/>
[![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)

This tutorial describes the administration of level 3 advanced authorization rules into **Authzforce**, either directly, or with the help of the **Keyrock**
GUI. The simple verb-resource based permissions are amended to use XACML and new XACML permissions added to the existing roles. The updated ruleset is automatically uploaded to **Authzforce** PDP, so that policy execution points such as the **PEP proxy** are able to apply the latest ruleset.

The tutorial demonstrates examples of interactions using the **Keyrock** GUI, as
well [cUrl](https://ec.haxx.se/) commands used to access the REST
APIs of **Keyrock**  and **Authzforce** -
[Postman documentation](https://fiware.github.io/tutorials.Identity-Management/)
is also available.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://www.getpostman.com/collections/23b7045a5b52a54a2666)

# Contents

- [Administrating XACML Rules](#administrating-xacml-rules)
  * [What is XACML](#what-is-xacml)
  * [PAP - Policy Administration Point](#pap---policy-administration-point)
    + [Authzforce PAP](#authzforce-pap)
    + [Keyrock PAP](#keyrock-pap)
  * [PEP - Policy Execution Point](#pep---policy-execution-point)
- [Prerequisites](#prerequisites)
  * [Docker](#docker)
  * [Cygwin](#cygwin)
- [Architecture](#architecture)
- [Start Up](#start-up)
    + [Dramatis Personae](#dramatis-personae)
- [XACML Administration](#xacml-administration)
  * [Authzforce - Administrating XACML PolicySets](#authzforce---administrating-xacml-policysets)
    + [Creating a new Domain](#creating-a-new-domain)
    + [Creating an initial PolicySet](#creating-an-initial-policyset)
    + [Activating the initial PolicySet](#activating-the-initial-policyset)
    + [Updating a PolicySet](#updating-a-policyset)
    + [Activating an updated PolicySet](#activating-an-updated-policyset)
  * [Keyrock - Administrating XACML Permissions](#keyrock---administrating-xacml-permissions)
    + [Create Token with Password](#create-token-with-password)
    + [Update an XACML Permission](#update-an-xacml-permission)
- [PEP Proxy - Extending Advanced Authorization](#pep-proxy---extending-advanced-authorization)
  * [PEP Proxy - Sample Code](#pep-proxy---sample-code)
  * [PEP Proxy - Running the Example](#pep-proxy---running-the-example)

# Administrating XACML Rules

> **12.3 Central Terminal Area**
>
> * Red or Yellow Zone
>    * No private vehicle shall stop, wait, or park in the red or yellow zone.
> * White Zone
>    * No vehicle shall stop, wait, or park in the white zone unless actively
> engaged in the immediate loading or unloading of passengers
> and/or baggage.
>
> — Los Angeles International Airport Rules and Regulations, Section 12 - Landside Motor Vehicle Operations

Business rules change over time, and it is necessary to be able to amend access controls accordingly. The [previous tutorial](https://github.com/Fiware/tutorials.XACML-Access-Rules) included a previously generated XACML `<PolicySet>` loaded into **Authzforce**. **Authzforce**. offers advanced authorization (level 3) access control - this means that every policy decision is calculated on the fly so new rules can be applied under new circumstances.
The [Authzforce](https://authzforce-ce-fiware.readthedocs.io/) Policy Decision Point (PDP) was discussed in the [previous tutorial](https://github.com/Fiware/tutorials.XACML-Access-Rules) - it interprets rules according to the
[XACML standard](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=xacml), this offers a means to adjudicate on any access request provided that sufficient information can be supplied.

For full flexibility, it must be possible to load, update and activate a new access control `<PolicySet>` whenever necessary. In order to do, this **Authzforce** offers a simple REST Policy Adminstration Point (PAP), an alternative role-based PAP is available within **Keyrock**

## What is XACML

eXtensible Access Control Markup Language (XACML) is a vendor neutral
declarative access control policy language. It was created to promote common
access control terminology and interoperability. The architectural naming
conventions for elements such as Policy Execution Point (PEP) and Policy
Decision Point (PDP) come from the XACML specifications.

XACML policies are split into a hierarchy of three levels - `<PolicySet>`,
`<Policy>` and `<Rule>`, the `<PolicySet>` is a collection of `<Policy>`
elements each of which contain one or more `<Rule>` elements.

Each `<Rule>` within a `<Policy>` is evaluated as to whether it should grant
access to a resource - the overall `<Policy>` result is defined by the overall
result of all `<Rule>` elements processed in turn. Separate `<Policy>` results
are then evaluated against each other using combining alogorthms define which
`<Policy>` wins in case of conflict.

Further information can be found within the [XACML standard](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=xacml)

## PAP - Policy Administration Point

For the first half of the tutorial, a simple two rule `<PolicySet>` will be administered using the **Authzforce** PAP API. Thereafter **Keyrock** will be used with the existing tutorial application to administer XACML rules on an individual `<Rule>` level. Code within the **PEP-Proxy** will be customized to enable the enforcement of complex XACML rules.

### Authzforce PAP

Within the **Authzforce** PAP all CRUD actions occur on the `<PolicySet>` level. It is therefore necessary to create a complete valid XACML file before uploading it to the service. There is no GUI available to ensure the validity of the  `<PolicySet>` prior to uploading the XACML.

### Keyrock PAP

**Keyrock** can create a valid XACML file based on available roles and permissions and pass this to **Authzforce**. Indeed **Keyrock** already does this when combined with **Authzforce** to translate its own basic authorization (level 2) permissions into advanced authorization (level 3) permissions which can be adjudicated by **Authzforce**.

Each role corresponds to a `<Policy>`, each permission within a role corresponds to a `<Rule>`. There is a GUI available for uploading and amending the XACML for each `<Rule>` and all CRUD actions occur on the `<Rule>` level.

Provided care is taken when creating `<Rule>` you can use **Keyrock** to simplify the administration of XACML and create a valid `<PolicySet>`.

## PEP - Policy Execution Point

When using advanced authorization (level 3),  a policy execution point sends the an authorization request to he relevant domain endpoint within **Authzforce**,
providing all of the information necessary for **Authzforce** to provide a
judgement. Details of the interaction can be found in the [previous tutorial](https://github.com/Fiware/tutorials.XACML-Access-Rules).

The full code to supply each request to **Authzforce** can be found within the
tutorials'
[Git Repository](https://github.com/Fiware/tutorials.Step-by-Step/blob/master/context-provider/lib/azf.js)

Obviously the definition of _"all of the information necessary"_ may change
over time, applications must therefore be flexible enough to be able to modify the requests sent to ensure that sufficient information is passed.

# Prerequisites

## Docker

To keep things simple all components will be run using
[Docker](https://www.docker.com). **Docker** is a container technology which
allows to different components isolated into their respective environments.

-   To install Docker on Windows follow the instructions
    [here](https://docs.docker.com/docker-for-windows/)
-   To install Docker on Mac follow the instructions
    [here](https://docs.docker.com/docker-for-mac/)
-   To install Docker on Linux follow the instructions
    [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker
applications. A
[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Identity-Management/master/docker-compose.yml)
is used configure the required services for the application. This means all
container services can be brought up in a single command. Docker Compose is
installed by default as part of Docker for Windows and Docker for Mac, however
Linux users will need to follow the instructions found
[here](https://docs.docker.com/compose/install/)

## Cygwin

We will start up our services using a simple bash script. Windows users should
download [cygwin](http://www.cygwin.com/) to provide a command-line
functionality similar to a Linux distribution on Windows.

# Architecture

This application demonstrates the admininistration of level 3 Advanced Authorization security into the existing
Stock Management and Sensors-based application created in
[previous tutorials](https://github.com/Fiware/tutorials.XACML-Access-Rules/) and
secures access to the context broker behind a
[PEP Proxy](https://github.com/Fiware/tutorials.PEP-Proxy/). It will make use of
five FIWARE components - the
[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/),the
[IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/),
the [Keyrock](https://fiware-idm.readthedocs.io/en/latest/) Identity Manager,
the [Wilma]() PEP Proxy and the
[Authzforce](https://authzforce-ce-fiware.readthedocs.io) XACML Server. All
access control decisions will be delegated to **Authzforce** which will read the
ruleset from a previously uploaded policy domain.

Both the Orion Context Broker and the IoT Agent rely on open source
[MongoDB](https://www.mongodb.com/) technology to keep persistence of the
information they hold. We will also be using the dummy IoT devices created in
the [previous tutorial](https://github.com/Fiware/tutorials.IoT-Sensors/).
**Keyrock** uses its own [MySQL](https://www.mysql.com/) database.

Therefore the overall architecture will consist of the following elements:

-   The FIWARE
    [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) which
    will receive requests using
    [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
-   The FIWARE
    [IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/)
    which will receive southbound requests using
    [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) and convert
    them to
    [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    commands for the devices
-   FIWARE [Keyrock](https://fiware-idm.readthedocs.io/en/latest/) offer a
    complement Identity Management System including:
    -   An OAuth2 authentication system for Applications and Users
    -   A site graphical frontend for Identity Management Administration
    -   An equivalent REST API for Identity Management via HTTP requests
-   FIWARE [Authzforce](https://authzforce-ce-fiware.readthedocs.io/) is a XACML
    Server providing an interpretive Policy Decision Point (PDP) protecting access to resources
    such as **Orion** and the tutorial application.
-   FIWARE [Wilma](https://fiware-pep-proxy.rtfd.io/) is a PEP Proxy securing
    access to the **Orion** microservices, it delegates the passing of
    authorisation decisions to **Authzforce** PDP
-   The underlying [MongoDB](https://www.mongodb.com/) database :
    -   Used by the **Orion Context Broker** to hold context data information
        such as data entities, subscriptions and registrations
    -   Used by the **IoT Agent** to hold device information such as device URLs
        and Keys
-   A [MySQL](https://www.mysql.com/) database :
    -   Used to persist user identities, applications, roles and permissions
-   The **Stock Management Frontend** does the following:
    -   Displays store information
    -   Shows which products can be bought at each store
    -   Allows users to "buy" products and reduce the stock count.
    -   Allows authorized users into restricted areas, it also delegates
        authoriation decisions to the **Authzforce** PDP
-   A webserver acting as set of
    [dummy IoT devices](https://github.com/Fiware/tutorials.IoT-Sensors) using
    the
    [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    protocol running over HTTP - access to certain resources is restricted.

Since all interactions between the elements are initiated by HTTP requests, the
entities can be containerized and run from exposed ports.

![](https://fiware.github.io/tutorials.Administrating-XACML/img/architecture.png)

The all container configuration values found in the YAML file
have been described in previous tutorials.

# Start Up

To start the installation, do the following:

```console
git clone git@github.com:Fiware/tutorials.Administrating-XACML.git
cd tutorials.Administrating-XACML

./services create
```

> **Note** The initial creation of Docker images can take up to three minutes

Thereafter, all services can be initialized from the command-line by running the
[services](https://github.com/Fiware/tutorials.Administrating-XACML/blob/master/services)
Bash script provided within the repository:

```console
./services start
```


> :information_source: **Note:** If you want to clean up and start over again
> you can do so with the following command:
>
> ```console
> ./services stop
> ```

### Dramatis Personae

The following people at `test.com` legitimately have accounts within the
Application

-   Alice, she will be the Administrator of the **Keyrock** Application
-   Bob, the Regional Manager of the supermarket chain - he has several store
    managers under him:
    -   Manager1
    -   Manager2
-   Charlie, the Head of Security of the supermarket chain - he has several
    store detectives under him:
    -   Detective1
    -   Detective2

The following people at `example.com` have signed up for accounts, but have no
reason to be granted access

-   Eve - Eve the Eavesdropper
-   Mallory - Mallory the malicious attacker
-   Rob - Rob the Robber


<details>
  <summary>
   For more details <b>(Click to expand)</b>
  </summary>

| Name       | eMail                     | Password |
| ---------- | ------------------------- | -------- |
| alice      | alice-the-admin@test.com  | `test`   |
| bob        | bob-the-manager@test.com  | `test`   |
| charlie    | charlie-security@test.com | `test`   |
| manager1   | manager1@test.com         | `test`   |
| manager2   | manager2@test.com         | `test`   |
| detective1 | detective1@test.com       | `test`   |
| detective2 | detective2@test.com       | `test`   |


| Name    | eMail               | Password |
| ------- | ------------------- | -------- |
| eve     | eve@example.com     | `test`   |
| mallory | mallory@example.com | `test`   |
| rob     | rob@example.com     | `test`   |

</details>

# XACML Administration

To apply an access control policy, it is necessary to be able to do the following:

1. Create a consistent `<PolicySet>`
2. Supply a Policy Execution Point (PEP) which provides necessary data

As will be seen, **Keyrock** is able help with the first point, and custom code within the **PEP Proxy** can help with the second. **Authzforce** itself does not offer a UI, and is not concerned with generation and management of XACML policies - it assumes that each `<PolicySet>` it receives has already been generated by another component.

Full-blown XACML editors are available, but the limited editor within **Keyrock** is usually sufficient for most access control scenarios.

## Authzforce PAP

**Authzforce** can act as a Policy Administration Point (PAP), this means that PolicySets can be created and amended using API calls directly to **Authzforce**

However there is no GUI for creating or amending a `<PolicySet>`, and no generation tool. All CRUD actions occur on the `<PolicySet>` level.

A full XACML `<PolicySet>` for any non-trivial application is very verbose. For simplicity, for this part of the tutorial, we will work on a completely new set of access rules designed for airport parking enforcement. **Authzforce** is implicitly multi-tenant - a single XACML server can be used to administrate access control policies for multiple applications. Security policies for each application are held in a separate **domain** where they can access their own `<PolicySets>` - therefore administrating the airport application will not interfere with the existing rules for the tutorial supermarket application.

### Creating a new Domain

To create a new domain in **Authzforce**, make a POST request to the
`/authzforce-ce/domains` endpoint including a unique `external-id` within
the `<domainProperties>` element

#### :one: Request

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<domainProperties xmlns="http://authzforce.github.io/rest-api-model/xmlns/authz/5" externalId="airplane"/>'
```

#### Response

The response includes a `href` in the `<n2:link>` element  which holds the
`domain-id` used internally within **Authzforce**.

An empty `PolicySet` will be created for the new domain. By default all
access will be permitted.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns4:link xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0" rel="item" href="Sv-RRw9vEem6UQJCrBIBDA" title="Sv-RRw9vEem6UQJCrBIBDA"/>
```

The new `domain-id`  (in this case `Sv-RRw9vEem6UQJCrBIBDA` ) will be used with all subsequent requests.

#### :two: Request

To request a decision from Authzforce, make a POST request to the
`domains/{domain-id}/pdp` endpoint. In this case the user
is requesting access to `loading` in the `white` zone.

Remember to amend the request below to use your own `{domain-id}`:

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
      </Attribute>
      <Attribute AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">white</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" />
</Request>'
```

#### Response

The response for the request includes a `<Decision>` element to `Permit` or `Deny` access to the resource. at this point all requests will be `Permit`

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Response xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0">
    <Result>
        <Decision>Permit</Decision>
    </Result>
</Response>
```

### Creating an initial PolicySet

To create a `PolicySet` for a given domain information in **Authzforce**, make a
POST request to the
`/authzforce-ce/domains/{{domain-id}}/pap/policies` endpoint including the full set
of XACML rules to upload.

For this initial Policy, the following rules will be enforced

* The **white** zone is for immediate loading and unloading of passengers only
* There is no stopping in the **red** zone


#### :three: Request

The full data for an XACML `<PolicySet>` is very verbose and has been omitted from the request below:

<details>
  <summary>

Remember to amend the request below to use your own `{domain-id}`:

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pap/policies \
  -H 'Content-Type: application/xml' \
  -d '<PolicySet>...etc</PolicySet>'
```

**(Click to expand)**
  </summary>

Remember to amend the request below to use your own `{domain-id}`:

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pap/policies \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<PolicySet xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" PolicySetId="f8194af5-8a07-486a-9581-c1f05d05483c" Version="1" PolicyCombiningAlgId="urn:oasis:names:tc:xacml:3.0:policy-combining-algorithm:deny-unless-permit">
   <Description>Policy Set for Airplane!</Description>
   <Target />
   <Policy PolicyId="airplane" Version="1.0" RuleCombiningAlgId="urn:oasis:names:tc:xacml:3.0:rule-combining-algorithm:deny-unless-permit">
      <Description>Vehicle Roles from the Male announcer in the movie Airplane!</Description>
      <Target>
         <AnyOf>
            <AllOf>
               <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                  <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
                  <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
               </Match>
            </AllOf>
         </AnyOf>
      </Target>
      <Rule RuleId="white-zone" Effect="Permit">
         <Description>The white zone is for immediate loading and unloading of passengers only</Description>
         <Target>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">white</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action" AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
         </Target>
      </Rule>
      <Rule RuleId="red-zone" Effect="Deny">
         <Description>There is no stopping in the red zone</Description>
         <Target>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">red</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">stopping</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action" AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
         </Target>
      </Rule>
   </Policy>
</PolicySet>
'
```

</details>


#### Response

The response contains the internal id of the policy held within **Authzforce** and
version information about the `PolicySet` versions available.
The rules of the new `PolicySet` will not be applied until the `PolicySet` is activated.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns4:link xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0" rel="item" href="f8194af5-8a07-486a-9581-c1f05d05483c/1" title="Policy 'f8194af5-8a07-486a-9581-c1f05d05483c' v1"/>
```

### Activating the initial PolicySet

To activate a `PolicySet`, make a PUT request to the
`/authzforce-ce/domains/{domain-id}/pap/pdp.properties` endpoint including the `policy-id`
to update within the `<rootPolicyRefExpresion>` attribute

#### :four: Request

Remember to amend the request below to use your own `{domain-id}`:

```console
curl -X PUT \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pap/pdp.properties \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
  <pdpPropertiesUpdate xmlns="http://authzforce.github.io/rest-api-model/xmlns/authz/5">
    <rootPolicyRefExpression>f8194af5-8a07-486a-9581-c1f05d05483c</rootPolicyRefExpression>
  </pdpPropertiesUpdate>'
```

#### Response

The response returns information about the `PolicySet` applied.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns3:pdpProperties xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0" lastModifiedTime="2019-01-03T15:54:45.341Z">
    <ns3:feature type="urn:ow2:authzforce:feature-type:pdp:core" enabled="false">urn:ow2:authzforce:feature:pdp:core:strict-attribute-issuer-match</ns3:feature>
    <ns3:feature type="urn:ow2:authzforce:feature-type:pdp:core" enabled="false">urn:ow2:authzforce:feature:pdp:core:xpath-eval</ns3:feature>
... ETC
    <ns3:rootPolicyRefExpression>f8194af5-8a07-486a-9581-c1f05d05483c</ns3:rootPolicyRefExpression>
    <ns3:applicablePolicies>
        <ns3:rootPolicyRef Version="1">f8194af5-8a07-486a-9581-c1f05d05483c</ns3:rootPolicyRef>
    </ns3:applicablePolicies>
</ns3:pdpProperties>
```

#### :five: Request

At this point, making a request to access to `loading` in the `white` zone will return `Permit`

Remember to amend the request below to use your own `{domain-id}`:

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
      </Attribute>
      <Attribute AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">white</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" />
</Request>'
```

#### Response

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Response xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0">
    <Result>
        <Decision>Permit</Decision>
    </Result>
</Response>
```

#### :six: Request

At this point, making a request to access to `loading` in the `red` zone will return `Deny`

Remember to amend the request below to use your own `{domain-id}`:

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
      </Attribute>
      <Attribute AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">red</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" />
</Request>'
```

#### Response

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Response xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0">
    <Result>
        <Decision>Deny</Decision>
    </Result>
</Response>
```

### Updating a PolicySet

To update a `PolicySet` for a given domain information in **Authzforce**, make a
POST request to the
`/authzforce-ce/domains/{{domain-id}}/pap/policies` endpoint including the full set
of XACML rules to upload. Note that the `Version` must be unique.

For the updated Policy, the previous rules will be reversed

* The **red** zone is for immediate loading and unloading of passengers only
* There is no stopping in the **white** zone


#### :seven: Request

The full data for an XACML `<PolicySet>` is very verbose and has been omitted from the request below:

<details>
  <summary>

Remember to amend the request below to use your own `{domain-id}`:

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pap/policies \
  -H 'Content-Type: application/xml' \
  -d '<PolicySet>...etc</PolicySet>'
```

**(Click to expand)**
  </summary>

Remember to amend the request below to use your own `{domain-id}`:

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pap/policies \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<PolicySet xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" PolicySetId="f8194af5-8a07-486a-9581-c1f05d05483c" Version="2" PolicyCombiningAlgId="urn:oasis:names:tc:xacml:3.0:policy-combining-algorithm:deny-unless-permit">
   <Description>Policy Set for Airplane!</Description>
   <Target />
   <Policy PolicyId="airplane" Version="1.0" RuleCombiningAlgId="urn:oasis:names:tc:xacml:3.0:rule-combining-algorithm:deny-unless-permit">
      <Description>Vehicle Roles from the Female announcer in the movie Airplane!</Description>
      <Target>
         <AnyOf>
            <AllOf>
               <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                  <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
                  <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
               </Match>
            </AllOf>
         </AnyOf>
      </Target>
      <Rule RuleId="red-zone" Effect="Permit">
         <Description>The red zone is for immediate loading and unloading of passengers only</Description>
         <Target>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">red</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action" AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
         </Target>
      </Rule>
      <Rule RuleId="white-zone" Effect="Deny">
         <Description>There is no stopping in the white zone</Description>
         <Target>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">white</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">stopping</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action" AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
         </Target>
      </Rule>
   </Policy>
</PolicySet>
'
```

</details>

#### Response

The response contains version information about the `PolicySet` versions available.
The rules of the new `PolicySet` will not be applied until the `PolicySet` is activated.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns4:link xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0" rel="item" href="f8194af5-8a07-486a-9581-c1f05d05483c/2" title="Policy 'f8194af5-8a07-486a-9581-c1f05d05483c' v2"/>
```

### Activating an updated PolicySet

To update an active a `PolicySet`, make another PUT request to the
`/authzforce-ce/domains/{domain-id}/pap/pdp.properties` endpoint including the `policy-id`
to update within the `<rootPolicyRefExpresion>` attribute. The ruleset will be updated to
apply the latest uploaded version.

#### :eight: Request

Remember to amend the request below to use your own `{domain-id}`:

```console
curl -X PUT \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pap/pdp.properties \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
  <pdpPropertiesUpdate xmlns="http://authzforce.github.io/rest-api-model/xmlns/authz/5">
    <rootPolicyRefExpression>f8194af5-8a07-486a-9581-c1f05d05483c</rootPolicyRefExpression>
  </pdpPropertiesUpdate>'
```

#### Response

The response returns information about the `PolicySet` applied.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns3:pdpProperties xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0" lastModifiedTime="2019-01-03T15:58:29.351Z">
    <ns3:feature type="urn:ow2:authzforce:feature-type:pdp:core" enabled="false">urn:ow2:authzforce:feature:pdp:core:strict-attribute-issuer-match</ns3:feature>
    <ns3:feature type="urn:ow2:authzforce:feature-type:pdp:core" enabled="false">urn:ow2:authzforce:feature:pdp:core:xpath-eval</ns3:feature>
... ETC
    <ns3:rootPolicyRefExpression>f8194af5-8a07-486a-9581-c1f05d05483c</ns3:rootPolicyRefExpression>
    <ns3:applicablePolicies>
        <ns3:rootPolicyRef Version="2">f8194af5-8a07-486a-9581-c1f05d05483c</ns3:rootPolicyRef>
    </ns3:applicablePolicies>
</ns3:pdpProperties>
```


#### :nine: Request

Since the new policy has been activated, at this point, making a request to access to `loading` in the `white` zone will return `Deny`

Remember to amend the request below to use your own `{domain-id}`:


```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
      </Attribute>
      <Attribute AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">white</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" />
</Request>'
```

#### Response

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Response xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0">
    <Result>
        <Decision>Deny</Decision>
    </Result>
</Response>
```

#### :one::zero: Request

Making a request to access to `loading` in the `red` zone under the current policy will return `Permit`

Remember to amend the request below to use your own `{domain-id}`:


```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
      </Attribute>
      <Attribute AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">red</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" />
</Request>'
```

#### Response

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Response xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0">
    <Result>
        <Decision>Permit</Decision>
    </Result>
</Response>
```



## Keyrock PAP

**Keyrock** offers a role based access control identity management system. Therefore every permission is only accessible to  users within a given role. We have already seen how Verb-Resource rules can be [set-up](https://github.com/Fiware/tutorials.Roles-Permissions/)) and [enforced](https://github.com/Fiware/tutorials.Securing-Access/)) using the basic authorization (level 2) access control mechanism found within Keyrock, the data for defining an advanced permission can also be administed using the **Keyrock** GUI or via **Keyrock** REST API requests.

**Keyrock** permissions work on individual XACML `<Rule>` elements rather than a complete `<PolicySet>`. The `<PolicySet>` is generated by combining all the roles and permissions.

### Create Token with Password

#### :one::one: Request

Enter a username and password to enter the application. The default super-user has the values `alice-the-admin@test.com` and `test`.

```console
curl -iX POST \
  http://localhost:3005/v1/auth/tokens \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "alice-the-admin@test.com",
  "password": "test"
}'
```

#### Response

The response header from **Keyrock** returns an `X-Subject-token` which identifies who has logged
on the application. This token is required in all subsequent requests to gain
access

```
HTTP/1.1 201 Created
X-Subject-Token: d848eb12-889f-433b-9811-6a4fbf0b86ca
Content-Type: application/json; charset=utf-8
Content-Length: 138
ETag: W/"8a-TVwlWNKBsa7cskJw55uE/wZl6L8"
Date: Mon, 30 Jul 2018 12:07:54 GMT
Connection: keep-alive
```

The response body from **Keyrock** indicates that **Authzforce** is in use and XACML Rules can be used to define access policies.

```json
{
    "token": {
        "methods": [
            "password"
        ],
        "expires_at": "2019-01-03T17:04:43.358Z"
    },
    "idm_authorization_config": {
        "level": "advanced",
        "authzforce": true
    }
}
```





#### :one: Request

```console
```

#### Response

```xml
```

### Update an XACML Permission

#### :one: Request

```console
```

#### Response

```xml
```


# PEP Proxy - Extending Advanced Authorization

## PEP Proxy - Sample Code

## PEP Proxy - Running the Example

# Next Steps

Want to learn how to add more complexity to your application by adding advanced
features? You can find out by reading the other
[tutorials in this series](https://fiware-tutorials.rtfd.io)

---

## License

[MIT](LICENSE) © FIWARE Foundation e.V.