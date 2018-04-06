+++
title = "setting up a ca"
date = "2016-01-30"
slug = "2016/01/30/setting-up-a-ca"
Categories = []
+++

Tired of creating your CA using openssl command line tools? Here's a whirlwind crash course on creating a functional web-based CA in a couple of minutes.

<!-- more -->

# Preparation

I'm assuming you're running CentOS 7, because that's what I'm using. Also, the code is redhat-ish in that it's in the Fedora/RHEL pipeline and not terribly friendly to other distros as far as packaging goes. Also, I happen to like CentOS.

We're going to be installing the Dogtag CA PKI solution. The folks at Fedora/RedHat are using it as the backing system for FreeIPA (which in turn becomes RedHat Identity management), however alot of the UI bits of the system are somewhat broken in minor ways. Doesn't really impact the functionality, however if you were planning an enterprise deployment, I'd probably spend some time finding/fixing all these annoyances to give the installation some polish. Or you could buy the RHEL product whenever it actually comes out.

But I digress...

First, we need to setup a code repository to get some working packages. There's no current version of it for RHEL/CentOS, so I built some packages to speed things up.

Getting the repo:

```
wget -O /etc/yum.repos.d/marksteele.repo http://www.control-alt-del.org/repo/mark.repo
```

Getting the package signing GPG key:

```
rpm --import http://www.control-alt-del.org/repo/MARK_STEELE_REPO_GPG_KEY
```

C'mon, you trust me right?

After the repo is setup, make sure that your DNS resolution is setup correctly. In this example, my hostname will be `dev1.control-alt-del.org`.

# Installation

You'll want to be running as root for the following commands. We start with installing packages.

```
yum install pki-* 389-ds-base
```

Next, we'll be setting up the LDAP server.

```
setup-ds.pl --silent General.FullMachineName=dev1.control-alt-del.org \
General.SuiteSpotUserID=nobody General.SuiteSpotGroup=nobody \
slapd.ServerPort=389 slapd.ServerIdentifier=pki-tomcat \
slapd.Suffix=dc=control-alt-del,dc=org \
slapd.RootDN="cn=diradmin" slapd.RootDNPwd=letmein
```

Important to note that you need to customize the above command for your environment.

You'll probably want to change:

* `General.FullMachineName`: The hostname of the machine you're installing this on.
* `slapd.Suffix`: The LDAP root hierarchy for this organization. In my case, my org is `dc=control-alt-del,dc=org`.
* `slapd.RootDNPwd`: The LDAP admin password. You'll need this later.
* `slapd.RootDN`: The LDAP admin username. You'll need this later.

Once the directory service is running, we want to configure the PKI services. We'll start with the Certificate Authority.

```
[root@dev1]# pkispawn

IMPORTANT:

    Interactive installation currently only exists for very basic deployments!

    For example, deployments intent upon using advanced features such as:

        * Cloning,
        * Elliptic Curve Cryptography (ECC),
        * External CA,
        * Hardware Security Module (HSM),
        * Subordinate CA,
        * etc.,

    must provide the necessary override parameters in a separate
    configuration file.

    Run 'man pkispawn' for details.

Subsystem (CA/KRA/OCSP/TKS/TPS) [CA]:

Tomcat:
  Instance [pki-tomcat]:
  HTTP port [8080]:
  Secure HTTP port [8443]:
  AJP port [8009]:
  Management port [8005]:

Administrator:
  Username [caadmin]:
  Password:
  Verify password:
  Import certificate (Yes/No) [N]?
  Export certificate to [/root/.dogtag/pki-tomcat/ca_admin.cert]:
Directory Server:
  Hostname [dev1.control-alt-del.org]:
  Use a secure LDAPS connection (Yes/No/Quit) [N]?
  LDAP Port [389]:
  Bind DN [cn=Directory Manager]: cn=diradmin
  Password:
  Base DN [o=pki-tomcat-CA]:

Security Domain:
  Name [control-alt-del.org Security Domain]:

Begin installation (Yes/No/Quit)? yes

Log file: /var/log/pki/pki-ca-spawn.20160129144945.log
Installing CA into /var/lib/pki/pki-tomcat.
Storing deployment configuration into /etc/sysconfig/pki/tomcat/pki-tomcat/ca/deployment.cfg.
Notice: Trust flag u is set automatically if the private key is present.
Created symlink from /etc/systemd/system/multi-user.target.wants/pki-tomcatd.target to /usr/lib/systemd/system/pki-tomcatd.target.

    ==========================================================================
                                INSTALLATION SUMMARY
    ==========================================================================

      Administrator's username:             caadmin
      Administrator's PKCS #12 file:
            /root/.dogtag/pki-tomcat/ca_admin_cert.p12

      To check the status of the subsystem:
            systemctl status pki-tomcatd@pki-tomcat.service

      To restart the subsystem:
            systemctl restart pki-tomcatd@pki-tomcat.service

      The URL for the subsystem is:
            https://dev1.control-alt-del.org:8443/ca

      PKI instances will be enabled upon system boot

    ==========================================================================
```

Next, we'll setup the OCSP responder.


