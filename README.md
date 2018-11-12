# OpenLDAP Proxy to OKTA

Middleware used to overcome LDAP paged results, such as with OKTA LDAP, which returns only 200 records, unless paging.

This work around was put in place because appliances that use LDAP dont support paged results, such as Palo Alto Firewalls.

## Installation

#### Install openldap and associated tools
```
apt-get update
apt-get install gcc make
apt install gnutls-bin ssl-cert
apt-get install libdb-dev libtool gnutls-dev libwrap0-dev unixodbc-dev libsasl2-dev libperl-dev
```

Download openldap
```
wget ftp://ftp.openldap.org/pub/OpenLDAP/openldap-release/openldap-2.4.46.tgz
tar zxf openldap-In2.4.46.tgz
```

Before we begin compiling, we need to "hash out" components of the code, to enable the `client-pr` function, which by default is not loaded on linux distro. This is why we are using source and compiling.
Within `servers/slapd/back-meta/` edit the following files `config.c back-meta.h search.c` and remove the following lines;
```
#ifdef SLAPD_META_CLIENT_PR
#endif /* SLAPD_META_CLIENT_PR */
```

Prep for compilation. The following was gatherd from doing a `apt-get source slapd` and finding out what is compiled on distro
```
./configure \
--prefix=/usr \
--libexecdir='${prefix}/lib' \
--sysconfdir=/etc \
--localstatedir=/var \
--mandir='${prefix}/share/man' \
--enable-debug \
--enable-dynamic \
--enable-syslog \
--enable-proctitle \
--enable-ipv6 \
--enable-local \
--enable-slapd \
--enable-dynacl \
--enable-aci \
--enable-cleartext \
--enable-crypt \
--disable-lmpasswd \
--enable-spasswd \
--enable-modules \
--enable-rewrite \
--enable-rlookups \
--enable-slapi \
--disable-slp \
--enable-wrappers \
--enable-backends=mod \
--disable-ndb \
--enable-overlays=mod \
--with-subdir=ldap \
--with-cyrus-sasl \
--with-threads \
--with-gssapi \
--with-tls=gnutls \
--with-odbc=unixodbc
```

Then execute
```
make depend
make
make install
```

#### Create certificate, required for TLS
Instructions for this copied from `https://help.ubuntu.com/lts/serverguide/openldap-server.html.en`. You might want to refer to that for later versioning; 

Create a private key for the Certificate Authority:
`sh -c "certtool --generate-privkey > /etc/ssl/private/cakey.pem"`

Create the template/file /etc/ssl/ca.info to define the CA:
```
cn = Example Company
ca
cert_signing_key
```

Create the self-signed CA certificate:
```
certtool --generate-self-signed \
--load-privkey /etc/ssl/private/cakey.pem \ 
--template /etc/ssl/ca.info \
--outfile /etc/ssl/certs/cacert.pem
```

Make a private key for the server:
```
certtool --generate-privkey \
--bits 1024 \
--outfile /etc/ssl/private/ldap01_slapd_key.pem
```

Create the /etc/ssl/ldap01.info info file containing:
```
organization = Example Company
cn = ldap01.example.com
tls_www_server
encryption_key
signing_key
expiration_days = 3650
```

Create the servers certificate:
```
certtool --generate-certificate \
--load-privkey /etc/ssl/private/ldap01_slapd_key.pem \
--load-ca-certificate /etc/ssl/certs/cacert.pem \
--load-ca-privkey /etc/ssl/private/cakey.pem \
--template /etc/ssl/ldap01.info \
--outfile /etc/ssl/certs/ldap01_slapd_cert.pem
```

#### Lets start testing the config file
Create Directory for your base dn which will be used to keep BDB files and indices for your domain
`mkdir /etc/ldap/slapd.d`

Paste `slapd.conf` into /etc/ldap directory

Test conf file and create Indices and BDB files
`slaptest -f /etc/ldap/slapd.conf -F /etc/ldap/slapd.d/`

Execute openldap with debug enabled, to check for any errors
`/usr/lib/slapd -f /etc/ldap/slapd.conf -d 255`

#### Setup init script for slapd

Paste from my git the contents of `slapd` into `/etc/inid.d/slapd`

Then
```
chmod +x /etc/init.d/slapd
ln -s /etc/init.d/slapd /etc/rc3.d/S90slapd
ln -s /etc/init.d/slapd /etc/rc4.d/S90slapd
ln -s /etc/init.d/slapd /etc/rc5.d/S90slapd
ln -s /etc/init.d/slapd /etc/rc0.d/K10slapd
ln -s /etc/init.d/slapd /etc/rc6.d/K10slapd
update-rc.d slapd defaults
```

Restart slapd
`service slapd restart`

#### Separate Logging for slapd

`vi /etc/rsyslog.d/50-default.conf` and edit following line as follows to enable slapd logging :
```
*.*;auth,authpriv,local4.none -/var/log/syslog
```
Add following line there to enable generation of slapd logs in separate file:
```
local4.* -/var/log/slapd/slapd`
```

Create configuration file to manage slapd log rotation, `vi /etc/logrotate.d/slapd`
```
/var/log/slapd/slapd
{

# Add extension of date with name of log file 
dateext

# Specify format of date appended with name of the log file
dateformat -%Y-%m-%d.log

# Keep 365 days worth of backlogs
rotate 365

# Rotate log files daily
daily

# If the log file is missing, go on to the next one without issuing an error message.
missingok

# Postpone compression of the previous log file until next rotation of logs. This only works when used in combination with
# compress
delaycompress

# Compress old log files
compress

# Lines between postrotate and endscript are executed after the log file is  rotated. Following line stands for gracefully restart 
# of rsyslog service after log rotation.
postrotate
invoke-rc.d rsyslog reload > /dev/null
endscript
}
```

`vi /etc/logrotate.conf` and uncomment following line to enable compression of all log files generated by rsyslog service:
`compress`

Restart rsyslog and slapd
```
service rsyslog restart
service slapd restart
```

## Usage

## Development

## References

## To do
