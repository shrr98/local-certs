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
0.organizationName = Rezky
commonName = Shin Rezky
emailAddress = shinrezky@example.com
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

## Membuat self-signed SSL certificate
Misal untuk domain ```example.local```.

1. Buat directory ```example.local``` di dalam folder ```local-certs```.
```

cd local-certs
mkdir example.local

```

2. Buat file ```req.conf``` di dalam folder ```example.local``` dan masukkan detil CSR.

```

[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no
[ req_distinguished_name ]
C = ID
ST = Jawa Timur
L = Surabaya
O = Rezky
OU = Shin Rezky
CN = example.local
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[ alt_names ]
DNS.1 = example.local
DNS.2 = www.example.local

```

Keterangan:
a. Common Name: FQDN (fully-qualified domain name) misalnya dev.local atau
example.org
b. Organization: Nama lengkap perusahaan
c. Organization Unit (OU): Nama departemen atau divisi misal IT Division.
d. City or Locality: Nama kota tempat perusahaan.
e. State or province: nama provinsi
f. Country: nama negara menggunakan 2-karakter kode negara. 

3. Buat private key dan CSR.

```

openssl req -new -out example.local.csr -newkey rsa:2048 -nodes -sha256 -keyout
dev.local.key -config req.conf

```

4. Memeriksa CSR sesuai dengan yang request.

```

openssl req -text -noout -verify -in example.local.csr

```

Pastikan Entri Subject Alternative Names (SAN) sudah masuk.

```

X509v3 Subject Alternative Name:
    DNS:dev.local, DNS:www.example.local

```

## Melakukan Signing menggunakan Root CA
1. Pindah ke root-ca
```
cd local-certs/root-ca
```

2. Edit ```conf/openssl.cnf``` untuk menambahkan SAN. Langkah ini harus dilakukan
untuk setiap signing dengan domain/CN yang berbeda.


<pre>

[ usr_cert ]
basicConstraints=CA:FALSE
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer:always
nsCaRevocationUrl = https://ca.local/ca-crl.pem
<b>subjectAltName = @alt_names
[alt_names]
DNS.1 = example.local
DNS.2 = www.example.local
</b>
</pre>


3. Lakukan signing

```
openssl ca -batch -config conf/openssl.cnf -in
../example.local/example.local.csr -out ../example.local/example.local.crt
```

4. Periksa SSL certificate hasil signing
```
openssl x509 -in example.local.crt -text -noout
```

Pastikan Entri SAN sudah masuk.

```
X509v3 Subject Alternative Name:
    DNS:dev.local, DNS:www.dev.local
```

## Memasang self-CA-signed SSL ke aplikasi web (Apache)

Edit file ```/etc/apache2/sites-available/example.local.conf```

```
sudo nano /etc/apache2/sites-available/example.local.conf
```

Masukkan:

```

<VirtualHost *:80>
        DocumentRoot "/var/www/html/phalcon-example/src/public"
        ServerName example.local
        ErrorLog ${APACHE_LOG_DIR}/example.local-error_log
        CustomLog ${APACHE_LOG_DIR}/example.local-access_log common

        <Directory "/var/www/html/phalcon-example/src/public">
                AllowOverride All
                Require all granted
        </Directory>

       Redirect permanent / https://example.local
</VirtualHost>

<VirtualHost *:443>
       DocumentRoot "/var/www/html/phalcon-example/src/public"
       ServerName example.local

       <Directory "/var/www/html/phalcon-example/src/public">
               AllowOverride all
               Require all granted
       </Directory>

       SSLEngine on
       SSLCertificateFile "/var/www/certs/example.local/example.local.crt"
       SSLCertificateKeyFile "/var/www/certs/example.local/example.local.key"
</VirtualHost>

```


## Menambahkan Root CA ke Browser ( Firefox )

1. Buka Mozilla Firefox
2. Buka ```Options``` -> ```Privacy & Security```. Pada bagian ```Certificates``` klik ```View Certificates...```.
3. Pada dialog Certificate Manager, pilih tab ```Authorities```. Import root-CA ( /mnt/d/local-certs/root-ca/public/root.der ). Klik OK.

![Certificate Manager](/cert.jpg)


## Testing
Buka ```example.local``` melalui Mozilla Firefox
![Secure](/secure.jpg)
![Success](/success.jpg)