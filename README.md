## 1. Self-signed certificate chain

This repository is a collection of certificates used in my lab environment.

The commands used to generate these certificates are listed below.

> [!Warning]
> 
> The certificates are generated with a validity of 10,958 days (30 years)

An example on generating certificates for use with Conjur is here: https://github.com/joetanx/lab-certs/blob/main/conjur.md

### 1.1. Create self-signed root certificate authority

Generate private key:

```
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-384 -out lab_root.key
```

Generate self-signed root certificate authority:

```
openssl req -x509 -new -nodes -key lab_root.key -sha256 -days 10958 -subj "/CN=Lab Root CA" -out lab_root.pem
```

> [!Note]
> 
> For a more detailed setup, use a openssl config file
>
> Using the below config file with command `openssl req -x509 -new -nodes -key lab_root.key -sha256 -days 10958 -config lab_root.cnf -out lab_root.pem` will generate a similar certificate
> 
> ```
> [ req ]
> prompt = no
> distinguished_name = req_distinguished_name
> [ req_distinguished_name ]
> commonName = Lab Root CA
> ```
> 
> There is a sample openssl config file at `/etc/pki/tls/openssl.cnf` for reference

### 1.2. Create intermediate certificate authority

Generate private key:

```
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-384 -out lab_issuer.key
```

Generate certificate signing request:

```
openssl req -new -key lab_issuer.key -subj "/CN=Lab Issuer" -out lab_issuer.csr
```
Create config file with parameters `basicConstraints=critical,CA:true,pathlen:0` to designate certificate as certificate authority:

> [!Note]
> 
> This command creates a minimal config file, there is a sample openssl config file at `/etc/pki/tls/openssl.cnf` for reference

```
echo "basicConstraints=critical,CA:true,pathlen:0" > lab_issuer.cnf
```

Generate intermediate certificate authority:

```
openssl x509 -req -in lab_issuer.csr -CA lab_root.pem -CAkey lab_root.key -CAcreateserial -days 10958 -sha256 -out lab_issuer.pem -extfile lab_issuer.cnf
```

### 1.3. Create certificates

Generate private key:

```
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-384 -out hq.ark.vx.key
```

Generate certificate signing request:

```
openssl req -new -key hq.ark.vx.key -subj "/CN=hq.ark.vx" -out hq.ark.vx.csr
```

Create config file with the required Subject Alternative Names (SANs):

> [!Note]
> 
> This command creates a minimal config file, there is a sample openssl config file at `/etc/pki/tls/openssl.cnf` for reference

```
echo "subjectAltName=DNS:ark.vx,DNS:hq.ark.vx,IP:192.168.17.201" > hq.ark.vx.cnf
```

Generate the certificate:

```
openssl x509 -req -in hq.ark.vx.csr -CA lab_issuer.pem -CAkey lab_issuer.key -CAcreateserial -days 10958 -sha256 -out hq.ark.vx.pem -extfile hq.ark.vx.cnf
```

Export certificate and key files to pkcs12 bundle:

```
openssl pkcs12 -export -out hq.ark.vx.pfx -inkey hq.ark.vx.key -in hq.ark.vx.pem -certfile lab_issuer.pem -keysig -passout pass:cyberark
```

Concatenate the intermediate CA into the certificate file:

```
cat lab_issuer.pem >> hq.ark.vx.pem
```

## 2. Miscellaneous certificate commands

|Usage|Command|
|---|---|
|Generate SSH key pair|EDDSA: `ssh-keygen -t ed25519 -f id_ed25519 -C "" -N ""`<br>ECDSA: `ssh-keygen -t ecdsa -b 384 -f id_ecdsa -C "" -N ""`<br>RSA: `ssh-keygen -b 2048 -f id_rsa -C "" -N ""`|
|Generate PKI key pair|EDDSA: `openssl genpkey -algorithm ed25519 -out ed25519.key`<br>ECDSA: `openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-384 -out ecdsa.key`<br>RSA: `openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -out rsa.key`|
|Convert key from OpenSSH to PKCS#8|`ssh-keygen -e -m PKCS8 -p -f openssh_to_pkcs8.key`<br>(Converted key is written in-place to the file specified)|
|Convert key from PKCS#8 to OpenSSH|`ssh-keygen -e -p -f pkcs8_to_openssh.key`<br>(Converted key is written in-place to the file specified)|
|Convert key from PKCS#8 to PKCS#1|`openssl pkey -in pkcs8.key -traditional -out pkcs1.key`|
|Convert key from PKCS#1 to PKCS#8|`openssl pkey -in pkcs1.key -out pkcs8.key`|
|Check live server certifcate|`openssl s_client -connect SERVER:PORT </dev/null 2>/dev/null \| openssl x509 -noout -text`|
|Check pem certifcate file|`openssl x509 -text -in CERT.pem`|
|Get fingerprint of pem certifcate file|SHA-1 (default):`openssl x509 -in CERT.pem -fingerprint`<br>SHA-256: `openssl x509 -in CERT.pem -fingerprint -sha256`|
|Check private key|`openssl pkey -check -text -in CERT.key`|
|Check CSR|`openssl req -text -noout -verify -in CERT.csr`|
|Check PKCS#12 file|`openssl pkcs12 -info -in CERT.pfx -nodes -passin pass:PASSWORD`|
|Combine cert and files into pkcs12|`openssl pkcs12 -export -out CERT.pfx -inkey CERT.key -in CERT.pem -keysig -passout pass:PASSWORD`|
|Extract cert from pkcs12|`openssl pkcs12 -in CERT.pfx -nokeys -out CERT.pem`|
|Extract key from pkcs12|`openssl pkcs12 -in CERT.pfx -nodes -out CERT.key`|
|Extract public key from private key|`openssl pkey -in CERT.key -pubout`|