```
[root@dev1]# pkispawn

IMPORTANT:

    Interactive installation currently only exists for very basic deployments!

    For example, deployments intent upon using advanced features such as:

        * Cloning,
        * Elliptic Curve Cryptography (ECC),
        * External CA,
        * Hardware Security Module (HSM),
        * Subordinate CA,
        * etc.,

    must provide the necessary override parameters in a separate
    configuration file.

    Run 'man pkispawn' for details.

Subsystem (CA/KRA/OCSP/TKS/TPS) [CA]: OCSP

Tomcat:
  Instance [pki-tomcat]:

Administrator:
  Username [ocspadmin]:
  Password:
  Verify password:
  Import certificate (Yes/No) [Y]?
  Import certificate from [/root/.dogtag/pki-tomcat/ca_admin.cert]:
  Export certificate to [/root/.dogtag/pki-tomcat/ocsp_admin.cert]:
Directory Server:
  Hostname [dev1.control-alt-del.org]:
  Use a secure LDAPS connection (Yes/No/Quit) [N]?
  LDAP Port [389]:
  Bind DN [cn=Directory Manager]: cn=diradmin
  Password:
  Base DN [o=pki-tomcat-OCSP]:

Security Domain:
  Hostname [dev1.control-alt-del.org]:
  Secure HTTP port [8443]:
  Name: control-alt-del.org Security Domain
  Username [caadmin]:
  Password:

Begin installation (Yes/No/Quit)? yes

Log file: /var/log/pki/pki-ocsp-spawn.20160130211923.log
Installing OCSP into /var/lib/pki/pki-tomcat.
Storing deployment configuration into /etc/sysconfig/pki/tomcat/pki-tomcat/ocsp/deployment.cfg.

    ==========================================================================
                                INSTALLATION SUMMARY
    ==========================================================================

      Administrator's username:             ocspadmin

      To check the status of the subsystem:
            systemctl status pki-tomcatd@pki-tomcat.service

      To restart the subsystem:
            systemctl restart pki-tomcatd@pki-tomcat.service

      The URL for the subsystem is:
            https://dev1.control-alt-del.org:8443/ocsp

      PKI instances will be enabled upon system boot

    ==========================================================================
```

Now that this is up and running, let's display our root CA certificate.

```
[root@dev1]# pki cert-show 1 --encoded
-----------------
Certificate "0x1"
-----------------
  Serial Number: 0x1
  Issuer: CN=CA Signing Certificate,O=control-alt-del.org Security Domain
  Subject: CN=CA Signing Certificate,O=control-alt-del.org Security Domain
  Status: VALID
  Not Before: Sat Jan 30 21:18:55 EST 2016
  Not After: Wed Jan 30 21:18:55 EST 2036

-----BEGIN CERTIFICATE-----
MIIDyDCCArCgAwIBAgIBATANBgkqhkiG9w0BAQsFADBPMSwwKgYDVQQKDCNjb250
cm9sLWFsdC1kZWwub3JnIFNlY3VyaXR5IERvbWFpbjEfMB0GA1UEAwwWQ0EgU2ln
bmluZyBDZXJ0aWZpY2F0ZTAeFw0xNjAxMzEwMjE4NTVaFw0zNjAxMzEwMjE4NTVa
ME8xLDAqBgNVBAoMI2NvbnRyb2wtYWx0LWRlbC5vcmcgU2VjdXJpdHkgRG9tYWlu
MR8wHQYDVQQDDBZDQSBTaWduaW5nIENlcnRpZmljYXRlMIIBIjANBgkqhkiG9w0B
AQEFAAOCAQ8AMIIBCgKCAQEAmj154Grlx1oEJTi/AXDk6ncifmpq5/RqNxexh6Yh
3ErtwIWWHT4G/beO7BGiep36OpaGWWnONhnk1b5RKn2fpEK3j31s75ZTwLwQXwi6
/9aEU+wKscWAVcb3DsVpoj7XiNpjVwDou3qBROvQY3jyqQYNIVQ7W61XgeErJYpG
D9z3Kok8qLloKMTvKCO/yyO3FNQkaJPHK5HNbgkhZTGPwKJfqLNRcxMiUFqgeO17
dEGHdrLe7n3n0E2gUO64g9jppQ8MEwN4JXiUK4K9UX2g96+xl5aQSClA+ialXBua
V09t9rpQJEnq3uXrfUAdqd8N55+XGM27fCzkZPMKS4P0MQIDAQABo4GuMIGrMB8G
A1UdIwQYMBaAFOiP+NkCPQUthP7kjFUe3bduVclzMA8GA1UdEwEB/wQFMAMBAf8w
DgYDVR0PAQH/BAQDAgHGMB0GA1UdDgQWBBToj/jZAj0FLYT+5IxVHt23blXJczBI
BggrBgEFBQcBAQQ8MDowOAYIKwYBBQUHMAGGLGh0dHA6Ly9kZXYxLmNvbnRyb2wt
YWx0LWRlbC5vcmc6ODA4MC9jYS9vY3NwMA0GCSqGSIb3DQEBCwUAA4IBAQAFfRzW
bv80UKXzm4CF9eMrXaxCxIKc/H3Qaq6fY3HcdN1hCXCkncm8gmZsqJIPbtrN8vgz
MXx32ZDEVkqXhajV8h52EWpqLoF7fgXzZ/dcy7Uv5yDi5utinI9AutYE6gN1oQUf
vVOeKYAYP28CiS1DpisW1Z8z/uJi+fyua0BzxzVWQkw0+JcfAD4BWKsWxn/HGukw
yqElj2bq67EqS6Dfi4yMabID7WDXTAfa+o37AU/MGZ3aj2akHVNXR3HxxQv9zbRB
qstFbKG8RnzFCGYC03npcgd5wNaaiQE5XWmlqnXV7lJLN7JaXQaZvPWjO6HDt+QT
ltHvYjjh6dOJCQjr
-----END CERTIFICATE-----
```

