#### Configure OpenLDAP to talk to KDC

LDAP [Phần 1] – Hướng dẫn cài đặt LDAP trên CentOS 7 https://news.cloud365.vn/huong-dan-cai-dat-ldap-tren-centos-7/
Installing Kerberos on Redhat 7 https://gist.github.com/ashrithr/4767927948eca70845db

- (Optional) First install OpenLDAP (not needed if already installed)
```
export LDAP_PASSWORD=123
export LDAP_ADMIN_USER=Manager
export DOMAIN=lakehouse


# slappasswd 123

yum -y install openldap-servers openldap-clients krb5-server-ldap phpldapadmin
vi /etc/openldap/slapd.d/cn\=config/olcDatabase\=\{2\}bdb.ldif 
	olcSuffix: dc=lakehouse,dc=com
	olcRootDN: cn=Manager,dc=lakehouse,dc=com
	olcRootPW: {SSHA}p8MC0Oikbm+HVOotK+Ur1URzabz1DXJF
	

cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown -R ldap:ldap /var/lib/ldap
chmod -R 700 /var/lib/ldap

vi /etc/sysconfig/ldap
# Options of slapd (see man slapd)
# Use this one to debug ACL...
# SLAPD_OPTIONS="-4 -d 128"
# Use this one for day-to-day production usage.
SLAPD_OPTIONS="-4"
# Run slapd with -h "... ldap:/// ..."
#   yes/no, default: yes
SLAPD_LDAP=yes
# Run slapd with -h "... ldapi:/// ..."
#   yes/no, default: yes
SLAPD_LDAPI=yes
# Run slapd with -h "... ldaps:/// ..."
#   yes/no, default: no
SLAPD_LDAPS=no
# Maximum allowed time to wait for slapd shutdown on 'service ldap 
# stop' (in seconds)
SLAPD_SHUTDOWN_TIMEOUT=15


vi /etc/openldap/ldap.conf
#add to bottom
BASE dc=lakehouse,dc=com
URI ldap://localhost
TLS_REQCERT never

vi /etc/rsyslog.conf
# Send slapd(8c) logs to /var/log/slapd.log
if $programname == 'slapd' then /var/log/slapd.log
& ~

touch /var/log/slapd.log

service rsyslog restart

service slapd start

ldapwhoami -D cn=Manager,dc=lakehouse,dc=com -w 123

wget https://github.com/tungtv289/ambari-security-workshops/raw/tungtv289/ldif/base.ldif
wget https://github.com/tungtv289/ambari-security-workshops/raw/tungtv289/ldif/groups.ldif
wget https://github.com/tungtv289/ambari-security-workshops/raw/tungtv289/ldif/users.ldif

# Fixed(1): "ldap_add: Invalid syntax (21) additional info: objectClass: value #0 invalid per syntax"
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
##################
ldapadd -H ldap://localhost:389 -x -a -D "cn=Manager,dc=lakehouse,dc=com" -f base.ldif -w 123
ldapadd -H ldap://localhost:389 -x -a -D "cn=Manager,dc=lakehouse,dc=com" -f groups.ldif -w 123
ldapadd -H ldap://localhost:389 -x -a -D "cn=Manager,dc=lakehouse,dc=com" -f users.ldif -w 123
##################

ldapsearch -h localhost -D "cn=Manager,dc=lakehouse,dc=com" -w 123 -b "dc=lakehouse,dc=com"
```


- Next install KDC 
```
export KDC_REALM=LAKEHOUSE.COM
export KDC_HOST=ldap
export KDC_DOMAIN=lakehouse.com

#export KDC_ADMIN=admin/admin
export KDC_PASSWORD=DfAr@e#$DFSyA9DD
export KDC_ADMINPASSWORD=DfAr@e#$DFSyA9DD
export TEMP_DIR=/root/ldap

export LDAP_ADMIN_USER=Manager
export KDC_DOMAIN_DN="dc=lakehouse,dc=com"

#on each node
yum install -y krb5-workstation 

#on admin node
yum install -y krb5-pkinit-openssl krb5-libs krb5-server-ldap krb5-server 
```