<details><summary><h2>Key formats: OpenSSH, PKCS1 and PKCS8</h2></summary>

### 1. `ssh-keygen` generates keys in OpenSSH format

#### 1.1. EDDSA:

```console
[root@foxtrot ~]# ssh-keygen -t ed25519 -f id_ed25519 -C "" -N ""
Generating public/private ed25519 key pair.
Your identification has been saved in id_ed25519
Your public key has been saved in id_ed25519.pub
The key fingerprint is:
SHA256:EhP2s+9ycmcdkZPC9n6rH43yWL8h3hblgZfdf0Y1Xi0
The key's randomart image is:
+--[ED25519 256]--+
|      o         .|
|     . o      E.+|
|      o o  .  o+B|
|       o o  +.===|
|      . S  . o.=+|
|       . .    oo*|
|          . .+o=+|
|        o.o +==o+|
|         =.o.o=*+|
+----[SHA256]-----+
[root@foxtrot ~]# cat id_ed25519
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACAFUJkloiDT5MWxXA8E7YCH0hwRMvTYTrura6o8miPkKgAAAIh7jugme47o
JgAAAAtzc2gtZWQyNTUxOQAAACAFUJkloiDT5MWxXA8E7YCH0hwRMvTYTrura6o8miPkKg
AAAEA3Xmaw5GH4AC2HPzgrfo5JS7OMlHbpscEIHuZDWxTP1AVQmSWiINPkxbFcDwTtgIfS
HBEy9NhOu6trqjyaI+QqAAAAAAECAwQF
-----END OPENSSH PRIVATE KEY-----
```

#### 1.2. ECDSA:

```console
[root@foxtrot ~]# ssh-keygen -t ecdsa -b 384 -f id_ecdsa -C "" -N ""
Generating public/private ecdsa key pair.
Your identification has been saved in id_ecdsa
Your public key has been saved in id_ecdsa.pub
The key fingerprint is:
SHA256:rvBwjwu38a+c/mGdA6NTFaealivok0RrW1GBSSq+tVs
The key's randomart image is:
+---[ECDSA 384]---+
|       ..o... .  |
|       .o .  +   |
|    . .  .  o    |
|   . .. .  =     |
|    ....S.O      |
|     o++.+ = .   |
|    =o*+E + +    |
|     O+@ = . .   |
|      O+B+o      |
+----[SHA256]-----+
[root@foxtrot ~]# cat id_ecdsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAiAAAABNlY2RzYS
1zaGEyLW5pc3RwMzg0AAAACG5pc3RwMzg0AAAAYQQJImk1ZwYNIgyzkRSCc4zRix7Rb4jQ
0uhFpCGMHeAK4ebXhtiVCSmmFm85GcDKO27qpf1lqVAToJYGpnotVoLgKI1aL1HznV8kEZ
oSjvrNQlwjETHhv0gBXBsz6v3v/soAAADIX2Wf3l9ln94AAAATZWNkc2Etc2hhMi1uaXN0
cDM4NAAAAAhuaXN0cDM4NAAAAGEECSJpNWcGDSIMs5EUgnOM0Yse0W+I0NLoRaQhjB3gCu
Hm14bYlQkpphZvORnAyjtu6qX9ZalQE6CWBqZ6LVaC4CiNWi9R851fJBGaEo76zUJcIxEx
4b9IAVwbM+r97/7KAAAAMD/rPOIeD836x8wiCmcLxvhNs0GzIgO1oeuXHYCF4zixLjTUv0
UYNvkc48pRG2BhngAAAAA=
-----END OPENSSH PRIVATE KEY-----
```

#### 1.3. RSA:

```console
[root@foxtrot ~]# ssh-keygen -b 2048 -f id_rsa -C "" -N ""
Generating public/private rsa key pair.
Your identification has been saved in id_rsa
Your public key has been saved in id_rsa.pub
The key fingerprint is:
SHA256:rdliSfAWrFzFzx3yAgmFRuue8MlG/clTHIO1ZbM7S2Y
The key's randomart image is:
+---[RSA 2048]----+
|       .o=o.  ..o|
|       .oo+ .o.+o|
|      ..=  +.++o |
|     . * +  +.oo.|
|      + S o  .oE |
|       O B o o+ o|
|        & . =  . |
|       o .   .   |
|                 |
+----[SHA256]-----+
[root@foxtrot ~]# cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEArEDY23VOdThEkUxacpvabTcVqq8tFvZwnxrAgzqPrmq1y8lpKTu4
NfSZdZe38iEODKRSvLED1vZi0DFWcja+SOMSQEJrMc1gLt/JqxDLh9kktt3WFFO95GMdje
JTYK4gmlwW+jSSNNYTPXUJ2CcYsam252TlM3Qf2Q75UXinUkUhkHz0+JLpcFBqVZQt1vyd
z3AAjGsyiBBtZWP+/9979GcyUvr4HbBMNzdKhEzl10p4fGvWdgRhPU90hGBSxcU/vDgevg
geSgbD8Cvw4uaulsaZDXCdRoDGx28lmbfjtrVJtRVP+MicUqzI0D+fPXLFV89XzE9hsENy
ijzsyETo8QAAA7jRSQoZ0UkKGQAAAAdzc2gtcnNhAAABAQCsQNjbdU51OESRTFpym9ptNx
Wqry0W9nCfGsCDOo+uarXLyWkpO7g19Jl1l7fyIQ4MpFK8sQPW9mLQMVZyNr5I4xJAQmsx
zWAu38mrEMuH2SS23dYUU73kYx2N4lNgriCaXBb6NJI01hM9dQnYJxixqbbnZOUzdB/ZDv
lReKdSRSGQfPT4kulwUGpVlC3W/J3PcACMazKIEG1lY/7/33v0ZzJS+vgdsEw3N0qETOXX
Snh8a9Z2BGE9T3SEYFLFxT+8OB6+CB5KBsPwK/Di5q6WxpkNcJ1GgMbHbyWZt+O2tUm1FU
/4yJxSrMjQP589csVXz1fMT2GwQ3KKPOzIROjxAAAAAwEAAQAAAQABAf03nBPC9VrXiuEK
fUPKXAl3/Uyhm+LY8XOzInLZzPUsxywpKGvpjky4C25RP+hO9can2AH7Kw4B5KoAos/OVo
LCTLxhEK/2S9gIYWgMoeSR69GvWBwuLTZfukIteIAhFKnlpplWe3KmUaxM/xFcSu9SoplB
loic3dzJGt0HivTHMkD/9nLIkiaetskA/DBO+itKBJx7Z7w8MI8BR4IlXSywm+RGBBwjfQ
jLCi51BCG0yvOOdC/SEbdg1eWbhT79MLmnsBP2p7eSp7WP2+W3/FAjfUKpZLtKDaHwQQXL
CrvRkx10nGZDsmJRZKls9zWil6TJ0Mb4+xeHFRn2BdvJAAAAgQDJ7bm/cY4f/GfxDvHr2j
tyLfpcaDLf2mmr0g7E3kUSXou9rOExB3Z47J9sgtK7Qf51OgicjTFrRxHZkHENElhSd3mA
64cdM6Yw0sRTeeExRcnF7z1EyGDk/COkLqCw+wpiADlvekKErOuhTCpANH4Gko2dfBIdXm
uWihpzTkUCPwAAAIEA6QZHP9Yg19+Trg8QDlstz2ilwhJolHeG8YLkaiWQ/u8JbfBIo+w6
bcUsB46NlxRwZZEA7wy6qPZHcSZ/qshLJKO3BgZoOmZe1j0drpVMqIax6mbWhnHEk47a5l
5hgr6h9qdHcF3S2088o9rSHCkNbrjb8zRVgiu/3zimJhD7xGkAAACBAL08p9kCswBxjQ2v
/L3fuqWhPhLAyFjIgD5CWJfhys7DBRk09QHQj0mGCr1gjrhnfTyVFU9DZwOaWR6fecYBv2
c8SB1D8i3gw7wcQizka9pPBUJ4d4De2tL4Ztnn2uENFs5Q9zLUSuGU0syGV3MjMc+ItwPx
WXlkhegYmdXOQc9JAAAAAAEC
-----END OPENSSH PRIVATE KEY-----
```

### 2. `openssl` generates keys in PKCS8 format

#### 2.1. EDDSA:

```console
[root@foxtrot ~]# openssl genpkey -algorithm ed25519 -out ed25519_pkcs8.key
[root@foxtrot ~]# cat ed25519_pkcs8.key
-----BEGIN PRIVATE KEY-----
MC4CAQAwBQYDK2VwBCIEIJyWodKqZsYLw/kFFTJAGw8hLg0PW9KMb1IlbZ8QShnG
-----END PRIVATE KEY-----
```

#### 2.2. ECDSA:

```console
[root@foxtrot ~]# openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-384 -out ecdsa_pkcs8.key
[root@foxtrot ~]# cat ecdsa_pkcs8.key
-----BEGIN PRIVATE KEY-----
MIG2AgEAMBAGByqGSM49AgEGBSuBBAAiBIGeMIGbAgEBBDBbLlWroJ85KnKFfa34
P4k3WtLRzfUV/MbFsQc88Kr1+kvQqhk/kMsBh/3XjVGLJeehZANiAASL2cJukCqs
w4ZYLU8ASlxgCze31Xgh/3LWx+3CziCDyr8IxGt2trOr0Hu4tMri+fySfCITXp8c
6N4AYdfgklBTBA7ujcf/cp00/NpQAXChYc8iHoRMz0Wsi12Zb3LVOjE=
-----END PRIVATE KEY-----
```

#### 2.3. RSA:

