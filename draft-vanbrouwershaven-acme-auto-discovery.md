---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "Auto-discovery mechanism for ACME client configuration"
abbrev: "ACME Auto-Discovery"
category: info

docname: draft-vanbrouwershaven-acme-auto-discovery-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: SEC
workgroup: ACME
keyword:
 - ACME
 - Auto-discovery
 - CAA
venue:
  group: ACME
  type: Working Group
  mail: acme@ietf.org
  arch: https://datatracker.ietf.org/wg/acme/about/
  github: ounsworth/acme-auto-discovery
  latest: TODO

author:
  -
      ins: P vanBrouwershaven
      name: Paul van Brouwershaven
      org: Entrust Limited
      abbrev: Entrust
      street: 2500 Solandt Road – Suite 100
      city: Ottawa, Ontario
      country: Canada
      code: K2K 3G5
      email: Paul.vanBrouwershaven@entrust.com
  -
      ins: M. Ounsworth
      name: Mike Ounsworth
      org: Entrust Limited
      abbrev: Entrust
      street: 2500 Solandt Road – Suite 100
      city: Ottawa, Ontario
      country: Canada
      code: K2K 3G5
      email: mike.ounsworth@entrust.com

normative:
  RFC8555:
  RFC8659:

informative:


--- abstract

This document presents a mechanism for automatically discovering the ACME (Automated Certificate Management Environment) [RFC8555] client configuration for the relevant domain name(s) in a shared environment. The mechanism leverages the DNS Certification Authority Authorization (CAA) Resource Record [RFC8659] to determine the authorized Certification Authorities (CAs) capable of issuing certificates for the given domain name(s). By retrieving instructions for the ACME client from the authorized CA(s), this mechanism allows for improved user preferences, increased diversity in certificate issuers, and an option to specify backup CA(s).


