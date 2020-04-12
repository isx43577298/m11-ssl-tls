# Ldaperver19:tls

Ldap con conexiones seguras TLS/SSL  y startTLS.

## Generar cerificats
### Generem keys privades
```
[gustavo@localhost ldapserver19:tls]$ openssl genrsa -out key_server.pem
Generating RSA private key, 2048 bit long modulus (2 primes)
.......+++++
...+++++
e is 65537 (0x010001)

[gustavo@localhost ldapserver19:tls]$ openssl genrsa -out key_CA.pem
Generating RSA private key, 2048 bit long modulus (2 primes)
.......................+++++
.....................................+++++
e is 65537 (0x010001)
```

### Generem un certificat propi de l'entitat CA.
```
[gustavo@localhost ldapserver19:tls]$ openssl req -new -x509 -nodes -sha1 -days 365 -key key_CA.pem -out certif_CA.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:ca
State or Province Name (full name) []:Barcelona
Locality Name (eg, city) [Default City]:Barcelona
Organization Name (eg, company) [Default Company Ltd]:Veritat Absoluta
Organizational Unit Name (eg, section) []:Certificats covid19
Common Name (eg, your name or your server's hostname) []:Veritat_Absoluta   
Email Address []:admin@edt.org
```

### Generem un certificat request per enviar a l'entitat CA
```
[gustavo@localhost ldapserver19:tls]$ openssl req -new -key key_server.pem -out certif_server.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:ca
State or Province Name (full name) []:Barcelona
Locality Name (eg, city) [Default City]:Barcelona
Organization Name (eg, company) [Default Company Ltd]:M11-SAD
Organizational Unit Name (eg, section) []:Informatica
Common Name (eg, your name or your server's hostname) []:ldap.edt.org
Email Address []:admin@edt.org

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:request password
An optional company name []:edt
```

### Definim les extensions en el fitxer ca.conf.
```
[gustavo@localhost ldapserver19:tls]$ cat ca.conf
basicConstraints = critical,CA:FALSE
extendedKeyUsage = serverAuth,emailProtection
```

### L'autoritat CA ha de firmar el certificat
```
[gustavo@localhost keys]$ openssl x509 -CA certif_CA.pem -CAkey key_CA.pem -req -in certif_server.pem -days 365 -sha1 -extfile ca.conf -CAcreateserial -out crt_server.pem
Signature ok
subject=C = ca, ST = Barcelona, L = Barcelona, O = M11-SAD, OU = Informatica, CN = ldap.edt.org, emailAddress = admin@edt.org
Getting CA Private Key
```

# Configuració

## Fitxer slapd.conf
```
# Afegir abans de definir les databases, les següents lineas:
TLSCACertificateFile    /etc/openldap/certs/cacrt.pem
TLSCertificateFile      /etc/openldap/certs/servercrt.pem
TLSCertificateKeyFile   /etc/openldap/certs/serverkey.pem
TLSVerifyClient         never
TLSCipherSuite          HIGH:MEDIUM:LOW:+SSLv2
```

## Fitxer ldap.conf
```
# Comentar o eliminar la següent linea:
TLS_CACERTDIR /etc/openldap/certs
# i canviarla per aquesta
TLS_CACERT /etc/openldap/certs/cacrt.pem

# Afegir al final
URI ldap://ldap.edt.org
BASE dc=edt,dc=org
---
```

## Fitxer startup.sh
```
# Afegir la següent linea
/sbin/slapd -d0 -h "ldap:/// ldaps:/// ldapi:///"
```

## Fitxer /etc/hosts (client)
```
# Afegir la següent linea:
172.17.0.2	ldap.edt.org
```

# Comprovacions

## Engueguem el docker
```
docker run --rm --name ldap.edt.org -h ldap.edt.org -p 389:389 -p 636:636 -d isx43577298/ldapserver19:tls
```

