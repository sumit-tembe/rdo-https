# rdo-https
Setup RDO Openstack with HTTPS

1] Prerequisites :
Execute below commands on controller :-
$ mkdir -p  /etc/pki/tls/certs
$ mkdir -p  /etc/pki/tls/private
$ mkdir -p /root/packstackca/certs
$ openssl req -x509 -sha256 -newkey rsa:2048 -keyout openstack.key -out openstack.crt -days 1024 -nodes
Note: Enter fqdn as hostname

$ cp openstack.crt /etc/pki/tls/certs/
$ cp openstack.key /etc/pki/tls/private/
$ ln -s /etc/pki/tls/certs/ssl_vnc.crt /root/packstackca/certs/$(hostname  -I | cut -f1 -d' ')ssl_vnc.crt

2] Generate Answer file:
$packstack --gen-answer-file=youranwserfile.packstack

3] Modify generated answer file:

# Disable Demo Version
CONFIG_PROVISION_DEMO=n

# Set KeyStone Admin Password or Admin user Password
CONFIG_KEYSTONE_ADMIN_PW=<password>

# Config Horizon over SSL
CONFIG_HORIZON_SSL=y

# Disable Nagios
CONFIG_NAGIOS_INSTALL=n

#Enable heat
CONFIG_HEAT_INSTALL=y

CONFIG_SSL_CERT_DIR=/root/packstackca/

4] Install Ocata: 
nohup packstack --answer-file=youranwserfile.packstack &




Post Installation :

A] Enable https for keystone:

1] Modify keystone httpd conf files:-

Update "/etc/httpd/conf.d/10-keystone_wsgi_admin.conf" file. Add below in <VirtualHost *:35357> tag:-

  ## Server aliases
  ServerAlias <ip>
  ServerAlias <fqdn>
  ServerAlias localhost

  ## SSL directives
  SSLEngine on
  SSLCertificateFile      "/etc/pki/tls/certs/openstack.crt"
  SSLCertificateKeyFile   "/etc/pki/tls/private/openstack.key"

Update "/etc/httpd/conf.d/10-keystone_wsgi_main.conf" file. Add below in <VirtualHost *:5000> tag:-

2] Add openstack.crt to ca for python:

	$ pip install certifi

	find the path of the cacert.pem file

	$python
	>>> import certifi
	>>> certifi.where()
	'/usr/lib/python2.7/site-packages/certifi/cacert.pem'

then add your own ca file in to that cacert.pem

	$cat /root/openstack.crt >> /usr/lib/python2.7/site-packages/certifi/cacert.pem

3] Create https endpoints for Keystone:

$source keystonerc_admin
$openstack endpoint create --region <Region> --enable keystone admin https://<fqdn>:35357/v3 

$openstack endpoint create --region <region> --enable keystone internal https://<fqdn>:5000/v3

$openstack endpoint create --region <region> --enable keystone public https://<fqdn>:5000/v3

4] Delete older keystone endpoints:

	#List keystone endpoints
	$openstack endpoint list | grep keystone | grep http:
	$openstack endpoint delete <endpoint id>

5] Update [ssl] section in "/etc/keystone/keystone.conf":

[ssl]
enable=true
certfile = /etc/pki/tls/certs/openstack.crt
keyfile = /etc/pki/tls/private/openstack.key

6] Restart httpd and keystone service:
	$service httpd restart
	#Install Openstack-utils
	$yum install openstack-utils -y
	$openstack-service restart
7] Update OS_AUTH_URL in keystonerc_admin and test keystone:

Replace with OS_AUTH_URL=https://<fqdn>:5000/v3 in keystonerc_admin.
$ source keystonerc_admin
$ openstack endpoint list
	
8] Update all services conf files to use https endpoints for keystone and uncomment insecure=true.Use fqdn when specifying endpoints.

	Eg."/etc/nova/nova.conf"
[keystone_authtoken]
auth_uri=https://<fqdn>:5000/
auth_url=https:/<fqdn>:35357
insecure=true

Note: Update keystone endpoint based on valued in conf files.
Eg. auth_uri=http://1.2.3.4:5000/ ⇒  auth_uri=https://<fqdn>:500

Do same for placement api.

B] Enable https for other services:
We are using haproxy for this.

1] Install and configure haproxy:

$ yum install haproxy -y
$cat openstack.crt openstack.key > /etc/haproxy/openstack.pem

Download and Replace haproxy.cfg with ⇒  haproxy.cfg

Note: Update <fqdn>, <ip> in haproxy.cfg.

2] Update ports for services:













References:

https://blog-rcritten.rhcloud.com/?p=5
http://liuhongjiang.github.io/hexotech/2016/12/23/setup-your-own-ca/#for-python-requests-to-add-ca