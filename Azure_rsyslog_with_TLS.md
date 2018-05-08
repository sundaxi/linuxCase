# Azure_rsyslog_with_TLS.md

Table of Contents
=================



## Introduction 

Azure rsyslog with TLS configuraiton 

Two Redhat virtual machine Client/Server

## Implementation 

### Reprequisite 

Install **rsyslog-gnutls** on client/server

```bash
yum install rsyslog-gnutls
```

Install the gnutls-utils tool 

```bash
yum install gnutls-utils
```

Clone the CA_tool from github and 

```bash
git clone https://github.com/sundaxi/CA_tools.git
cd CA_tools
```

Modify the CA_config with requirement 

### Generate the self-sign certificate 

Generate private Key of CA on a certificate Server 

```bash
./CA_cert.sh -p
Create private key in /etc/pki/CA/private/cakey.pem success
```

Genereate a self-signed CA certificate 

```bash
./CA_cert.sh -c
Self Sign ROOT CA in /etc/pki/CA/cacert.pem success
```

Generate a private key for client, running on client

```bash
./CA_cert.sh -p
Create private key in /etc/pki/CA/private/cakey.pem success
```

Generate certificate request from client 

```bash
./CA_cert.sh -r
Create the CSR under /etc/pki/CA/newcerts/client.csr success
```

Copy the CSR to the server end and sign it 

```bash
./CA_cert.sh -s client.csr
Create the cert file on client.csr.crt sucess
```

Do the same generate the certificate request and sign it on server end 
Diable the password for the private key 

```bash
./CA_cert.sh -t /etc/pki/CA/private/cakey.pem
writing RSA key
Create the /etc/pki/CA/private/cakey.pem.key success
```

Copy all the necessary certificate to /etc/pki/tls/private folder and chmod to 600 

```bash 
# server end 
ls -lrat /etc/pki/tls/private/
-rw-------. 1 root root 1743 May  8 08:36 ca-cert.pem
-rw-------. 1 root root 1245 May  8 08:36 collector-cert.pem
-rw-------. 1 root root 1743 May  8 08:36 collector-key.pem
# client end 
ls -lrat /etc/pki/tls/private/
-rw-------. 1 root root 1743 May  8 08:38 ca-cert.pem
-rw-------. 1 root root 1245 May  8 08:39 sender-cert.pem
-rw-------. 1 root root 1751 May  8 08:39 sender-key.pem
```

rsyslog configuraiton on client 

```bash
#### configuration of TLS rsyslog
# make gtls driver the default
$DefaultNetstreamDriver gtls

# certificate files
$DefaultNetstreamDriverCAFile /etc/pki/tls/private/ca-cert.pem
$DefaultNetstreamDriverCertFile /etc/pki/tls/private/sender-cert.pem
$DefaultNetstreamDriverKeyFile /etc/pki/tls/private/sender-key.pem

$ActionSendStreamDriverAuthMode x509/name
$ActionSendStreamDriverPermittedPeer server.example.com
$ActionSendStreamDriverMode 1 # run driver in TLS-only mode
*.* @@172.16.101.8:6514
```

rsyslog configuration on server 

```bash
#### configuration of TLS rsyslog
$ModLoad imtcp

# make gtls driver the default
$DefaultNetstreamDriver gtls

# certificate files
$DefaultNetstreamDriverCAFile /etc/pki/tls/private/ca-cert.pem
$DefaultNetstreamDriverCertFile /etc/pki/tls/private/collector-cert.pem
$DefaultNetstreamDriverKeyFile /etc/pki/tls/private/collector-key.pem

$InputTCPServerStreamDriverAuthMode x509/name
$InputTCPServerStreamDriverPermittedPeer *.example.com
$InputTCPServerStreamDriverMode 1 # run driver in TLS-only mode
$InputTCPServerRun 6514
```

restart rsyslog service on both server/client 
Diable firewalld and selinux on server end 

```bash
setenforce 0 
systemctl stop firewalld 
```

Do the test on client 

```bash
logger test 
```

Get the logs from server end 

```bash
# tail messages
May  8 09:22:32 server systemd: Stopping firewalld - dynamic firewall daemon...
May  8 09:22:32 server kernel: Ebtables v2.0 unregistered
May  8 09:22:32 server systemd: Stopped firewalld - dynamic firewall daemon.
May  8 09:22:53 server rsyslogd: gnutls returned error on handshake: The TLS connection was non-properly terminated.  [v8.24.0 try http://www.rsyslog.com/e/2083 ]
May  8 09:22:53 client systemd: Stopping System Logging Service...
May  8 09:22:53 client rsyslogd: [origin software="rsyslogd" swVersion="8.24.0" x-pid="37605" x-info="http://www.rsyslog.com"] exiting on signal 15.
May  8 09:22:53 client systemd: Starting System Logging Service...
May  8 09:22:53 client rsyslogd: [origin software="rsyslogd" swVersion="8.24.0" x-pid="37626" x-info="http://www.rsyslog.com"] start
May  8 09:22:53 client systemd: Started System Logging Service.
May  8 09:23:04 client yinsun: test
```