## Si el client està ben configurat i la resolució al /etc/hosts també, les consultas TLS/SSL funcionaràn.
```
[gustavo@localhost ldapserver19:tls]$ ldapsearch -x -LLL -ZZ dn | head
dn: dc=edt,dc=org

dn: ou=maquines,dc=edt,dc=org

dn: ou=clients,dc=edt,dc=org

dn: ou=productes,dc=edt,dc=org

dn: ou=usuaris,dc=edt,dc=org


[gustavo@localhost ldapserver19:tls]$
[gustavo@localhost ldapserver19:tls]$ ldapsearch -x -LLL -ZZ -h ldap.edt.org -b 'dc=edt,dc=org' dn | head
dn: dc=edt,dc=org

dn: ou=maquines,dc=edt,dc=org

dn: ou=clients,dc=edt,dc=org

dn: ou=productes,dc=edt,dc=org

dn: ou=usuaris,dc=edt,dc=org


[gustavo@localhost ldapserver19:tls]$ ldapsearch -x -LLL -H ldaps://ldap.edt.org dn | head
dn: dc=edt,dc=org

dn: ou=maquines,dc=edt,dc=org

dn: ou=clients,dc=edt,dc=org

dn: ou=productes,dc=edt,dc=org

dn: ou=usuaris,dc=edt,dc=org


[gustavo@localhost ldapserver19:tls]$ openssl s_client -connect ldap.edt.org:636
CONNECTED(00000003)
depth=1 C = ca, ST = Barcelona, L = Barcelona, O = Veritat Absoluta, OU = Certificats covid19, CN = Veritat_Absoluta, emailAddress = admin@edt.org
verify error:num=19:self signed certificate in certificate chain
verify return:1
depth=1 C = ca, ST = Barcelona, L = Barcelona, O = Veritat Absoluta, OU = Certificats covid19, CN = Veritat_Absoluta, emailAddress = admin@edt.org
verify return:1
depth=0 C = ca, ST = Barcelona, L = Barcelona, O = M11-SAD, OU = Informatica, CN = ldap.edt.org, emailAddress = admin@edt.org
verify return:1
---
Certificate chain
 0 s:C = ca, ST = Barcelona, L = Barcelona, O = M11-SAD, OU = Informatica, CN = ldap.edt.org, emailAddress = admin@edt.org
   i:C = ca, ST = Barcelona, L = Barcelona, O = Veritat Absoluta, OU = Certificats covid19, CN = Veritat_Absoluta, emailAddress = admin@edt.org
 1 s:C = ca, ST = Barcelona, L = Barcelona, O = Veritat Absoluta, OU = Certificats covid19, CN = Veritat_Absoluta, emailAddress = admin@edt.org
   i:C = ca, ST = Barcelona, L = Barcelona, O = Veritat Absoluta, OU = Certificats covid19, CN = Veritat_Absoluta, emailAddress = admin@edt.org
---
Server certificate
-----BEGIN CERTIFICATE-----
MIID+DCCAuCgAwIBAgIUezeP83c2oAgshtaU3F8aPT119sowDQYJKoZIhvcNAQEF
BQAwgacxCzAJBgNVBAYTAlNQMRIwEAYDVQQIDAlCYXJjZWxvbmExEjAQBgNVBAcM
CUJhcmNlbG9uYTEZMBcGA1UECgwQVmVyaXRhdCBBYnNvbHV0YTEcMBoGA1UECwwT
Q2VydGlmaWNhdHMgY292aWQxOTEZMBcGA1UEAwwQVmVyaXRhdF9BYnNvbHV0YTEc
MBoGCSqGSIb3DQEJARYNYWRtaW5AZWR0Lm9yZzAeFw0yMDA0MDYxODMzMTBaFw0y
MTA0MDYxODMzMTBaMIGSMQswCQYDVQQGEwJjYTESMBAGA1UECAwJQmFyY2Vsb25h
MRIwEAYDVQQHDAlCYXJjZWxvbmExEDAOBgNVBAoMB00xMS1TQUQxFDASBgNVBAsM
C0luZm9ybWF0aWNhMRUwEwYDVQQDDAxsZGFwLmVkdC5vcmcxHDAaBgkqhkiG9w0B
CQEWDWFkbWluQGVkdC5vcmcwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIB
AQDkf3CmJcnN3gu4xrorUC9QSfEJaQ9lhRNRxI40f4Vd+eJIvBvlyCPBCDb7K/aU
bXw6cOJW0iidLDi2ZFAUvqPHZxhwfk3nGnukf9bzLcPCOgHY+e53l1NasiEyHcI2
1V8XR+2PFEEP8tn7xqTHCnVc+F3evcqwiNxuSfNFICtsdhnRRrYSQWDJ9HWFIqWI
IgtIdEXIm1GBH2bF53QgHr46bj/643SWfu0+qRbtEIZZmsuXdQdmxIwnULfpOdFG
X2BsYDhfBS6tLa02DsSL9bP7NMi3ru42uYIk+11EvLz7O1fe+GCq2jo9ciFBQ9ga
QR3m0thoweJjnCEnihr0aB8XAgMBAAGjLzAtMAwGA1UdEwEB/wQCMAAwHQYDVR0l
BBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMEMA0GCSqGSIb3DQEBBQUAA4IBAQBPEkSA
ENahTZ/R+0ldY1OChXXRoj9NtWtmJq59mPfndIk43iZdFdn8ZEUVTn0+so/zHOoM
mSKMSbdxCC0Gn7ykwGfrCEyc7AevDx1JsM+5h7AMomHshHwglpTqzWIiBvISOfiF
CJZBGq/8E/5l017UF6HzbZQwBatFNpwLJGizUFBC7m1VDQsKFps57qVLv1eW5gt9
D11jDOy+kiL/o7eW8+6R88Mk3Fb9p3+YTWTp2Hv5ONI+vmrePQK7FsFv+vc7wFUP
EsHcFKA5EE3R2zdctZqCjU9SD9RcxWmGNTbl8Jpwba+h2non7Yg1MkJtsOAx2A6h
jp1A2qwpcerwTjTx
-----END CERTIFICATE-----
subject=C = ca, ST = Barcelona, L = Barcelona, O = M11-SAD, OU = Informatica, CN = ldap.edt.org, emailAddress = admin@edt.org

issuer=C = ca, ST = Barcelona, L = Barcelona, O = Veritat Absoluta, OU = Certificats covid19, CN = Veritat_Absoluta, emailAddress = admin@edt.org

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 2557 bytes and written 417 bytes
Verification error: self signed certificate in certificate chain
---
New, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES128-GCM-SHA256
    Session-ID: 0013561E13971399E11B22E514A97CD7C2DE57DA504D87087867A38F6497B853
    Session-ID-ctx:
    Master-Key: A4D4ECD0ED011D617520D0E2E944C2FE2D8D40A650EF96D60E91BDD717FDCD86F7D2216BF929A27EEF9D0485B835806C
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1586260530
    Timeout   : 7200 (sec)
    Verify return code: 19 (self signed certificate in certificate chain)
    Extended master secret: no
---
```
