# ./scriptyk

## Overview

`./scriptyk` is a simple command line tool used to build a PKI CA, 
powered by a Yubikey (or other PKCS#11 tokens) for private key
management and git for history.

Every operation that change the state of the CA (such as emitting or
revoking certificates) creates a git commit ; you can for example `git
revert` a revoke operation.

## Dependencies

* Python >= 3.2
* OpenSSL
* [OpenSSL PKCS#11 engine from OpenSC](https://github.com/OpenSC/libp11/)
* `p11tool` from GnuTLS
* A PKCS#11 provider; for a Yubikey, you can use OpenSC

## PIN

The default PIN used to access the Yubikey is 123456. You can change
it with the `PIN` environment variable ; the special value `ask` will
issue a prompt if the PIN is needed.

## Initialization

`./scriptyk init-ca` will initialize a new CA in the current directory
(you probably want an empty one). It will create a new git repository,
files needed by the CA, and do a first commit.

`./scriptyk init-key` will initialize the private key on the Yubikey. You
have three possibilities :

* On-device key generation: `./scriptyk init-key -g`. This
is the recommended method, unless your device is vulnerable to
[this security advisory](https://www.yubico.com/support/security-advisories/ysa-2017-01/).
* Using an existing key, in PEM format: `./scriptyk init-key -f key.pem`.

You should also give a name to your CA, the default being "Yubikey
CA". You can do so with the `-s` option. Also, you can choose the slot of yubikey where you want to generate the key with the `--slot`. The algorithm (RSA2048 or ECCP256) can also be choosed with the `--typealgorithm` option:

* `./scriptyk init-key -g -s "/O=Example Inc/CN=Certificate Authority" --slot "9c" --typealgorithm="RSA2048"`

Now that your CA key is on the device, you can access to your CA
certificate:

* `./scriptyk ca-cert`

You can choose different slots with the `-u` option identified by key.

* `./scriptyk ca-cert -u 'pkcs11:manufacturer=piv_II;id=%02'`


## Using the PKI

### Issuing and revoking certificates

Generate a certificate with a subject name signed by the CA, for example:

* `./scriptyk client-cert -s "/O=Example Inc/CN=John Doe"`

You also can specify the name for the private key and certificate files with "-n":

* `./scriptyk client-cert -s "/O=Example Inc/CN=John Doe -n "john"`

You can use other algorithms for your client certificates, for example
`openssl genrsa 4096` for a different key size or `openssl ecparam
-genkey -name secp256k1` for an ECC key.

The `-e` option can be used to export the certificate + key in the PKCS#12
format for importing it in the browser.

The certificates are stored in the `certs/` directory, and are listed
in the `index.txt` file. The first field describe the certificate status
(`V` for valid), the second the issuing date, the third the certificate
serial and the last one the certificate name.

To revoke certificates, you must provide their serial:

* `./scriptyk client-revoke 01 03`

Generate a CRL containing the revoked certificates:

* `./scriptyk crl`

### For manually creation of a communication between Charles and Denis, being Charles the owner of the yubikey and self-sign the certificate

Generate private key of Charles

* `openssl ecparam -name prime256v1 -genkey -noout -out charles_priv_key.pem`

Importing key on yubikey

* `yubico-piv-tool -a import-key -s 9d -k -i charles_priv_key.pem`

Generate certificate from private key

* `openssl req -x509 -days 3650 -sha256 -subj "/O=DCC/CN=Charles" -key charles_priv_key.pem > CharlesCert.pem`

Import certificate to the slot 9d

* `yubico-piv-tool -s9d -aimport-certificate -i CharlesCert.pem`

#### Example of generation of a shared secret between two parties

Derive a shared symmetric key between Charles and Dennis (With dennis public key)

* ` openssl pkeyutl -derive -engine pkcs11 -keyform engine -inkey  'pkcs11:manufacturer=piv_II;id=%03' -peerkey denis_pub_key.pem -out charles_shared_secret.bin`

The other peer has to do the same

Then, we can test the functionality with:

* `echo 'I love you Bob' > plain.txt`
* `openssl enc -aes256 -base64 -k $(base64 charles_shared_secret.bin) -e -in plain.txt -out cipher.txt`
* `openssl enc -aes256 -base64 -k $(base64 denis_shared_secret.bin) -d -in cipher.txt -out plain_again.txt`


## Known bugs


## Limitations

TBD 

## Using another PKCS#11 token

Althought ./scriptyk is primarily intended to be used with a Yubikey,
it can be used with others PKCS#11 tokens:

* If you must use another provider than the default (`opensc-pkcs11.so`),
either use the `PKCS11_MODULE` environment variable or the `pkcs11-module`
command-line option.
* Specify the slot URL with `PKCS11_URL` environment
variable or `pkcs11-url` option. You can get the list with `p11tool
--provider=$PKCS11_MODULE --list-all` ; you can use only parts of the URL
(for example, only `manufacturer` and `id`), you must remove the `type=`
part of the URL, and you must keep the `id=` part of the URl.
* `init-key` won't work, you will have to manually initialize the token.

### Example with SoftHSM

Initialization (you may want to change the token name, `SoftHSM`, the
CA subject, `Test CA`, and the PINs):

```
export SOFTHSM2_CONF=/path/to/softhsm2.conf
TOKEN="SoftHSM"
CA="/CN=Test CA"
PIN="123456"
SO_PIN="$PIN"

softhsm2-util --init-token --slot 0 --label "$TOKEN" --pin "$PIN" --so-pin "$SO_PIN"
KEY=$(openssl genrsa 2048)
pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so --id 02 --type privkey --label "SIGN privkey" --pin "$PIN" --write <(openssl rsa -in <(echo "$KEY") -outform der)
pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so --id 02 --type pubkey --label "SIGN pubkey" --pin "$PIN" --write <(openssl rsa -in <(echo "$KEY") -outform der -pubout)
pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so --id 02 --type cert --label "Certificate for Digital Signature" --pin "$PIN" --write <(openssl req -x509 -key <(echo "$KEY") -sha256 -days 3560 -subj "$CA" -outform der)
```

Example usage:

```
export SOFTHSM2_CONF=/path/to/softhsm2.conf

./scriptyk -m /usr/lib/softhsm/libsofthsm2.so -u "pkcs11:token=SoftHSM;id=%02" ca-cert
PKCS11_MODULE="/usr/lib/softhsm/libsofthsm2.so" PKCS11_URL="pkcs11:token=SoftHSM;id=%02" ./scriptyk client-cert
```


