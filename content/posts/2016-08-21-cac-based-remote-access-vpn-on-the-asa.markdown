---
author: rn.wolfe
date: 2016-08-21 17:22:40+00:00
draft: false
title: CAC Based Remote Access VPN on the ASA
type: post
url: /2016/08/21/cac-based-remote-access-vpn-on-the-asa/
categories:
  - Tech
tags:
  - asa
  - cac
  - certificate
  - configure
  - how to
  - lua
  - piv
  - smartcard
  - vpn
---

## Introduction

In general, using CACs (or smartcards, PIV cards, etc.) as the authentication mechanism is pretty straightforward. The certificate is used for authentication, and, if desired, authorization can then be performed using a value in the certificate. The ASA essentially pulls a username from a field to use for a lookup against a backend server, e.g. LDAP or RADIUS.

I'm not going to cover the configuration and setup of the initial VPN group policies, tunnel groups, etc. This is simply going to review authorizing clients based on the username in the certificate using a couple different methods to grab the username. The backend authentication server configuration will not be covered. In this example, LDAP is being used directly as backend source.

## Selecting the Username from a Certificate Field

In most cases, the ASA can extract the username from the certificate with ease. The following cases will require a straightforward configuration:

- The value of the field is identical to the value in the backend server

- This is the most common solution, as the User Principal Name (UPN) generally provides the same value as the UPN assigned value in the backend server when the CAC certificate is issued.

* The needed value is easily identifiable in the certificate field using pattern matching, e.g. the only 10 digits in a field, or the 10th through the 15th characters.

These scenarios account for the most common, and simplest, configuration scenarios.

### How to Configure

These simple scenarios can be configured right from the ASDM GUI. This can not actually be configured from the CLI. On ASDM, navigate to **Remote Access VPN > AnyConnect Connection Profile > [vpn-profile] > Advanced > Authentication > “Use script to select username” > Add**. In this prompt, you can select which field to use, as well as some filtering options, if applicable.

{{< smallimg src="/images/cac-vpn-use-script-upn.png" width="70%" >}}

### Under the Hood

