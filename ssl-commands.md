h1. SSL Commands
Most of this is gleaned from:
  * https://jamielinux.com/docs/openssl-certificate-authority/sign-server-and-client-certificates.html

h2. Gen a private key

        openssl genrsa -out intermediate/private/openshift.example.com.key.pem 2048

h2. Create a CSR

        openssl req -config intermediate/openssl.cnf                        \
                    -new -sha256                                            \
                    -key intermediate/private/openshift.example.com.key.pem \
                    -out intermediate/csr/openshift.example.com.csr.pem

h2. Sign a CSR

        openssl ca -config intermediate/openssl.cnf                       \
                   -extensions server_cert -days 750 -notext -md sha256   \
                   -in intermediate/csr/openshift.example.com.csr.pem     \
                   -out intermediate/certs/openshift.example.com.cert.pem


h2. Create a CSR that contains a SAN
    * This generates a key and CSR at the same time

        openssl req -new -nodes \
                  -config intermediate/openshift.san.csr.cnf                  \
                  -keyout intermediate/private/openshift.example.com.key.pem  \
                  -out intermediate/csr/openshift.example.com.csr.pem

h2. Sign a CSR that contains a SAN

        openssl ca -config intermediate/openssl.san.cnf                                    \
                   -extensions server_cert -days 750 -notext -md sha256 -extensions v3_req \
                   -in intermediate/csr/openshift.example.com.csr.pem                      \
                   -out intermediate/certs/openshift.example.com.cert.pem

h2. Examine a CERT

        openssl x509 -noout -text                                          \
                   -in intermediate/certs/openshift.example.com.cert.pem


h2. Validate a cert against a CA

        openssl verify -CAfile intermediate/certs/ca-chain.cert.pem        \
              intermediate/certs/www.example.com.cert.pem



