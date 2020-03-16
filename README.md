# Membuat Sertifikat SSL untuk Lingkungan Pengembangan Aplikasi Berbasis Web

## Membuat self-signed Root Certificate Authority ( Root CA )

1. Buat directory dengan nama local-certs pada /mnt/d/

```

cd /mnt/d/
mkdir local-certs

```

2. Di dalam folder local-certs buat direktori dengan nama root-ca dan di dalam
direktori root-ca buat direktori dengan nama conf, private, dan public.

```

cd local-certs
mkdir -p root-ca/{conf,private,public}
cd root-ca

```

3. Buat file ```conf/openssl.cnf```. Ganti nama developer/perusahaan dengan
nama Anda atau nama perusahaan Anda.

```

[ req ]
default_bits = 4096
default_keyfile = ./private/root.pem
default_md = sha256
prompt = no
distinguished_name = root_ca_distinguished_name
x509_extensions = v3_ca
[ root_ca_distinguished_name ]
countryName = ID
stateOrProvinceName = Jawa Timur
localityName = Surabaya
0.organizationName = Nama Developer/Perusahaan
commonName = Nama Developer/Perusahaan Root CA
emailAddress = email@example.com
[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer:always
basicConstraints = CA:true

```

4. Buat private dan public key

```

openssl req -nodes -config conf/openssl.cnf -days 1825
 -x509 -newkey rsa:4096 -out public/root.pem -outform PEM

```

5. Membuat public key dalam format DER (jika diperlukan).

```

openssl x509 -in public/root.pem -outform DER -out public/root.der

```

6. Menambahkan konfigurasi di conf/openssl.cnf untuk melakukan signing.

```

[ ca ]
default_ca = CA_default
[ CA_default ]
dir =.
new_certs_dir = ./signed-keys/
database = ./conf/index
certificate = ./public/root.pem
serial = ./conf/serial
private_key = ./private/root.pem
x509_extensions = usr_cert
name_opt = ca_default
cert_opt = ca_default
default_crl_days = 30
default_days = 365
default_md = sha256
preserve = no
policy = policy_match
[ policy_match ]
countryName = match
stateOrProvinceName = optional
organizationName = optional
organizationalUnitName = optional
commonName = supplied
emailAddress = optional
[ usr_cert ]
basicConstraints=CA:FALSE
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer:always
nsCaRevocationUrl = https://ca.local/ca-crl.pem

```

7. Buat direktori untuk menyimpan daftar key yang telah di-signed (pwd : root-ca)

```

mkdir signed-keys
echo "01" > conf/serial
touch conf/index

```

