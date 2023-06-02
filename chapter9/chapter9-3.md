
## Crypto

### What is SSL?
Secure Sockets Layer, this is its full name, its role is a protocol, which defines the format and rules for encrypting data sent over the network.

```js
      +------+                                            +------+
Server | data | -- SSL encryption --> Send --> Receive -- SSL decryption -- | data | Client 
      +------+                                            +------+   

      +------+                                            +------+
Server | data | -- SSL decryption --> Receive <-- Send -- SSL encryption -- | data | Client 
      +------+                                            +------+
```
> Note: TLS 1.0 is equivalent to SSL 3.1, TLS 1.1 is equivalent to SSL 3.2, and TLS 1.2 is equivalent to SSL 3.3.

### OpenSSL
OpenSSL is an implementation of the SSL standard in the program, providing:

* libcrypto universal encryption library
* libssl TLS/SSL implementation
* openssl command line tool

Node.js is fully encrypted using OpenSSL, and its related TLS HTTPS server module and Crypto encryption module are called OpenSSL at the bottom layer through C++.

OpenSSL implements symmetric encryption:
```shell
AES(128) | DES(64) | Blowfish(64) | CAST(64) | IDEA(64) | RC2(64) | RC5(64)
```
Asymmetric encryption:
```shell
DH | RSA | DSA | EC
```
And some message digests:
```shell
MD2 | MD5 | MDC2 | SHA | RIPEMD | DSS
```
The message digest is a type of encryption that uses a hash algorithm, which means that this encryption is one-way and cannot be decrypted in reverse. This type of encryption is mostly used to protect secure passwords, such as login passwords. The most commonly used ones are MD5 and SHA (it is recommended to use more stable SHA1, MD5 is no longer one-way through the table lookup method).

Using asymmetric encryption will consume performance, and symmetric encryption cannot be transmitted on the network, so what should I do?
The answer is: combine the two together. `ssl/tls` is actually the same, and `tls` combines the two perfectly.

### TLS HTTPS server
To establish a Node.js TLS server, you need to use the tls module:

`var Tls = require('tls');`

Before starting to build the server, we have some important work to do, that is, certificates and signed certificates.

Based on SSL encryption, when establishing a connection with the client, the server will send a signed certificate. The client stores some recognized authoritative certificate authentication agencies, that is, CA. The client matches the signature agency on the certificate sent by the server by looking up in its own CA table to determine whether the server facing is a trusted server.

If the signature agency on the certificate sent by this server is not in the client's CA list, then this server is likely to be forged, and you should have heard of "man-in-the-middle attacks".

```js
+------------+                                
| Real server |              Choice: Connect? Not connected?  +-------+
+------------+             +------------------ | Client |
                    https  |                   +-------+
+--------------+           | Intercept communication packets       
| Attacker's server | ----------+
+--------------+  The certificate is forged
```



### Summary
The security of encryption is mainly determined by the following two factors

The encryption algorithm used
* Length of key
* Under the same algorithm, the difficulty of brute force cracking increases exponentially as the key length increases by one bit. If symmetric encryption is used, AES-256 is now secure enough.

However, it is said that the 1024-bit key of RSA can be brute-forced by the government with very expensive equipment, so when using the RSA algorithm, the key length should be selected as 2048, and there is no absolute security.

In terms of encryption performance,

### Reference
* http://www.jianshu.com/p/a8b87e436ac7