- Configure OpenLDAP to talk to KDC
```
cp /usr/share/doc/krb5-server-ldap-1.10.3/kerberos.schema /etc/openldap/schema/

mkdir $TEMP_DIR
cd $TEMP_DIR
mkdir  $TEMP_DIR/ldif_output
export SCHEMA_CONV=$TEMP_DIR/schema_convert.conf 

echo "include /etc/openldap/schema/core.schema" > $SCHEMA_CONV
echo "include /etc/openldap/schema/collective.schema" >> $SCHEMA_CONV
echo "include /etc/openldap/schema/corba.schema" >> $SCHEMA_CONV
echo "include /etc/openldap/schema/cosine.schema" >> $SCHEMA_CONV
echo "include /etc/openldap/schema/duaconf.schema" >> $SCHEMA_CONV
echo "include /etc/openldap/schema/dyngroup.schema" >> $SCHEMA_CONV
echo "include /etc/openldap/schema/inetorgperson.schema" >> $SCHEMA_CONV
echo "include /etc/openldap/schema/java.schema" >> $SCHEMA_CONV
echo "include /etc/openldap/schema/misc.schema" >> $SCHEMA_CONV
echo "include /etc/openldap/schema/nis.schema" >> $SCHEMA_CONV
echo "include /etc/openldap/schema/openldap.schema" >> $SCHEMA_CONV
echo "include /etc/openldap/schema/ppolicy.schema" >> $SCHEMA_CONV
echo "include /etc/openldap/schema/kerberos.schema" >> $SCHEMA_CONV

slapcat -f schema_convert.conf -F /root/ldap/ldif_output -n0 -s "cn={12}kerberos,cn=schema,cn=config" > cn=kerberos.ldif

cp cn\=kerberos.ldif cn\=kerberos.ldif.orig

#remove {12} from /root/ldap/cn\=kerberos.ldif
sed -i "s/{12}kerberos/kerberos/g" cn\=kerberos.ldif

#remove bottom lines
head -n -8 cn\=kerberos.ldif > cn\=kerberos.ldif.tmp
/bin/rm -f cn\=kerberos.ldif 
mv cn\=kerberos.ldif.tmp cn\=kerberos.ldif   

ldapadd -c -Y EXTERNAL -H ldapi:/// -f cn\=kerberos.ldif


#vi /etc/openldap/slapd.d/cn\=config/olcDatabase\=\{2\}bdb.ldif
#add olcDbIndex: krbPrincipalName eq,pres,sub
sed -i "s/olcDbLinearIndex: FALSE/olcDbIndex: krbPrincipalName eq,pres,sub\nolcDbLinearIndex: FALSE/g" /etc/openldap/slapd.d/cn\=config/olcDatabase\=\{2\}bdb.ldif



mv /var/kerberos/krb5kdc/kdc.conf /var/kerberos/krb5kdc/kdc.conf.orig
wget https://github.com/abajwa-hw/kdc-stack/raw/master/package/templates/kdc.conf -P /var/kerberos/krb5kdc/
sed -i "s/EXAMPLE.COM/$KDC_REALM/g" /var/kerberos/krb5kdc/kdc.conf

#update ACL file
sed -i "s/EXAMPLE.COM/$KDC_REALM/g" /var/kerberos/krb5kdc/kadm5.acl

mv /etc/krb5.conf /etc/krb5.conf.orig
wget https://github.com/abajwa-hw/kdc-stack/raw/master/package/templates/krb5.conf -P /etc
sed -i "s/EXAMPLE.COM/$KDC_REALM/g" /etc/krb5.conf
sed -i "s/kerberos.example.com/$KDC_HOST/g" /etc/krb5.conf
sed -i "s/example.com/$KDC_DOMAIN/g" /etc/krb5.conf
sed -i "s/dc=EXAMPLE,dc=COM/$KDC_DOMAIN_DN/g" /etc/krb5.conf
sed -i "s/Manager/$LDAP_ADMIN_USER/g" /etc/krb5.conf

#create KDC entries in LDAP

kdb5_ldap_util -D "cn=$LDAP_ADMIN_USER,dc=lakehouse,dc=com" create -subtrees "ou=kerberos,dc=lakehouse,dc=com" -r LAKEHOUSE.COM -s -H ldapi:///
# Fixed(2): kdb5_ldap_util: Kerberos Container create FAILED: Object class violation while creating realm 'LAKEHOUSE.COM' -> change to cn=kerberos,dc=lakehouse,dc=com and modify /etc/krb5.conf(ldap_kerberos_container_dn)

kdb5_ldap_util -D "cn=$LDAP_ADMIN_USER,dc=lakehouse,dc=com" create -subtrees "cn=kerberos,dc=lakehouse,dc=com" -r LAKEHOUSE.COM -s -H ldap://localhost:389

#kdb5_ldap_util -D "cn=$LDAP_ADMIN_USER,dc=lakehouse,dc=com" create -subtrees "dc=lakehouse,dc=com" -r LAKEHOUSE.COM -s -H ldapi:///


ldapsearch -LLLY EXTERNAL -H ldapi:/// -b cn=kerberos,dc=lakehouse,dc=com dn

mkdir /etc/krb5.d

kdb5_ldap_util -D "cn=$LDAP_ADMIN_USER,dc=lakehouse,dc=com" stashsrvpw -f /etc/krb5.d/stash.keyfile "cn=$LDAP_ADMIN_USER,dc=lakehouse,dc=com"

cat /etc/krb5.d/stash.keyfile

touch /var/log/krb5kdc.log /var/log/kadmind.log
```

- Setup servie logs for KDC services
```
vi /etc/rsyslog.conf
# Send kadmind(8) logs to /var/log/kadmind.log
if $programname == 'kadmind' then /var/log/kadmind.log
& ~

# Send krb5kdc(8) logs to /var/log/krb5kdc.log
if $programname == 'krb5kdc' then /var/log/krb5kdc.log
& ~

service rsyslog restart
```
- Start KDC services
```
chkconfig krb5kdc on
chkconfig kadmin on

/etc/init.d/krb5kdc start
/etc/init.d/kadmin start
```

- Add principal to business user
```
kadmin.local
addprinc ali@LAKEHOUSE.COM
#lakehouse
exit
```

- Kinit as business user and query HDFS
```
su ali
kinit
hadoop fs -ls /
```