Copy from `-----BEGIN CERTIFICATE-----` to `-----END CERTIFICATE-----` into a text file named CA.crt. My Desktop is a windows machine, so I import the root CA cert using the commands:

* `start -> run -> certmgr.msc`
* Click `import -> browse to the path you saved the file in -> next -> specify the store to save the cert as the trusted root CA store -> finish`

Now we'll also want to import the CA admin certificate into our desktop certificate store. The server path is:

`/root/.dogtag/pki-tomcat/ca_admin_cert.p12`

Copy that file onto your desktop somewhere. Then (on windows):

* `start -> run -> certmgr.msc`
* `import -> browse to the path you saved the p12 file (select the file type dropdown to get p12) -> next -> select the store as personal -> finish`

The process is similar with MacOS using the Keyring.

Restarting your browser is probably a good idea at this point if it's running.

Finally, you are now ready to login to your shiny new CA server.

Hit the URL mentioned during the installation. It will look something like:

`https://dev1.control-alt-del.org:8443/`

Your browser should prompt you to select a client certificate, which you can now select.

Once logged in you'll have two options: 

* Certificate authority (go here to submit certificate signing requests, sign certificates, revoke certs, etc...)
* Online Certificate Status Protocol Manager (go here to manage the OCSP responder)

The rest should be fairly self-explanatory.

One final note, is that out of the box, the certificate signing profiles that Dogtag include an OCSP responder URI, but no CRL URL.

I found this pretty annoying, and fixed the profile. To do this:

Edit the file `/var/lib/pki/pki-tomcat/ca/profiles/ca/caServerCert.cfg`:

Change

```
policyset.serverCertSet.list=1,2,3,4,5,6,7,8
```

To

```
policyset.serverCertSet.list=1,2,3,4,5,6,7,8,9
```

Then at the end of the file, add this (adjusting URL to fit your domain):

```
policyset.serverCertSet.9.constraint.class_id=noConstraintImpl
policyset.serverCertSet.9.constraint.name=No Constraint
policyset.serverCertSet.9.default.class_id=crlDistributionPointsExtDefaultImpl
policyset.serverCertSet.9.default.name=CRL Distribution Points Extension Default
policyset.serverCertSet.9.default.params.crlDistPointsCritical=false
policyset.serverCertSet.9.default.params.crlDistPointsNum=1
policyset.serverCertSet.9.default.params.crlDistPointsEnable_0=true
policyset.serverCertSet.9.default.params.crlDistPointsIssuerName_0=O=control-alt-del.org Security Domain, CN=CA Signing Certificate
policyset.serverCertSet.9.default.params.crlDistPointsIssuerType_0=DirectoryName
policyset.serverCertSet.9.default.params.crlDistPointsPointName_0=http://dev1.control-alt-del.org:8080/ca/ee/ca/getCRL?op=getCRL&crlIssuingPoint=MasterCRL
policyset.serverCertSet.9.default.params.crlDistPointsPointType_0=URIName
policyset.serverCertSet.9.default.params.crlDistPointsReasons_0=
```

Make sure you restart your PKI instance with

```
systemctl restart pki-tomcatd@pki-tomcat.service
```

# Creating certificate requests

Glad you asked! Adjust the following command to your taste:

```
openssl req -new -newkey rsa:4096 -nodes -out "test.control-alt-del.org.csr" -keyout "test.control-alt-del.org.key" -subj "/C=CA/ST=ON/L=Toronto/O=control-alt-del.org/OU=Security/CN=test.control-alt-del.org"
cat test.control-alt-del.org.csr
```

# Signing certificates

Navigate to

`https://dev1.control-alt-del.org:8443/ca/ee/ca`

Click on `Manual Server Certificate Enrollment` and paste the certificate signing request into the text input. Click submit.

Navigate to

`https://dev1.control-alt-del.org:8443/ca/agent/ca/`

Click `Find`. Click on the request you just submitted, and approve it. You can now copy the signed certificate from the resulting web page.

As an alternative, you can use the `pki` command from the command-line to do all this stuff. I haven't tested that though.

![that's all folks](http://img11.deviantart.net/920c/i/2014/137/a/2/pinkie_pie_thats_all_folks_by_dan232323-d7ipnd4.jpg)
