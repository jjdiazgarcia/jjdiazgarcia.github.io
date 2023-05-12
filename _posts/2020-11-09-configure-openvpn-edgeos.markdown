---
layout: post
title:  "Configure OpenVPN EdgeOS"
date:   2020-11-09 18:00:00 +0200
categories: openvpn edgeos
---
In this post, I am going to detail steps I did to get OpenVPN configured and working using an EdgeRouter-X (EdgeOS OS 1.10.9).
The main goal of having a VPN server is to access my local home network from outside of my home in a secure way.

# Generating certificates and private keys

1. Login to the router through SSH using a sudo user 
    ```
    ssh ubnt@<ip-router>
    sudo su -
    ```
2. Create a Certification Authority (CA) that will be the one used to create certificates to be able to connect to VPN.
    ```
    cd /usr/lib/ssl/misc/
    ./CA.sh -newca
    ```
    Some info will be requested
    ```
    PEM Passphrase: Abc123..
    Country Name: ES
    State Or Province Name: Malaga
    Locality Name: Malaga
    Organization Name: Jeronimo Diaz
    Organizational Unit Name:
    Common Name: UBNT Server
    Email Address: email@example.com
    ```
3. Create the certificate that will be served by OpenVPN server
    ```
    ./CA.sh -newreq
    ```
    We will need to fill some info as we did to create CA
    ```
    Country Name: ES
    State Or Province Name: Malaga
    Locality Name: Malaga
    Organization Name: Jeronimo Diaz
    Organizational Unit Name:
    Common Name: Server
    Email Address: email@example.com
    ```
4. Sign generated certificate to allow server trusts it
    ```
    ./CA.sh -sign
    ```
5. Copy certificate and CA to the route where OpenVPN server will look for them
    ```
    cp /usr/lib/ssl/misc/demoCA/cacert.pem /config/auth/
    cp /usr/lib/ssl/misc/demoCA/private/cakey.pem /config/auth/
    mv /usr/lib/ssl/misc/newcert.pem /config/auth/server.pem
    mv /usr/lib/ssl/misc/newkey.pem /config/auth/server.key
    ```
6. Decrypt key generated for the previous certificate (when a certificate is requested, private keys are generated encrypted)
    ```
    openssl rsa -in /config/auth/server.key -out /config/auth/server-decrypted.key
    mv /config/auth/server-decrypted.key /config/auth/server.key
    ```
7. Generate a [Diffie-Hellman][diffie-hellman] key to allow secure connections between OpenVPN server and clients connected to it. This key is used to cipher information in the client side and decipher it in the server side.
    ```
    openssl dhparam -out /config/auth/dh2048.pem -2 2048
    ```
8. Generate a certificate revocation list to allow revoking certificates in the future if we need to revoke any certificate
    ```
    openssl ca -gencrl -out crl.pem
    cp crl.pem /config/auth
    ```
    By default, this list expires every 30 days. It means that we need to regenerate it if we want to keep expired certificates keep marked as expired so clients are not allowed to use OpenVPN using expired certificates.
9. Last but not least, we must generate a TLS key so OpenVPN only reponds to requests that contain the TLS key generated
    ```
    openvpn --genkey --secret ta.key
    cp ta.key /config/auth
    ```

# OpenVPN configuration