```console
[root@foxtrot ~]# openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -out rsa_pkcs8.key
............+..+.+...........+.+...+.....+.+.....+...+..........+...+...+.........+..+...+..........+........+...+....+...........+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+......+...+..+....+...........+.+...+..+...+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+.....+...............+.+......+..+...+....+....................+.+..............+.+...+..............+...............+.+......+........+.......+........+.+.................+.......+..+....+............+..+.......+..............+.+...+......+......+...+..+....+...........+.+...+............+..+......+.+.....+.........+......+..................+.+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
.+..........+...............+......+..+.......+..............................+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+...+............+...+....+......+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*............+...+...+......+...............+......+.........+.+...+..............+......+.........+.+..+...+.+.....+..........+.....+...+......+............+.......+..+......+...................+...+.....+...+....+......+..+.+..+.........................+..+......+....+..+.+............+..+....+.....+.......+.....+.........+...+..........+........+.......+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
[root@foxtrot ~]# cat rsa_pkcs8.key
-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQDfm0rVrGvoflMp
IuDcn+RPrnEIDLpTmSUVJdJa2oWbp66RCPkra2bNqrTRfvqCepo8TsNtZwozaimH
mFJp511woGzew/s6oTgolwaWyEdXAKqvRMENe9R/SOpTOm3gWw2MdcE1Gpb1YmG/
J4UH6eSAsvQAcy/93SrsAZGVmwweWSFo9cC2QVCbSR1yNPU45NTEEsO5RSy9MSJJ
4F1yvdygxmhYqj+zMeR7ENFkCzYOR4o2Df3Z2iZWNe0q4JaVSePingC8YJ4u/UXi
n4nfqzEIozv/vLy4TwkiOkHwUAzq4uKFArUCmLDGYTC+4R+OFw8o7v92wBBxNnOr
VQ+yJAIDAgMBAAECggEAANDhG5laU3Mk168DeC8y5KZmBOeAINet7qz3OhvuBgX0
C7CU+ndL3QX5lctCQHg06rrZIf8Q1HagtU9i+1CFGej3YU5uV2XbNJq2MN877tYy
pK0VjPh5RuNXOdcX4meypvVqeH9sDiril9qLopUvn2m+nQwXsr0p3YDi0a7vcKvC
jmQEWn6z0Rs0YikKIIwewZOzZk93r6mbAUX9smfIj2WD0kepDWKVR8mNLt0nXM0p
CFU3I/tAIQRY/u1mYzbfnuhB4OGFoskGYHQ86TUPBUC5gce/JPRsGxwimi3Fug0J
RJvRjieJ+GCR2dOjWzDpzojC0W9Hyol2+v+oAIWGQQKBgQDi5oGWx33FDCP4lsVv
Qgvsuyv7BjXcF2gRyZ1oXa6LArwJ9nFVqN21ViAhZIJYGADA54DA6anrgP8FU13r
duPie4hBfgTPupJqrpAgdduKMHnVnRRlvxUNarK+lBHw9RXYLXRmdWAnbdp3a6NU
tdmGlySTKrnhDQU+6IGAnvix8wKBgQD8SKU4rWd0hOBC5iWBNPoSkSETApI5apii
+19H2aswPU9pTuWE6ChYplfx7IRXvXCPbmanAwnhPLHYaF4WbXz32ZrewGBz5hl6
f92gJLh4BWUlT0UrtXwiwV1OuSCK3lnZyRaK1XRZuOdvfunyBiLGBqGFKuLsTIU4
jAIzbyBjsQKBgQDVjcHWGbhj50NLywvT5UO38Yo5XuT+WwFWDH4cJmAK8e3tKogM
6TySWZcwFpsfMqgy5zClYMbOosBjUM2KuoFNPptFmMgKgz0fL2DzTDnu3CUvSgJS
qP+1ewD0ogQo12NR7aYqcLqpIZmG4EX/ipBLPqHr6UC9cjXHual5VyYWxQKBgQCI
VlX7qDJllL2BSdDw34lZaVbfaB9Pqhysz33xXV+XJTr6JSoCRlgveE3ErtXieL0Q
tlABZ7H6KAvQcK6QHkFPzChWws4dNDeGrP0/YzjRm9DKdelisqRQQAFF3uQISBt0
h6iIBMzpA/UGmyagpdI7BDBbwA58Nuoz4e36j86IMQKBgAO2+j0+XYmWE2RhhLdI
wyfxuPlNf4L6owtEtOz6OedwJleyP0EB5e9JyB8ONNf0VCFzw2B7+7pNCC7Sapbw
y/5uc1pu0jtzlIdjTgDW5DrRoQfjEyDRX1By5FgAtIf/ANDgLRQ4P0N5mZCBDd5W
yCQ1DMtlH6fxeRhSisvWwoxR
-----END PRIVATE KEY-----
```

### 3. `ssh-keygen` can convert keys between OpenSSH and PKCS8 formats

#### 3.1. EDDSA:

> [!Note]
> 
> OpenSSH doesn't currently support reading or writing Ed25519 keys in any format other than the OpenSSH native key format.
> 
> https://bugzilla.mindrot.org/show_bug.cgi?id=3195

- `ssh-keygen -e -m PKCS8 -p -f ed25519_pkcs8.key` does nothing
- `ssh-keygen -e -p -f id_ed25519` throws error: `Failed to load key id_ed25519: invalid format`