h2. Contents of intermediate/openssl.san.cnf

        # OpenSSL intermediate CA configuration file.
        # Copy to `/root/ca/intermediate/openssl.cnf`.
        
        [ ca ]
        # `man ca`
        default_ca = CA_default
        
        [ CA_default ]
        # Directory and file locations.
        dir               = /home/rob.brucks/root/ca/intermediate
        certs             = $dir/certs
        crl_dir           = $dir/crl
        new_certs_dir     = $dir/newcerts
        database          = $dir/index.txt
        serial            = $dir/serial
        RANDFILE          = $dir/private/.rand
        
        # The root key and root certificate.
        private_key       = $dir/private/intermediate.key.pem
        certificate       = $dir/certs/intermediate.cert.pem
        
        # For certificate revocation lists.
        crlnumber         = $dir/crlnumber
        crl               = $dir/crl/intermediate.crl.pem
        crl_extensions    = crl_ext
        default_crl_days  = 30
        
        # SHA-1 is deprecated, so use SHA-2 instead.
        default_md        = sha256
        
        name_opt          = ca_default
        cert_opt          = ca_default
        default_days      = 375
        preserve          = no
        policy            = policy_loose
        
        [ policy_strict ]
        # The root CA should only sign intermediate certificates that match.
        # See the POLICY FORMAT section of `man ca`.
        countryName             = match
        stateOrProvinceName     = match
        organizationName        = match
        organizationalUnitName  = optional
        commonName              = supplied
        emailAddress            = optional
        
        [ policy_loose ]
        # Allow the intermediate CA to sign a more diverse range of certificates.
        # See the POLICY FORMAT section of the `ca` man page.
        countryName             = optional
        stateOrProvinceName     = optional
        localityName            = optional
        organizationName        = optional
        organizationalUnitName  = optional
        commonName              = optional
        emailAddress            = optional
        
        [ req ]
        # Options for the `req` tool (`man req`).
        default_bits        = 2048
        distinguished_name  = req_distinguished_name
        string_mask         = utf8only
        
        # SHA-1 is deprecated, so use SHA-2 instead.
        default_md          = sha256
        
        # Extension to add when the -x509 option is used.
        x509_extensions     = v3_ca
        
        [ req_distinguished_name ]
        # See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
        countryName                     = Country Name (2 letter code)
        stateOrProvinceName             = State or Province Name
        localityName                    = Locality Name
        0.organizationName              = Organization Name
        organizationalUnitName          = Organizational Unit Name
        commonName                      = Common Name
        emailAddress                    = Email Address
        
        # Optionally, specify some defaults.
        countryName_default             = US
        stateOrProvinceName_default     = Texas
        localityName_default            = Seguin
        0.organizationName_default      = 4q.com
        organizationalUnitName_default  =
        emailAddress_default            =
        
        [ v3_ca ]
        # Extensions for a typical CA (`man x509v3_config`).
        subjectKeyIdentifier = hash
        authorityKeyIdentifier = keyid:always,issuer
        basicConstraints = critical, CA:true
        keyUsage = critical, digitalSignature, cRLSign, keyCertSign
        
        [ v3_req ]
        # Extensions for a typical CA (`man x509v3_config`).
        subjectKeyIdentifier = hash
        authorityKeyIdentifier = keyid:always,issuer
        basicConstraints = critical, CA:true
        keyUsage = critical, digitalSignature, cRLSign, keyCertSign
        subjectAltName = @alt_names
        
        [ v3_intermediate_ca ]
        # Extensions for a typical intermediate CA (`man x509v3_config`).
        subjectKeyIdentifier = hash
        authorityKeyIdentifier = keyid:always,issuer
        basicConstraints = critical, CA:true, pathlen:0
        keyUsage = critical, digitalSignature, cRLSign, keyCertSign
        
        [ usr_cert ]
        # Extensions for client certificates (`man x509v3_config`).
        basicConstraints = CA:FALSE
        nsCertType = client, email
        nsComment = "OpenSSL Generated Client Certificate"
        subjectKeyIdentifier = hash
        authorityKeyIdentifier = keyid,issuer
        keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
        extendedKeyUsage = clientAuth, emailProtection
        
        [ server_cert ]
        # Extensions for server certificates (`man x509v3_config`).
        basicConstraints = CA:FALSE
        nsCertType = server
        nsComment = "OpenSSL Generated Server Certificate"
        subjectKeyIdentifier = hash
        authorityKeyIdentifier = keyid,issuer:always
        keyUsage = critical, digitalSignature, keyEncipherment
        extendedKeyUsage = serverAuth
        
        [ crl_ext ]
        # Extension for CRLs (`man x509v3_config`).
        authorityKeyIdentifier=keyid:always
        
        [ ocsp ]
        # Extension for OCSP signing certificates (`man ocsp`).
        basicConstraints = CA:FALSE
        subjectKeyIdentifier = hash
        authorityKeyIdentifier = keyid,issuer
        keyUsage = critical, digitalSignature
        extendedKeyUsage = critical, OCSPSigning
        
        [alt_names]
        DNS.1 = master1.example.com
        DNS.2 = master2.example.com
        DNS.3 = master3.example.com
        DNS.4 = openshift.example.com



h2. Contents of intermediate/openshift.san.csr.cnf

        [ req ]
        # Options for the `req` tool (`man req`).
        default_bits        = 2048
        distinguished_name  = req_distinguished_name
        string_mask         = utf8only
        req_extensions      = v3_req
        
        # SHA-1 is deprecated, so use SHA-2 instead.
        default_md          = sha256
        
        [ req_distinguished_name ]
        # See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
        countryName                     = Country Name (2 letter code)
        stateOrProvinceName             = State or Province Name
        localityName                    = Locality Name
        0.organizationName              = Organization Name
        organizationalUnitName          = Organizational Unit Name
        commonName                      = Common Name
        emailAddress                    = Email Address
        
        # Optionally, specify some defaults.
        countryName_default             = US
        stateOrProvinceName_default     = Texas
        localityName_default            = Seguin
        0.organizationName_default      = 4q.com
        organizationalUnitName_default  =
        
        [ v3_req ]
        # Extensions for a typical CA (`man x509v3_config`).
        basicConstraints = CA:FALSE
        keyUsage = nonRepudiation, digitalSignature, keyEncipherment
        subjectAltName = @alt_names
        
        [alt_names]
        DNS.1 = master1.example.com
        DNS.2 = master2.example.com
        DNS.3 = master3.example.com
        DNS.4 = openshift.example.com

