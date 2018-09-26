# TLS

A simple guide into how to use it. We will be leveraging `openssl` and `cfssl`.
Unfortunately I did not lean completely on `cfssl` as it makes things a bit too
magically for a detailed explanation about whats going on.

## Reqs

Install the following:

* cfssl
* cfssljson
* python3

## Certificate Authority

Validate that everything in ca.json is valid:

```
view ca.json
```

Create the private public key for the `ca`:

```sh 
$ cd ca
$ cfssl gencert -initca ca.json | cfssljson -bare ca
$ rm ca.csr
$ mv ca-key.pem ca.priv
$ mv ca.pem ca.pub
```
> NOTE: Usually this is more than one step with openssl but cfssl
> streamlines the process

## Server Cert

Go into the following dir:

```sh
cd ../cert1
```

Generate a private key that 2048 bits in size:

```sh
openssl genrsa -out coolapi.priv 2048
```

Validate config in config file for csr generation:

```sh
vim coolapi.conf
```

Generate the `csr`:

```sh
openssl req -new -config coolapi.conf -key coolapi.priv -out coolapi.csr
```

Double check that the `csr` contains the expected values:

```sh
openssl req -in coolapi.csr -noout -text -nameopt sep_multiline
```

Provide the generated coolapi.csr to the CA to issue our cert!

## Generate crt

Go to `ca` dir:

```sh
cd ../ca
mkdir sign-cert1
cd sign-cert1
```

Copy in `csr` that was generated:

```sh
cp ../../cert1/coolapi.csr .
```

Generate the `crt` for `coolapi.nebulaworks.com`:

```sh
cfssl sign -ca=../ca.pub -ca-key=../ca.priv ./coolapi.csr | cfssljson -bare coolapi
mv coolapi.pem coolapi.crt
```

Verify that the cert indeed came from the `ca`:

```sh
$ openssl verify -verbose -CAfile ../ca.pub coolapi.crt
coolapi.crt
```

Copy `coolapi.crt` back to cert1 folder along with the `ca.pub` public key

### Enabling the service

Go to the `cert1` dir:

```sh
cd cert1
```

Validate that the cert came from your private key:

```sh
openssl x509 -noout -modulus -in coolapi.crt | openssl md5
openssl rsa -noout -modulus -in coolapi.priv | openssl md5
```
> Note both outputs should match

Create a combo file with both the `private` and `cert` file together:

```sh
cat coolapi.priv coolapi.crt > coolapi.combo 
```

Run the webserver:

```sh
python3 webserver.py
```

You now have a TLS server running on `127.0.0.1`

## Testing

With the webserver running, attempt to `curl` it:

```sh
$ curl https://127.0.0.1:4443
curl: (60) SSL certificate problem: unable to get local issuer certificate                                            
More details here: https://curl.haxx.se/docs/sslcerts.html
```

We got an error because we did not pass the `ca` public key for curl since it isnt in our default trust list:

```sh
$ curl --cacert ca/ca.pub https://127.0.0.1:4443
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
...
```

Yay the valid response we wanted!

## Conclusion

* We were able to generate our own certificate authority
* We were able to generate a server private key
* We were able to generate a server certificate signing request
* We signed that certificate request using our own CA
* Validated that the certificate was generate with that CA
* Validated that the certificate signed by the CA was indeed for our server private key
* Tested the key out with a simple webserver and curl