1. Configure OpenVPN using below commands
    ```
    configure
    set interfaces openvpn vtun0 mode server
    set interfaces openvpn vtun0 server subnet X.X.X.X/X
    set interfaces openvpn vtun0 server push-route X.X.X.X/X
    set interfaces openvpn vtun0 server name-server X.X.X.X
    set interfaces openvpn vtun0 tls ca-cert-file /config/auth/cacert.pem
    set interfaces openvpn vtun0 tls cert-file /config/auth/server.pem
    set interfaces openvpn vtun0 tls key-file /config/auth/server.key
    set interfaces openvpn vtun0 tls dh-file /config/auth/dh2048.pem
    set interfaces openvpn vtun0 tls crl-file /config/auth/crl.pem
    set interfaces openvpn vtun0 description "OpenVPN server"
    set interfaces openvpn vtun0 encryption aes256
    set interfaces openvpn vtun0 hash sha256
    set interfaces openvpn vtun0 openvpn-option "--port 1194"
    set interfaces openvpn vtun0 openvpn-option --tls-server
    set interfaces openvpn vtun0 openvpn-option "--tls-auth /config/auth/ta.key 0"
    set interfaces openvpn vtun0 openvpn-option "--comp-lzo yes"
    set interfaces openvpn vtun0 openvpn-option --persist-key
    set interfaces openvpn vtun0 openvpn-option --persist-tun
    set interfaces openvpn vtun0 openvpn-option "--keepalive 10 120"
    set interfaces openvpn vtun0 openvpn-option "--user nobody"
    set interfaces openvpn vtun0 openvpn-option "--group nogroup"
    set interfaces openvpn vtun0 openvpn-option "--ifconfig-pool-persist /var/log/ipp.txt"
    set interfaces openvpn vtun0 openvpn-option "--status /var/log/openvpn-status.log"
    set service dns forwarding listen-on vtun0
    ```
    All X characters must be replaced in lines 2-4
2. Inbound connections must be allowed to the interface used by openvpn
    ```
    set firewall name WAN_LOCAL rule 30 action accept
    set firewall name WAN_LOCAL rule 30 description OpenVPN
    set firewall name WAN_LOCAL rule 30 destination port 1194
    set firewall name WAN_LOCAL rule 30 protocol udp
    commit
    save
    exit
    ```

# Generate certificate for client
```
cd /usr/lib/ssl/misc
./CA.sh -newreq
./CA.sh -sign
mv newcert.pem /config/auth/client1.pem
mv newkey.pem /config/auth/client1.key
openssl rsa -in /config/auth/client1.key -out /config/auth/client1-decrypted.key
mv /config/auth/client1-decrypted.key /config/auth/client1.key
```

Client should use below file to connect to OpenVPN server
```
client
dev tun
proto udp
remote domain\ip 1194
cipher AES-256-CBC
redirect-gateway def1
auth SHA256
resolv-retry infinite
nobind
comp-lzo yes
persist-key
persist-tun
user nobody
group nogroup
verb 3
key-direction 1
<ca>
  paste /config/auth/cacert.pem
</ca>
<cert>
  paste /config/auth/client.pem
</cert>
<key>
  paste /config/auth/client.key
</key>
<tls-auth>
  paste /config/auth/ta.key
</tls-auth>
```

Bear in mind that domain/ip (line 4) must be replaced by the OpenVPN server URL or IP and content of files must be added to the latest four sections in the file (ca, cert, key and tls-auth).

# Revoke a certificate

1. Get certificate CN
    ```
    cat /usr/lib/ssl/misc/demoCA/index.txt

    V       231101212507Z           89A35FBCB65F3220        unknown /C=ES/ST=Malaga/O=Jeronimo Diaz/CN=UBNT Jeronimo Diaz Server/emailAddress=xxxxxxxx
    V       211101213123Z           89A35FBCB65F3221        unknown /C=ES/ST=Malaga/L=Malaga/O=Jeronimo Diaz/CN=VPN Server/emailAddress=xxxxxxxxxx
    V       211101213503Z           89A35FBCB65F3222        unknown /C=ES/ST=Malaga/L=Malaga/O=Jeronimo Diaz/CN=Example Common Name/emailAddress=xxxxxxxx
    ```
    In this example, the certificate that will be used is ```89A35FBCB65F3222```
2. Regenerate the list of revoked certificates that include the new certificate to be revoken
    ```
    cd /usr/lib/ssl/misc
    openssl ca -revoke demoCA/newcerts/89A35FBCB65F3222.pem
    ```
3. Replace the old list of revoke certificates with the new one
    ```
    openssl ca -gencrl -out crl.pem
    cp crl.pem /config/auth
    ```

# Notes

To check when the revoked certificates list should be renewed, use below command
```
openssl crl -in /config/auth/crl.pem -noout -text
```

[diffie-hellman]: https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange