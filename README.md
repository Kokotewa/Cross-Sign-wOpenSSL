# How to Create a Cross-Signed Certificate Authority with OpenSSL
To the best of my knowledge, this procedure isn't documented anywhere else and is relatively unintuitive.  After discovering the procedure for myself, I've opted to document it here with the hope of saving others a significant effort.  If this is helpful for you, please consider documenting another unknown topic of similar complexity to pay it forward.

## Background (Example Usecase)
You have an existing Certificate Authority that is trusted by field devices (`CA_1`).  `CA_1` has been in use for some time, e.g. has multiple intermediates which signed other certificates and so on, and is well trusted.  However, as `CA_1` is relatively old, it is no longer supported by modern openssl.

To support new devices, with modern openssl, it is necessary to create a new Certificate Authority (`CA_2`).  However, it is desired that existing devices, which trust `CA_1`, also trust `CA_2`.  This trust must be achieved without importing `CA_2` into these devices (i.e. certificate must be inherently trusted).

Alternative Usecases:
* It is desired to 'rotate' trusted CAs in a rolling fashion (e.g. 1&2 -> 2&3 -> ...)
* It is desired to implement an asymmetric trust model

## Solution
This can be achieved by cross-signing `CA_2` with `CA_1`.  Existing devices, which trust `CA_1`, trust `CA_2` as it is signed by `CA_1`.  New devices, which trust `CA_2`, trust `CA_2`.  See the following reference procedure for an example on how to cross sign a new, or existing, certificate with an existing certificate.

Within a production environment, e.g. the example usecase, `CA_1` and `CA_2` already exist.  However, to facilitate example readability, these CAs are newly generated.  Several flags, which should be considered for a production environment (e.g. certificate lifespan), are omitted for clarity.
```
## Generate CA_1
mkdir CA_1
openssl genrsa -out CA_1/ca.key 2048
openssl req -new -x509 -key CA_1/ca.key -out CA_1/ca.crt

## Generate CA_2
mkdir CA_2
openssl genrsa -out CA_2/ca.key 2048
openssl req -new -x509 -key CA_2/ca.key -out CA_2/ca.crt
```
To cross sign `CA_2` with `CA_1`, we must first generate a certificate signing request (CSR) with `CA_2`.  After the CSR has been generated, `CA_1` can sign the `CA_2` CSR and generate a new cross-signed certificate (`xca.crt`).  As before, selected arguments are for clarity and additional arguments may be desired for a production environment (e.g. lifespan, non-random serial number).
```
## Cross Sign CA_2 with CA_1
# Generate the CSR from CA_2
openssl x509 -x509toreq -in CA_2/ca.crt -signkey CA_2/ca.key -out CA_2/ca.csr
# Sign CA_2 with CA_1
openssl ca -cert CA_1/ca.crt -keyfile CA_1/ca.key -in CA_2/ca.csr -out CA_2/xca.crt -extensions v3_ca -rand_serial
```

Once the cross-signed certificate has been generated, it can be used as normal.  The following commands illustrate generating a new leaf certificate from the cross-signed CA, building a certificate chain, and related nginx certificate/key configuration.
```
## Generate Web Certificate and Sign with xCA_2
# Generate the Web CSR
mkdir web
openssl genrsa -out web/web.key 2048
openssl req -new -key web/web.key -out web/web.csr
# Sign the CSR with CA_2
openssl x509 -req -in web/web.csr -CA CA_2/xca.crt -CAkey CA_2/ca.key -CAcreateserial -out web/web.crt
# Build the Certificate Chain
cat web/web.crt CA_2/xca.crt >> web/chain.crt

# nginx conf
cat /etc/nginx/conf.d/ssl.conf 
ssl_certificate             web/chain.crt;
ssl_certificate_key         web/web.key;
```