--- middle

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Introduction {#sec-intro}

The ACME protocol [RFC8555] offers a powerful framework for automating the issuance and validation of certificates, eliminating the need for user intervention. This capability has significantly streamlined the process of obtaining certificates for servers and infrastructure software. However, in shared environments, where multiple entities coexist, users often face limitations in modifying ACME client configurations. Consequently, they are forced to rely on manual certificate management systems or default Certificate Authorities (CAs) preconfigured for the shared environment.

This document introduces a mechanism to address the aforementioned challenge by enabling the automatic discovery of ACME client configurations for relevant domain names within a shared environment. The solution leverages the DNS Certification Authority Authorization (CAA) Resource Record [RFC 8659] to identify the authorized Certification Authorities capable of issuing certificates for a specific domain name or set of domain names.

By leveraging the power of CAA records, this mechanism empowers users with enhanced control and flexibility. Users can specify their preferences, choose from a broader range of certificate issuers, and even designate a backup Certification Authority.

This approach facilitates a more diverse and adaptable certificate management process within shared and managed environments.
This document provides a detailed description of the proposed mechanism, along with its benefits and considerations.

Additionally, it outlines the security aspects associated with the use of CAA records for ACME client configuration discovery. Finally, this document presents IANA considerations and references relevant normative and informative documents.


It is important to note that this document is informational in nature and serves to provide guidance and recommendations to implementers and operators within the Internet community.


# CAA Record

## Extensions to the CAA Record

This document defines the "discovery" and "priority" CAA parameters in the context of ACME auto discovery.

### The "discovery" Parameter

The "discovery" parameter is used to control the auto-discovery functionality of the record in the context of this document.
The value of this parameter, if specified, MUST be a Boolean, where "true" indicates that this record can be used for auto discovery, and "false" indicates that this record should not be used for auto discovery.

When this parameter is not specified the client MUST assume that discovery is enabled.

### The "priority" Parameter

The value of this parameter, if specified, MUST contain digits, where the value "1" represents the highest priority, and subsequent values like "2", "3", and so on, indicate progressively lower priorities.

The default priority, in case this parameter is not specified will be "0".

## Examples

This section shows some examples of how CAA records can be configured in the context of ACME auto discovery. CAA records are used to authorize Certification Authorities (CAs) to issue certificates for a specific domain name.

A simple CAA record allows the issuance of certificates from a single designated CA. In the following example, the CAA record for the domain "example.com" authorizes the CA "ca.example" to issue certificates. However, it does not specify any backup CA. Consequently, if the authorized CA is unable to issue the requested certificate, the certificate issuance will fail.

```
example.com CAA 0 issue "ca.example"
```

By default, when multiple CAA records are present, the CAs are randomized to distribute the load. However, some users may have preferences regarding the order in which CAs are attempted for certificate issuance. To explicitly specify the order, the "priority" parameter can be used.

In the next example, the domain "example.com" has two CAA records. The CAA record with "ca2.example" has a higher priority value of 2, indicating it should be attempted first. The CAA record with "ca1.example" has a lower priority value of 1, indicating it should be attempted second.

```
example.com CAA 0 issue "ca1.example; priority=2"
example.com CAA 0 issue "ca2.example; priority=1"
```

CAA records that do not explicitly specify a priority are automatically assigned the highest priority, which is 0. In cases where multiple CAA records have the same priority, the usage will be randomized.

Consider the following example, where the domain "example.com" has three CAA records. The CAA record with "ca1.example" has no specified priority, and thus it is assigned the highest priority of 0. The CAA records with "ca2.example" and "ca3.example" both have a priority of 1. In this scenario, the ACME client will first attempt to obtain its configuration from "ca1.example". If that fails, it will randomly select either "ca2.example" or "ca3.example" and attempt to obtain the configuration. If all attempts fail, the certificate issuance will ultimately fail.

```
example.com CAA 0 issue "ca1.example"
example.com CAA 0 issue "ca2.example; priority=1"
example.com CAA 0 issue "ca3.example; priority=1"
```

Furthermore, it is possible to configure CAA records to indicate a preference for specific types of certificates. In the following example, the domain "example.com" prefers Extended Validation (EV) certificates issued by "ca1.example". If the issuance of an EV certificate fails, the ACME client will attempt to obtain any type of certificate from "ca1.example". If that also fails, it will then try to obtain any type of certificate from "ca2.example".

```
example.com CAA 0 issue "ca1.example; validationmethods=ca-ev"
example.com CAA 0 issue "ca1.example; priority=1"
example.com CAA 0 issue "ca2.example; priority=2"
```

Implementers and operators should carefully configure CAA records according to their specific requirements and considerations.
// COMMENT: If a mechanism for enabling or disabling auto-discovery is required, users may need to configure their preferences accordingly. While enabling auto-discovery by default could promote adoption, it could lead to unexpected certificate issuance (see the security considerations). For instance, in the given example, if the default setting is set to false, only CA 2 will be used to retrieve the ACME client configuration.

```
example.com CAA 0 issue "ca1.example"
example.com CAA 0 issue "ca2.example; discovery=true"
```

# ACME Client Configuration

To enable the ACME client to obtain the necessary configuration information for interacting with the authorized Certification Authority (CA)'s ACME server, a mechanism leveraging the well-known directory is proposed.

The well-known directory is a standardized location within the domain's web server where clients can discover specific resources or configurations. In the context of ACME client configuration retrieval, a copy of the ACME directory object or a redirect to it is placed in the well-known directory of the CA's domain, which is specified as a constraint in the CAA record. This allows the ACME client to conveniently retrieve the required configuration.

For instance, when the CAA record restricts certificate issuance to the CA "ca.example" for the domain "example.com", the ACME client retrieves the ACME directory object as specified in Section 7.1.1 of ACME [RFC 8555] from the URL "https://ca.example/.well-known/acme".

While an alternative consideration was to include the ACME server address directly as an attribute in the CAA record, it was determined that this approach could introduce clutter and significantly increase the size of the record. Additionally, a rigid binding between the CAA record and the ACME server address may present challenges if the CA needs to change its server address in the future.

Thus, the approach outlined in this document, utilizing the well-known directory for ACME client configuration retrieval, offers flexibility for CAs to manage and update their ACME server addresses while maintaining a concise and focused CAA record.
It is important for implementers and operators to ensure the availability and accessibility of the ACME directory object within the well-known directory to facilitate successful ACME client configuration retrieval.

# ACME Client Behavior {#sec-behavior}

The process looks as follows:

1. The ACME client initiates a DNS lookup to retrieve the CAA record(s) according to RFC 8659.
  a. The DNS resolver responds with the CAA record for each domain, specifying the authorized CAs capable of issuing certificates, along with their priorities and other optional parameters.
2. The ACME client analyzes the CAA records for the domain and selects the CA with the highest priority.
3. The ACME client will download the ACME directory from the well-known location of the issuer-domain-name of the selected CA (https://[issuer-domain-name]/.well-known/acme)
4. If an External Account Binding is required but not configured the ACME client will try to determine an alternative common CA in step 2.
  a. If no alternative CA can be found, the process with end with a failure and the user will be informed.
5. The ACME client proceeds with the ACME challenge process, where it interacts with the ACME server to complete the required validation steps.
6. Upon successful completion of the challenge, the ACME client sends a finalize request to the ACME server, indicating the completion of the certificate issuance process.
7. The ACME server processes the request and issues the certificate.
8. The ACME client receives the issued certificate from the ACME server.
9. The certificate is ready for use by the ACME client for the specified domain(s).

Prior to establishing a connection with the default ACME server or a pool of ACME servers, the ACME client verifies the presence of any configured CA Authorization records (CAA) as defined in RFC 8659. If a CAA record is found, the ACME client will attempt to obtain a certificate from the CA with the highest priority. If the certificate issuance attempt fails, the client will proceed to lower-priority CAs in an attempt to obtain the certificate.

In the event of a failed attempt to obtain a certificate from a particular CA, the ACME client employs a retry mechanism to ensure successful certificate acquisition. However, in cases where certain CAs are known to be temporarily unavailable, the ACME client MAY choose to ignore those CAs for a limited period of time. By temporarily excluding unresponsive CAs from the issuance process, the client can optimize its certificate acquisition strategy and enhance overall efficiency. This approach helps mitigate potential delays caused by unresponsive CAs and allows the client to focus on viable options for obtaining the required certificate.

ACME clients MUST notify the user if the enrollment of a certificate from a specific CA fails multiple times, even if the client successfully obtains a certificate from an alternative CA. This notification is essential to ensure that users are promptly informed about recurring enrollment failures, allowing them to take appropriate measures. By providing such notifications, clients enable users to assess and address any underlying issues, seek alternative solutions, or make informed decisions regarding their certificate management processes.

In order to promote the adoption of ACME in Enterprise environments, it is crucial for implementers and operators to recognize the significance of External Account Bindings. The section dedicated to External Account Bindings provides valuable information and guidelines for effectively incorporating this feature.

## Certificates with multiple domain names

When the ACME client initiates a certificate request for multiple domain names, it is required to check the CAA records for each domain. The purpose of this check is to identify a Certification Authority (CA) that is authorized by all the domain names intended to be included in the certificate. If, during the evaluation of CAA records, no common CA can be identified that satisfies the authorization requirements of all the domain names, the certificate issuance process will fail.

To mitigate the risk of encountering failures in the certificate issuance process due to incompatible CAA records, it is crucial to ensure that certificates only include domain names that are under the control of the same entity. By maintaining a consistent ownership and control of the domain names included in a certificate, the likelihood of encountering authorization conflicts among CAA records is minimized. This practice promotes a more streamlined and reliable certificate issuance process, reducing the potential for errors and ensuring that the certificate accurately represents the domains controlled by a single entity.

The process with multiple domain names looks as follows:
1. The ACME client identifies the list of domain names for which a certificate is requested.
2. For each domain in the list, the ACME client initiates a DNS lookup to retrieve the CAA record(s) according to RFC 8659.
  a. The DNS resolver responds with the CAA record for each domain, specifying the authorized CAs capable of issuing certificates, along with their priorities and other optional parameters.
3. The ACME client analyzes the CAA records for all domains to identify a common CA that is authorized by all included domains and has the highest priority.
  a. If a common CA is found, the ACME client proceeds with step 4.
  b. If no common CA is found, the ACME client tries to find a compromise using as few as possible domains with a lower priority.
  c. If no compromise can be found, the process will end with a failure and the user will be informed.
4. The ACME client will download the ACME directory from the well-known location of the issuer-domain-name of the selected common CA (https://[issuer-domain-name]/.well-known/acme)
5. If an External Account Binding is required but not configured the ACME client will try to determine an alternative common CA in step 3.
  a. If no alternative CA can be found, the process with end with a failure and the user will be informed.
6. The ACME client proceeds with the ACME challenge process, where it interacts with the ACME server to complete the required validation steps.
7. Upon successful completion of the challenge, the ACME client sends a finalize request to the ACME server, indicating the completion of the certificate issuance process.
8. The ACME server processes the request and issues the certificate.
9. The ACME client receives the issued certificate from the ACME server.
10. The certificate is ready for use by the ACME client for the specified domain(s).

# External Account Binding

Clients SHOULD provide users with the ability to configure and utilize external account bindings per CA or ACME server, as it offers enhanced security and flexibility in managing the certificate provisioning process.

External account bindings are not only crucial for users seeking to provision certificates with stronger authentication requirements such as OV, EV, and QWAC, but they can also be applicable to Domain Validation (DV) certificates covered by a commercial agreement. By offering the configuration option for external account bindings, clients enable users to establish a secure association between their ACME accounts and external accounts, facilitating streamlined and authenticated certificate issuance processes. This flexibility accommodates a wide range of certificate types and use cases, ensuring that users can provision certificates with the appropriate level of authentication based on their specific requirements, whether they be DV certificates covered by a commercial agreement or certificates with higher levels of validation. Therefore, it is essential for clients to implement the external account binding configuration option to support the diverse needs of users in obtaining certificates with varying authentication levels.

It is crucial for Certification Authorities (CAs) to carefully consider the internal account binding mechanisms described in this document. This is especially critical given the current lack of widespread support for external account bindings in user interfaces, with only a few command line utilities offering such functionality. By recognizing and implementing the internal account binding approach, CAs can provide a viable alternative for users who may not have access to or be familiar with external account binding options. This will help ensure a seamless and secure account linkage process, even in situations where the availability of external account binding configurations is limited.

## Internal Account Binding using Domain Control Validation

In addition to the external account binding mechanism, an alternative approach can be implemented by the CAs that offers distinct advantages, particularly in cases where service providers may not expose the account binding configuration options to their users. This alternative method leverages domain control validation as the initial step in the process and subsequently pauses to await confirmation from the account holder regarding the account binding. The email address associated with the account can be utilized for differentiating between multiple Certification Authority (CA) accounts holding the same domain name.
The process to establish an internal account binding would be as follows:

1. The user adds the domain name to their Certification Authority (CA) account and completes the domain control verification process within this account.
2. The user initiates the ACME process for certificate issuance or a new authorization.
3. The ACME client validates domain control for the requested domain.
4. Upon successful domain control validation the CA does not mark the authorizations as completed but awaits the completion of the account binding.
5. The CA, having confirmed domain control, uses the provided email address associated with the ACME account to distinguish between different CA accounts that have confirmed control over the same domain name.
6. The account holder receives a notification or prompt and confirms the account binding within the CA's system.
7. Once the account binding is confirmed, the CA marks the ACME authorization as completed and the ACME client proceeds with the remaining steps of the ACME process, such as finalizing the certificate issuance.

It is important to note that before the ACME process can start, the domain name must be added to the CA account and the domain control validation process must be successfully completed. This ensures that the domain ownership is verified for both accounts before proceeding with the account binding.

## Internal Account Binding using Email Address

When ACME clients provide the email address associated with the user's account during the creation of a new account, Certification Authorities (CAs) can utilize this email address to establish an internal account binding between the ACME account and the corresponding customer account within their internal systems. This approach offers an alternative method for establishing the account linkage, particularly in cases where the ACME integration's user interface does not provide explicit external account binding configuration options.

However, it is crucial to acknowledge that this internal account binding mechanism introduces potential vulnerabilities, particularly in relation to phishing attacks. It is imperative to exercise caution when utilizing this mechanism since the email address associated with the ACME account is not verified, and the account binding request can be initiated by any party. Careful consideration should be given to the security implications of relying solely on the email address for establishing the account linkage.

# IANA Considerations {#sec-iana}

##  Well-Known URI for the ACME Directory

The following value has been registered in the "Well-Known URIs" registry (using the template from [RFC5785]):

```
URI suffix: acme
Change controller: IETF
Specification document(s): RFC XXXX, Section Y.Z
Related information: N/A
```

// TODO: add CAA attributes (not sure if these can be registered)

<!-- End of IANA Considerations section -->

# Security Considerations

## Risks with Auto-Discovery of Authorized CAs

The mechanism described in this document relies on the DNS Certification Authority Authorization (CAA) Resource Record [RFC 8659] to determine the authorized Certification Authorities (CAs) capable of issuing certificates for a given domain name(s). However, there are potential risks associated with the automatic provisioning of certificates without an explicit indication from the user.

### Unexpected Certificate Issuance

Where the issuance of certificates is currently restricted through CAA records and certificates are provisioned through alternative means (i.e., manual or via a proprietary API) certificates can unexpectedly be replaced with a similar certificate or a certificate of a different type (e.g., DV versus EV) if the ACME client supports this new mechanism.

Its recommended that users who which to obtain certificates attesting to more than domain validation (DV) control, restrict the validation method using a CA specific “validationmethods” CAA parameter value (e.g., “ca-ov”, “ca-ev”, “ca-qwac”) as specified by RFC 8657.

### Issuance by the ‘wrong’ authorized CA

In scenarios where a domain name authorizes multiple CAs without specifying a weight or preference attribute, there is a risk that the ACME client may unexpectedly request a certificate from one of the authorized CAs that was only included as backup.
To mitigate this risk, it is recommended that users who have multiple CAA records explicitly configure the CAA record to include a weight or preference attribute to indicate their desired CA for certificate issuance.

Additionally, ACME clients should provide clear visibility and feedback to users regarding the CA from which certificates will be obtained, ensuring that it aligns with their expectations.

## Terms of Service and Acceptance

The terms of service associated with different CAs can vary, and it is important to consider how these terms are handled within the context of auto-discovery.

### Implicit Acceptance of Terms of Service

As the ACME client is not explicitly controlled by the user in a shared environment, the user's explicit approval of the terms of service presented by the CA becomes challenging. In the absence of a direct user interaction with the ACME client, it is assumed that the user accepts the terms of service by explicitly configuring the CAA record to authorize the CA.

ACME clients are strongly encouraged to display the relevant terms of service for the obtained certificates to ensure users have visibility into the associated obligations and restrictions. This helps users make informed decisions about their certificate management and ensures compliance with the terms of service set by the authorized CA.

### Acceptance Through CAA Attribute

One potential enhancement to address the explicit acceptance of terms of service is the inclusion of a CAA attribute called "termsOfServiceAgreed". This attribute would provide a direct mechanism for users to indicate their agreement to the terms of service.

However, it is important to consider the trade-offs associated with adding this type of data to the CAA record. The inclusion of additional attributes can be perceived as clutter and may increase the complexity of configuring the CAA record. Therefore, the authors of this document recommend relying on the implicit acceptance of the terms of service.

By configuring the CAA record to authorize a specific CA, users implicitly indicate their acceptance of the associated terms of service. This approach strikes a balance between simplicity and compliance with the CA's requirements. It is crucial for ACME clients to display the relevant terms of service for the obtained certificates, ensuring that users have visibility and can make informed decisions regarding their certificate management.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