So, under the hood, when selecting which value to select for the username, the ASA is actually creating a script for you. This is stored on the **disk0:/** partition in a file called **username_from_cert.xml**. The scripting is done using a language called [Lua](http://www.lua.org/); however, the ASA uses some local functions that aren't exposed to the administrator to perform the specified actions in the ASDM. These can be seen by examining the created script when specifying different parameters.

#### Example 1

Settings:
{{< smallimg src="/images/cac-vpn-use-script-upn-1.png" width="70%" >}}

Resulting Code:

    <TunnelGroupData>
        <TableEntry>
            <TunnelGroup>ipsecvpn</TunnelGroup>
            <Function>CAC_UPN</Function>
        </TableEntry>
        <Function>
            <ScriptName>CAC_UPN</ScriptName>
            <Custom>false</Custom>
            <Content>return CSCO_CertificateToUser(cert.subjectaltname.upn)</Content>
        </Function>
    </TunnelGroupData>

#### Example 2

Settings:
{{< smallimg src="/images/cac-vpn-use-script-upn-filtering.png" width="70%" >}}

Resulting Code:

    ciscoasa# more disk0:/username_from_cert.xml
    <TunnelGroupData>
        <TableEntry>
            <TunnelGroup>ipsecvpn</TunnelGroup>
            <Function>CAC_UPN</Function>
        </TableEntry>
        <Function>
            <ScriptName>CAC_UPN</ScriptName>
            <Custom>false</Custom>
            <Content>return CSCO_CertificateToUser(cert.subjectaltname.upn,1,10)</Content>
        </Function>
    </TunnelGroupData>

## Less Common Scenarios

In some less common scenarios, the use of the script can become challenging. The only scenario I came across requiring a custom script is outlined below.

### Appending a Suffix to a Field's String

The use case for this was a customer who's CACs had a null value for the UPN field; however, needed the EDIPI value that is normally stored there to perform the LDAP lookup. To make things more complicated, the UPN field should have been in a format similar to 0123456789@domain.com. Since the UPN was blank, we ended up grabbing the EDIPI from the common name (CN) value which was mixed up with the user's name full name and lacked the suffix we needed. So, we matched the only continuous string of digits and then appended our domain on the end of it.

The LUA scripting language actually uses its own implementation of pattern matching as opposed to run of the mill regular expressions. This created quite the challenge. In addition, scripts that were working successfully on my machine would throw errors when attempting to use them on the ASA with the built in variable referencing the cert field (cert.subject.cn in this case). In the end, we ended up using the pre-existing Cisco function to match the string of digits and appending ("..") a string to the end of it.

    return (CSCO_CertificateToUser(cert.subject.cn, "%d+")).."@test.somewolfe.com"

### Debugging Custom Scripts

When using custom scripts, often times error are thrown that are related to the script not successfully running. This doesn't require any level of debugging, but generates a warning (4) syslog message as seen below:

    %ASA-7-717025: Validating certificate chain containing 1 certificate(s).
    %ASA-7-717029: Identified client certificate within certificate chain. serial number: 4E819769406A4BEE8D9825C9E1052208, subject name: c=US,st=MD,l=Fort Meade,o=Military,ou=NEC,cn=RYAN.WOLFE.9101161615.
    %ASA-7-717030: Found a suitable trustpoint ISE_VPN_TrustPoint7 to validate certificate.
    %ASA-6-717022: Certificate was successfully validated. serial number: 4E819769406A4BEE8D9825C9E1052208, subject name:  c=US,st=MD,l=Fort Meade,o=Military,ou=NEC,cn=RYAN.WOLFE.9101161615.
    %ASA-6-717028: Certificate chain was successfully validated with warning, revocation status was not checked.
    %ASA-6-725002: Device completed SSL handshake with client inside:10.202.190.133/50536
    %ASA-7-717036: Looking for a tunnel group match based on certificate maps for peer certificate with serial number: 4E819769406A4BEE8D9825C9E1052208, subject name: c=US,st=MD,l=Fort Meade,o=Military,ou=NEC,cn=RYAN.WOLFE.9101161615, issuer_name: cn=Certificate Services Endpoint Sub CA - ise20.
    %ASA-4-717037: Tunnel group search using certificate maps failed for peer certificate: serial number: 4E819769406A4BEE8D9825C9E1052208, subject name: c=US,st=MD,l=Fort Meade,o=Military,ou=NEC,cn=RYAN.WOLFE.9101161615, issuer_name: cn=Certificate Services Endpoint Sub CA - ise20.
    %ASA-7-113028: Extraction of username from VPN client certificate has been requested.  [Request 7]
    %ASA-7-113028: Extraction of username from VPN client certificate has started.  [Request 7]
    %ASA-4-113026: Error <[string "user_from_cert_xlate_lua"]:264: No function found> while executing Lua script for group <ipsecvpn>
    %ASA-2-113027: Error activating tunnel-group scripts
    %ASA-7-113028: Extraction of username from VPN client certificate has finished with error.  [Request 7]
    %ASA-7-113028: Extraction of username from VPN client certificate has completed.  [Request 7]
    %ASA-7-609001: Built local-host management:10.202.192.10
    %ASA-6-302013: Built outbound TCP connection 17135 for management:10.202.192.10/389 (10.202.192.10/389) to identity:10.202.192.3/9892 (10.202.192.3/9892)

    [88] Session Start
    %ASA-6-113005: AAA user authorization Rejected : reason = Unspecified : server = 10.202.192.10 : user = <Unknown>

Following that, if the script runs successfully and does get a value, you can debug the LDAP process, or whatever authentication process that is in use, to see what value is being used for the lookup. For LDAP, debugging is enabled using "debug ldap 254". You can see below that the string being used to filter is displayed:

    [108] Performing Simple authentication for Administrator to 10.202.192.10
    [108] LDAP Search:
    Base DN = [DC=test,DC=somewolfe,DC=com]
    Filter  = [userPrincipalName=9101161615@test.somewolfe.com]
    Scope   = [SUBTREE]
    [108] User DN = [CN=Ryan Wolfe,OU=Test Users,DC=test,DC=somewolfe,DC=com]

If the LDAP query is failing with with that string, you may need to validate that the value in Active Directory matches what you are looking for. After a successful attempt, you should see LDAP return a slew of attributes about the user you looked up.

    [108] Talking to Active Directory server 10.202.192.10
    %ASA-6-302014: Teardown TCP connection 17292 for inside:10.202.190.133/50796 to identity:10.202.191.20/443 duration 0:00:00 bytes 10719 TCP FINs
    %ASA-7-609002: Teardown local-host inside:10.202.190.133 duration 0:00:00
    %ASA-7-609002: Teardown local-host identity:10.202.191.20 duration 0:00:00
    [108] Reading password policy for 9101161615@test.somewolfe.com, dn:CN=Ryan Wolfe,OU=Test Users,DC=test,DC=somewolfe,DC=com
    [108] Read bad password count 0
    [108] LDAP Search:
    Base DN = [DC=test,DC=somewolfe,DC=com]
    Filter  = [userPrincipalName=9101161615@test.somewolfe.com]
    Scope   = [SUBTREE]
    [108] Retrieved User Attributes:
    [108]   objectClass: value = top
    [108]   objectClass: value = person
    [108]   objectClass: value = organizationalPerson
    [108]   objectClass: value = user
    [108]   cn: value = Ryan Wolfe
    [108]   sn: value = Wolfe
    [108]   givenName: value = Ryan
    [108]   distinguishedName: value = CN=Ryan Wolfe,OU=Test Users,DC=test,DC=somewolfe,DC=com
    [108]   instanceType: value = 4
    [108]   whenCreated: value = 20151116202638.0Z
    [108]   whenChanged: value = 20160226023444.0Z
    [108]   displayName: value = Ryan Wolfe
    [108]   uSNCreated: value = 210819
    [108]   memberOf: value = CN=ISE Admins,OU=Test Groups,DC=test,DC=somewolfe,DC=com
    [108]   memberOf: value = CN=Domain Admins,CN=Users,DC=test,DC=somewolfe,DC=com
    [108]   uSNChanged: value = 244462
    [108]   name: value = Ryan Wolfe
    [108]   objectGUID: value = .z&d.QjF.`....ny
    [108]   userAccountControl: value = 66048
    [108]   badPwdCount: value = 0
    [108]   codePage: value = 0
    [108]   countryCode: value = 0
    [108]   badPasswordTime: value = 0
    [108]   lastLogoff: value = 0
    [108]   lastLogon: value = 0
    [108]   pwdLastSet: value = 131009121452888556
    [108]   primaryGroupID: value = 513
    [108]   objectSid: value = ..............M.6.F.R4.Jh...
    [108]   adminCount: value = 1
    [108]   accountExpires: value = 9223372036854775807
    [108]   logonCount: value = 0
    [108]   sAMAccountName: value = rwolfe
    [108]   sAMAccountType: value = 805306368
    [108]   userPrincipalName: value = 9101161615@test.somewolfe.com
    [108]   objectCategory: value = CN=Person,CN=Schema,CN=Configuration,DC=test,DC=somewolfe,DC=com
    [108]   dSCorePropagationData: value = 20160226021603.0Z
    [108]   dSCorePropagationData: value = 16010101000000.0Z
    [108]   lastLogonTimestamp: value = 131009098327345443
    [108] Fiber exit Tx=626 bytes Rx=4547 bytes, status=1
    [108] Session End

This should be followed, of course, by a successful VPN connection.