Alternatively, the [sshpk](https://www.npmjs.com/package/sshpk) npm package performs this conversion

|Usage|Command|
|---|---|
|Convert key from OpenSSH to PKCS#8|`sshpk-conv -p -t pkcs8 id_eddsa \| tee eddsa.key`|
|Convert key from PKCS#8 to OpenSSH|`sshpk-conv -p -t ssh eddsa_pkcs8.key \| tee id_eddsa`|

#### 3.2. ECDSA:

##### 3.2.1. OpenSSH to PKCS8

```console
[root@foxtrot ~]# mv id_ecdsa ecdsa_pkcs8.key
[root@foxtrot ~]# ssh-keygen -e -m PKCS8 -p -f ecdsa_pkcs8.key
Key has comment ''
Enter new passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved with the new passphrase.
[root@foxtrot ~]# cat ecdsa_pkcs8.key
-----BEGIN PRIVATE KEY-----
MIG2AgEAMBAGByqGSM49AgEGBSuBBAAiBIGeMIGbAgEBBDA/6zziHg/N+sfMIgpn
C8b4TbNBsyIDtaHrlx2AheM4sS401L9FGDb5HOPKURtgYZ6hZANiAAQJImk1ZwYN
IgyzkRSCc4zRix7Rb4jQ0uhFpCGMHeAK4ebXhtiVCSmmFm85GcDKO27qpf1lqVAT
oJYGpnotVoLgKI1aL1HznV8kEZoSjvrNQlwjETHhv0gBXBsz6v3v/so=
-----END PRIVATE KEY-----
```

##### 3.2.2. PKCS8 to OpenSSH

```console
[root@foxtrot ~]# mv ecdsa_pkcs8.key id_ecdsa
[root@foxtrot ~]# ssh-keygen -e -p -f id_ecdsa
Enter new passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved with the new passphrase.
[root@foxtrot ~]# cat id_ecdsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAiAAAABNlY2RzYS
1zaGEyLW5pc3RwMzg0AAAACG5pc3RwMzg0AAAAYQQJImk1ZwYNIgyzkRSCc4zRix7Rb4jQ
0uhFpCGMHeAK4ebXhtiVCSmmFm85GcDKO27qpf1lqVAToJYGpnotVoLgKI1aL1HznV8kEZ
oSjvrNQlwjETHhv0gBXBsz6v3v/soAAADIhAXncIQF53AAAAATZWNkc2Etc2hhMi1uaXN0
cDM4NAAAAAhuaXN0cDM4NAAAAGEECSJpNWcGDSIMs5EUgnOM0Yse0W+I0NLoRaQhjB3gCu
Hm14bYlQkpphZvORnAyjtu6qX9ZalQE6CWBqZ6LVaC4CiNWi9R851fJBGaEo76zUJcIxEx
4b9IAVwbM+r97/7KAAAAMD/rPOIeD836x8wiCmcLxvhNs0GzIgO1oeuXHYCF4zixLjTUv0
UYNvkc48pRG2BhngAAAAA=
-----END OPENSSH PRIVATE KEY-----
```

#### 3.3. RSA:

##### 3.3.1. OpenSSH to PKCS8

```console
[root@foxtrot ~]# mv id_rsa rsa_pkcs8.key
[root@foxtrot ~]# ssh-keygen -e -m PKCS8 -p -f rsa_pkcs8.key
Key has comment ''
Enter new passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved with the new passphrase.
[root@foxtrot ~]# cat rsa_pkcs8.key
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQCsQNjbdU51OESR
TFpym9ptNxWqry0W9nCfGsCDOo+uarXLyWkpO7g19Jl1l7fyIQ4MpFK8sQPW9mLQ
MVZyNr5I4xJAQmsxzWAu38mrEMuH2SS23dYUU73kYx2N4lNgriCaXBb6NJI01hM9
dQnYJxixqbbnZOUzdB/ZDvlReKdSRSGQfPT4kulwUGpVlC3W/J3PcACMazKIEG1l
Y/7/33v0ZzJS+vgdsEw3N0qETOXXSnh8a9Z2BGE9T3SEYFLFxT+8OB6+CB5KBsPw
K/Di5q6WxpkNcJ1GgMbHbyWZt+O2tUm1FU/4yJxSrMjQP589csVXz1fMT2GwQ3KK
POzIROjxAgMBAAECggEAAQH9N5wTwvVa14rhCn1DylwJd/1MoZvi2PFzsyJy2cz1
LMcsKShr6Y5MuAtuUT/oTvXGp9gB+ysOAeSqAKLPzlaCwky8YRCv9kvYCGFoDKHk
kevRr1gcLi02X7pCLXiAIRSp5aaZVntyplGsTP8RXErvUqKZQZaInN3cyRrdB4r0
xzJA//ZyyJImnrbJAPwwTvorSgSce2e8PDCPAUeCJV0ssJvkRgQcI30IywoudQQh
tMrzjnQv0hG3YNXlm4U+/TC5p7AT9qe3kqe1j9vlt/xQI31CqWS7Sg2h8EEFywq7
0ZMddJxmQ7JiUWSpbPc1opekydDG+PsXhxUZ9gXbyQKBgQDpBkc/1iDX35OuDxAO
Wy3PaKXCEmiUd4bxguRqJZD+7wlt8Eij7DptxSwHjo2XFHBlkQDvDLqo9kdxJn+q
yEsko7cGBmg6Zl7WPR2ulUyohrHqZtaGccSTjtrmXmGCvqH2p0dwXdLbTzyj2tIc
KQ1uuNvzNFWCK7/fOKYmEPvEaQKBgQC9PKfZArMAcY0Nr/y937qloT4SwMhYyIA+
QliX4crOwwUZNPUB0I9Jhgq9YI64Z308lRVPQ2cDmlken3nGAb9nPEgdQ/It4MO8
HEIs5GvaTwVCeHeA3trS+GbZ59rhDRbOUPcy1ErhlNLMhldzIzHPiLcD8Vl5ZIXo
GJnVzkHPSQKBgEkfDDqO4c17veavmVU37V8ZMnJ8vk5gV3rvnOdmFGK69ZWHAfRW
S1totNFGPU38PuzQHJ/muagNaAusjgE0Ssgbi3IbjpdMylOl5+uBtAVqBuhMDuMv
TgUTncMOOMEDOuWgRj2PY3woGBo+rxHhG/LzlSly8aYgPlw4dYKab7aJAoGAEBp+
ShhRtUL0duq3/kxwrLGY/62KHwwI5cNtmJctVAUChQ+dnebqmp4egdkarBSacrJZ
GuKofIUA+nsluLTjXdyiYmMq076hyXs6ImnZx70bvHlV6hCM3JEo53g0hxw/CZWY
Q6oPKT0p5x+zh2fCUF/Y+yvpqkvknUiiprAjp4kCgYEAye25v3GOH/xn8Q7x69o7
ci36XGgy39ppq9IOxN5FEl6LvazhMQd2eOyfbILSu0H+dToInI0xa0cR2ZBxDRJY
Und5gOuHHTOmMNLEU3nhMUXJxe89RMhg5PwjpC6gsPsKYgA5b3pChKzroUwqQDR+
BpKNnXwSHV5rlooac05FAj8=
-----END PRIVATE KEY-----
```

##### 3.3.2. PKCS8 to OpenSSH

```console
[root@foxtrot ~]# mv rsa_pkcs8.key id_rsa
[root@foxtrot ~]# ssh-keygen -e -p -f id_rsa
Enter new passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved with the new passphrase.
[root@foxtrot ~]# cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEArEDY23VOdThEkUxacpvabTcVqq8tFvZwnxrAgzqPrmq1y8lpKTu4
NfSZdZe38iEODKRSvLED1vZi0DFWcja+SOMSQEJrMc1gLt/JqxDLh9kktt3WFFO95GMdje
JTYK4gmlwW+jSSNNYTPXUJ2CcYsam252TlM3Qf2Q75UXinUkUhkHz0+JLpcFBqVZQt1vyd
z3AAjGsyiBBtZWP+/9979GcyUvr4HbBMNzdKhEzl10p4fGvWdgRhPU90hGBSxcU/vDgevg
geSgbD8Cvw4uaulsaZDXCdRoDGx28lmbfjtrVJtRVP+MicUqzI0D+fPXLFV89XzE9hsENy
ijzsyETo8QAAA7jUU8jO1FPIzgAAAAdzc2gtcnNhAAABAQCsQNjbdU51OESRTFpym9ptNx
Wqry0W9nCfGsCDOo+uarXLyWkpO7g19Jl1l7fyIQ4MpFK8sQPW9mLQMVZyNr5I4xJAQmsx
zWAu38mrEMuH2SS23dYUU73kYx2N4lNgriCaXBb6NJI01hM9dQnYJxixqbbnZOUzdB/ZDv
lReKdSRSGQfPT4kulwUGpVlC3W/J3PcACMazKIEG1lY/7/33v0ZzJS+vgdsEw3N0qETOXX
Snh8a9Z2BGE9T3SEYFLFxT+8OB6+CB5KBsPwK/Di5q6WxpkNcJ1GgMbHbyWZt+O2tUm1FU
/4yJxSrMjQP589csVXz1fMT2GwQ3KKPOzIROjxAAAAAwEAAQAAAQABAf03nBPC9VrXiuEK
fUPKXAl3/Uyhm+LY8XOzInLZzPUsxywpKGvpjky4C25RP+hO9can2AH7Kw4B5KoAos/OVo
LCTLxhEK/2S9gIYWgMoeSR69GvWBwuLTZfukIteIAhFKnlpplWe3KmUaxM/xFcSu9SoplB
loic3dzJGt0HivTHMkD/9nLIkiaetskA/DBO+itKBJx7Z7w8MI8BR4IlXSywm+RGBBwjfQ
jLCi51BCG0yvOOdC/SEbdg1eWbhT79MLmnsBP2p7eSp7WP2+W3/FAjfUKpZLtKDaHwQQXL
CrvRkx10nGZDsmJRZKls9zWil6TJ0Mb4+xeHFRn2BdvJAAAAgQDJ7bm/cY4f/GfxDvHr2j
tyLfpcaDLf2mmr0g7E3kUSXou9rOExB3Z47J9sgtK7Qf51OgicjTFrRxHZkHENElhSd3mA
64cdM6Yw0sRTeeExRcnF7z1EyGDk/COkLqCw+wpiADlvekKErOuhTCpANH4Gko2dfBIdXm
uWihpzTkUCPwAAAIEA6QZHP9Yg19+Trg8QDlstz2ilwhJolHeG8YLkaiWQ/u8JbfBIo+w6
bcUsB46NlxRwZZEA7wy6qPZHcSZ/qshLJKO3BgZoOmZe1j0drpVMqIax6mbWhnHEk47a5l
5hgr6h9qdHcF3S2088o9rSHCkNbrjb8zRVgiu/3zimJhD7xGkAAACBAL08p9kCswBxjQ2v
/L3fuqWhPhLAyFjIgD5CWJfhys7DBRk09QHQj0mGCr1gjrhnfTyVFU9DZwOaWR6fecYBv2
c8SB1D8i3gw7wcQizka9pPBUJ4d4De2tL4Ztnn2uENFs5Q9zLUSuGU0syGV3MjMc+ItwPx
WXlkhegYmdXOQc9JAAAAAAEC
-----END OPENSSH PRIVATE KEY-----
```

### 4. `openssl` can convert keys between PKCS8 and PKCS1 formats

#### 4.1. EDDSA:

PKCS1 doesn't seem to be supported for EDDSA

```console
[root@foxtrot ~]# openssl pkey -in ed25519_pkcs8.key -traditional -out ed25519_pkcs1.key
80CBA973187F0000:error:0480006E:PEM routines:PEM_write_bio_PrivateKey_traditional:unsupported public key type:crypto/pem/pem_pkey.c:345:
[root@foxtrot ~]# openssl pkey -in ed25519_pkcs8.key
```

#### 4.2. ECDSA:

##### 4.2.1. PKCS8 to PKCS1:

```console
[root@foxtrot ~]# openssl pkey -in ecdsa_pkcs8.key -traditional -out ecdsa_pkcs1.key
[root@foxtrot ~]# cat ecdsa_pkcs1.key
-----BEGIN EC PRIVATE KEY-----
MIGkAgEBBDA/6zziHg/N+sfMIgpnC8b4TbNBsyIDtaHrlx2AheM4sS401L9FGDb5
HOPKURtgYZ6gBwYFK4EEACKhZANiAAQJImk1ZwYNIgyzkRSCc4zRix7Rb4jQ0uhF
pCGMHeAK4ebXhtiVCSmmFm85GcDKO27qpf1lqVAToJYGpnotVoLgKI1aL1HznV8k
EZoSjvrNQlwjETHhv0gBXBsz6v3v/so=
-----END EC PRIVATE KEY-----
```

##### 4.2.2. PKCS1 to PKCS8:

```console
[root@foxtrot ~]# openssl pkey -in ecdsa_pkcs1.key
-----BEGIN PRIVATE KEY-----
MIG2AgEAMBAGByqGSM49AgEGBSuBBAAiBIGeMIGbAgEBBDA/6zziHg/N+sfMIgpn
C8b4TbNBsyIDtaHrlx2AheM4sS401L9FGDb5HOPKURtgYZ6hZANiAAQJImk1ZwYN
IgyzkRSCc4zRix7Rb4jQ0uhFpCGMHeAK4ebXhtiVCSmmFm85GcDKO27qpf1lqVAT
oJYGpnotVoLgKI1aL1HznV8kEZoSjvrNQlwjETHhv0gBXBsz6v3v/so=
-----END PRIVATE KEY-----
```

#### 4.3. RSA:

##### 4.3.1. PKCS8 to PKCS1:

```console
[root@foxtrot ~]# openssl pkey -in rsa_pkcs8.key -traditional -out rsa_pkcs1.key
[root@foxtrot ~]# cat rsa_pkcs1.key
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEArEDY23VOdThEkUxacpvabTcVqq8tFvZwnxrAgzqPrmq1y8lp
KTu4NfSZdZe38iEODKRSvLED1vZi0DFWcja+SOMSQEJrMc1gLt/JqxDLh9kktt3W
FFO95GMdjeJTYK4gmlwW+jSSNNYTPXUJ2CcYsam252TlM3Qf2Q75UXinUkUhkHz0
+JLpcFBqVZQt1vydz3AAjGsyiBBtZWP+/9979GcyUvr4HbBMNzdKhEzl10p4fGvW
dgRhPU90hGBSxcU/vDgevggeSgbD8Cvw4uaulsaZDXCdRoDGx28lmbfjtrVJtRVP
+MicUqzI0D+fPXLFV89XzE9hsENyijzsyETo8QIDAQABAoIBAAEB/TecE8L1WteK
4Qp9Q8pcCXf9TKGb4tjxc7MictnM9SzHLCkoa+mOTLgLblE/6E71xqfYAfsrDgHk
qgCiz85WgsJMvGEQr/ZL2AhhaAyh5JHr0a9YHC4tNl+6Qi14gCEUqeWmmVZ7cqZR
rEz/EVxK71KimUGWiJzd3Mka3QeK9McyQP/2csiSJp62yQD8ME76K0oEnHtnvDww
jwFHgiVdLLCb5EYEHCN9CMsKLnUEIbTK8450L9IRt2DV5ZuFPv0wuaewE/ant5Kn
tY/b5bf8UCN9Qqlku0oNofBBBcsKu9GTHXScZkOyYlFkqWz3NaKXpMnQxvj7F4cV
GfYF28kCgYEA6QZHP9Yg19+Trg8QDlstz2ilwhJolHeG8YLkaiWQ/u8JbfBIo+w6
bcUsB46NlxRwZZEA7wy6qPZHcSZ/qshLJKO3BgZoOmZe1j0drpVMqIax6mbWhnHE
k47a5l5hgr6h9qdHcF3S2088o9rSHCkNbrjb8zRVgiu/3zimJhD7xGkCgYEAvTyn
2QKzAHGNDa/8vd+6paE+EsDIWMiAPkJYl+HKzsMFGTT1AdCPSYYKvWCOuGd9PJUV
T0NnA5pZHp95xgG/ZzxIHUPyLeDDvBxCLORr2k8FQnh3gN7a0vhm2efa4Q0WzlD3
MtRK4ZTSzIZXcyMxz4i3A/FZeWSF6BiZ1c5Bz0kCgYBJHww6juHNe73mr5lVN+1f
GTJyfL5OYFd675znZhRiuvWVhwH0VktbaLTRRj1N/D7s0Byf5rmoDWgLrI4BNErI
G4tyG46XTMpTpefrgbQFagboTA7jL04FE53DDjjBAzrloEY9j2N8KBgaPq8R4Rvy
85UpcvGmID5cOHWCmm+2iQKBgBAafkoYUbVC9Hbqt/5McKyxmP+tih8MCOXDbZiX
LVQFAoUPnZ3m6pqeHoHZGqwUmnKyWRriqHyFAPp7Jbi0413comJjKtO+ocl7OiJp
2ce9G7x5VeoQjNyRKOd4NIccPwmVmEOqDyk9Kecfs4dnwlBf2Psr6apL5J1Ioqaw
I6eJAoGBAMntub9xjh/8Z/EO8evaO3It+lxoMt/aaavSDsTeRRJei72s4TEHdnjs
n2yC0rtB/nU6CJyNMWtHEdmQcQ0SWFJ3eYDrhx0zpjDSxFN54TFFycXvPUTIYOT8
I6QuoLD7CmIAOW96QoSs66FMKkA0fgaSjZ18Eh1ea5aKGnNORQI/
-----END RSA PRIVATE KEY-----
```

##### 4.3.2. PKCS1 to PKCS8:

```console
[root@foxtrot ~]# openssl pkey -in rsa_pkcs1.key
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQCsQNjbdU51OESR
TFpym9ptNxWqry0W9nCfGsCDOo+uarXLyWkpO7g19Jl1l7fyIQ4MpFK8sQPW9mLQ
MVZyNr5I4xJAQmsxzWAu38mrEMuH2SS23dYUU73kYx2N4lNgriCaXBb6NJI01hM9
dQnYJxixqbbnZOUzdB/ZDvlReKdSRSGQfPT4kulwUGpVlC3W/J3PcACMazKIEG1l
Y/7/33v0ZzJS+vgdsEw3N0qETOXXSnh8a9Z2BGE9T3SEYFLFxT+8OB6+CB5KBsPw
K/Di5q6WxpkNcJ1GgMbHbyWZt+O2tUm1FU/4yJxSrMjQP589csVXz1fMT2GwQ3KK
POzIROjxAgMBAAECggEAAQH9N5wTwvVa14rhCn1DylwJd/1MoZvi2PFzsyJy2cz1
LMcsKShr6Y5MuAtuUT/oTvXGp9gB+ysOAeSqAKLPzlaCwky8YRCv9kvYCGFoDKHk
kevRr1gcLi02X7pCLXiAIRSp5aaZVntyplGsTP8RXErvUqKZQZaInN3cyRrdB4r0
xzJA//ZyyJImnrbJAPwwTvorSgSce2e8PDCPAUeCJV0ssJvkRgQcI30IywoudQQh
tMrzjnQv0hG3YNXlm4U+/TC5p7AT9qe3kqe1j9vlt/xQI31CqWS7Sg2h8EEFywq7
0ZMddJxmQ7JiUWSpbPc1opekydDG+PsXhxUZ9gXbyQKBgQDpBkc/1iDX35OuDxAO
Wy3PaKXCEmiUd4bxguRqJZD+7wlt8Eij7DptxSwHjo2XFHBlkQDvDLqo9kdxJn+q
yEsko7cGBmg6Zl7WPR2ulUyohrHqZtaGccSTjtrmXmGCvqH2p0dwXdLbTzyj2tIc
KQ1uuNvzNFWCK7/fOKYmEPvEaQKBgQC9PKfZArMAcY0Nr/y937qloT4SwMhYyIA+
QliX4crOwwUZNPUB0I9Jhgq9YI64Z308lRVPQ2cDmlken3nGAb9nPEgdQ/It4MO8
HEIs5GvaTwVCeHeA3trS+GbZ59rhDRbOUPcy1ErhlNLMhldzIzHPiLcD8Vl5ZIXo
GJnVzkHPSQKBgEkfDDqO4c17veavmVU37V8ZMnJ8vk5gV3rvnOdmFGK69ZWHAfRW
S1totNFGPU38PuzQHJ/muagNaAusjgE0Ssgbi3IbjpdMylOl5+uBtAVqBuhMDuMv
TgUTncMOOMEDOuWgRj2PY3woGBo+rxHhG/LzlSly8aYgPlw4dYKab7aJAoGAEBp+
ShhRtUL0duq3/kxwrLGY/62KHwwI5cNtmJctVAUChQ+dnebqmp4egdkarBSacrJZ
GuKofIUA+nsluLTjXdyiYmMq076hyXs6ImnZx70bvHlV6hCM3JEo53g0hxw/CZWY
Q6oPKT0p5x+zh2fCUF/Y+yvpqkvknUiiprAjp4kCgYEAye25v3GOH/xn8Q7x69o7
ci36XGgy39ppq9IOxN5FEl6LvazhMQd2eOyfbILSu0H+dToInI0xa0cR2ZBxDRJY
Und5gOuHHTOmMNLEU3nhMUXJxe89RMhg5PwjpC6gsPsKYgA5b3pChKzroUwqQDR+
BpKNnXwSHV5rlooac05FAj8=
-----END PRIVATE KEY-----
```

</details>
