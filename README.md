# OQS-openssl-in-PHP
This repository contains notes on multiple ways of how to install post-quantum version of OpenSSL.

## PHP OpenSSL extension remarks
My main goal was to use `openssl_XXX` functions in the PHP, but as it turns out DSA, DH, RSA, and EC algorithms are [hardcoded in the OpenSSL PHP extension](https://github.com/open-quantum-safe/openssl/issues/433). Therefore, only few functions are usable (those, that do not require algorithm identifier, like `openssl_verify` and `openssl_sign`, but not `openssl_pkey_new`).

## OQS-OpenSSLv1.1 fork - integrated functions for PHP - Not Recommended
- follow instructions at [open-quantum-safe/openssl](https://github.com/open-quantum-safe/openssl#building)
    - Step 2: `./Configure no-shared linux-aarch64 --lm` for Apple Silicon
    - Step X: `make install` in the end
- then we can build PHP again with modified openssl
    - Before PHP configure: `apt install apache2-dev libxml2-dev sqlite3 libsqlite3-dev zlib1g-dev libcurl4-openssl-dev libonig-dev libreadline-dev libsodium-dev libargon2-dev`
    - remove keccak mentions from PHP configure file (it creates duplicate errors during the build, it is already defined in OpenSSL fork)
        - <img width="1213" alt="PQC-Web-eID-image-20230314100556020" src="https://github.com/Muzosh/OQS-openssl-installation-notes/assets/30979983/bf2b16b6-3594-4e4d-ae2c-86d06ea0ac1e">
    - remove sha3 mentions from ext/hash/hash.c:
        - ![PQC-Web-eID-image-20230314100643119](https://github.com/Muzosh/OQS-openssl-installation-notes/assets/30979983/470839fd-1386-4a9f-bab1-28c9b844cbd4)
    - PHP configure (basically same arguments you would get by running `php-config --configure-options`, except change the `with-openssl`:
        - `./configure --build=aarch64-linux-gnu --with-config-file-path=/usr/local/etc/php --with-config-file-scan-dir=/usr/local/etc/php/conf.d --enable-option-checking=fatal --with-mhash --with-pic --enable-ftp --enable-mbstring --enable-mysqlnd --with-password-argon2 --with-sodium=shared --with-pdo-sqlite=/usr --with-sqlite3=/usr --with-curl --with-iconv --with-openssl=/usr/local/ssl --with-readline --with-zlib --disable-phpdbg --with-pear --with-libdir=lib/aarch64-linux-gnu --disable-cgi --with-apxs2 build_alias=aarch64-linux-gnu`
    - add `-lpthread` to make file here:
        - ![PQC-Web-eID-image-20230314100747753](https://github.com/Muzosh/OQS-openssl-installation-notes/assets/30979983/ae063c04-6b22-4cb8-b9a4-62c4bf919578)
    - `make` the PHP
    - `make install` the PHP
 
## OQS-OpenSSLv1.1 fork - execution from PHP - Not Recommended
- this works: `shell_exec("/usr/local/bin/openssl dgst -sha256 -verify /tmp/dil5_CA_public.pem -signature /tmp/dilhash.sig /tmp/example.txt");`
- most probably would work with OpenSSL3 as well
```php
$result = exec(
    "/usr/local/bin/openssl dgst -sha256 -verify /tmp/dil5_CA_public.pem -signature /tmp/dilhash.sig /tmp/example.txt",
    $output,
    $return
);
```

## OQS-OpenSSLv3 provider/extension - integrated functions for PHP - Recommended

- `git clone https://github.com/open-quantum-safe/oqs-provider.git`
- `./scripts/fullbuild.sh` to obtain `_build/lib/oqsprovider.so`
    - fullbuild.sh is probably best way (without openssl installation if you already have one)
- place `oqsprovider.so` into `OPENSSL_DIR/ossl_modules`
- insert this snippet into used `openssl.cnf`:

```
[openssl_init]
providers = provider_sect

# List of providers to load
[provider_sect]
default = default_sect
oqsprovider = oqsprovider_sect

[default_sect]
activate = 1
[oqsprovider_sect]
activate = 1
```

- build PHP:
```bash
./configure --build=aarch64-linux-gnu --with-config-file-path=/usr/local/etc/php --with-config-file-scan-dir=/usr/local/etc/php/conf.d --enable-option-checking=fatal --with-mhash --with-pic --enable-ftp --enable-mbstring --enable-mysqlnd --with-password-argon2 --with-sodium=shared --with-pdo-sqlite=/usr --with-sqlite3=/usr --with-curl --with-iconv --with-openssl=/usr/local/ssl --with-readline --with-zlib --disable-phpdbg --with-pear --with-libdir=lib/aarch64-linux-gnu --disable-cgi --with-apxs2 build_alias=aarch64-linux-gnu

make
make install
```

### Testing: openssl_verify and openssl_sign

```bash TI:"generate keys and sigatures in command line"
echo "test data" > /tmp/example.txt

openssl genpkey -algorithm dilithium5 -out /tmp/dil5.key
openssl genrsa -out rsa.key 2048

openssl pkey -pubout -in /tmp/dil5.key -out /tmp/dil5.pem
openssl rsa -pubout -in /tmp/rsa.key -out /tmp/rsa.pem

openssl dgst -sign /tmp/rsa.key -out /tmp/rsa.sig -sha256 /tmp/example.txt
openssl dgst -sign /tmp/dil5.key -out /tmp/dil5.sig -sha256 /tmp/example.txt

openssl dgst -sha256 -verify /tmp/rsa.pem -signature /tmp/rsa.sig /tmp/example.txt
openssl dgst -sha256 -verify /tmp/dil5.pem -signature /tmp/dil5.sig /tmp/example.txt
```

```php TI:"Test snippet" "FOLD"
<?php
$data = file_get_contents('/tmp/example.txt');

// RSA
$rsa_verification_public_key = file_get_contents('/tmp/rsa.pem');
$rsa_verification_signature = file_get_contents('/tmp/rsa.sig');

$rsa_verification_results = openssl_verify($data, $rsa_verification_signature, $rsa_verification_public_key, OPENSSL_ALGO_SHA256);

// DIL5
$dil5_verification_public_key = file_get_contents('/tmp/dil5.pem');
$dil5_verification_signature = file_get_contents('/tmp/dil5.sig');

$dil5_verification_results = openssl_verify($data, $dil5_verification_signature, $dil5_verification_public_key, OPENSSL_ALGO_SHA256);

// RSA create signature
$rsa_signing_private_key = file_get_contents('/tmp/rsa.key');
$rsa_signing_signature = null;
$rsa_signing_result = openssl_sign($data, $rsa_signing_signature, $rsa_signing_private_key, OPENSSL_ALGO_SHA256);
$rsa_signing_verify_result = openssl_verify($data, $rsa_signing_signature, $rsa_verification_public_key, OPENSSL_ALGO_SHA256);

// DIL5 create signature
$dil5_signing_private_key = file_get_contents('/tmp/dil5.key');
$dil5_signing_signature = null;
$dil5_signing_result = openssl_sign($data, $dil5_signing_signature, $dil5_signing_private_key, OPENSSL_ALGO_SHA256);
$dil5_signing_verify_result = openssl_verify($data, $dil5_signing_signature, $dil5_verification_public_key, OPENSSL_ALGO_SHA256);

echo "Done";
```
