# Install Apicurio Studion on Raspberry Pi

## Install JDK 11

```bash
ssh root@bastion.${LAB_DOMAIN}
opkg update && opkg install unzip
mkdir /tmp/work-dir
cd /tmp/work-dir
PKG="openjdk11-11 openjdk11-jdk-11 openjdk11-jre-headless-11"
for package in ${PKG}
do FILE=$(lftp -e "cls -1 alpine/edge/community/aarch64/${package}*; quit" http://dl-cdn.alpinelinux.org)
    curl -LO http://dl-cdn.alpinelinux.org/${FILE}
done
for i in $(ls)
  do tar xzf ${i}
done
mv ./usr/lib/jvm/java-11-openjdk /usr/local/java-11-openjdk
cd
rm -rf /tmp/work-dir
```

## Install PostgreSQL

```bash
opkg update && opkg install pgsql-cli pgsql-cli-extra pgsql-server
mv /var/postgresql /usr/local
uci set postgresql.config.PGDATA='/usr/local/postgresql/data'
uci set postgresql.config.PGOPTS='-h localhost'
uci commit postgresql

/usr/bin/pg_ctl -D /var/postgresql/data -l logfile start

```

## Install KeyCloak

```bash
mkdir -p /usr/local/keycloak
cd /usr/local/keycloak
KEYCLOAK_VER=$(basename $(curl -Ls -o /dev/null -w %{url_effective} https://github.com/keycloak/keycloak/releases/latest))
wget -O keycloak-${KEYCLOAK_VER}.zip https://github.com/keycloak/keycloak/releases/download/${KEYCLOAK_VER}/keycloak-${KEYCLOAK_VER}.zip
unzip keycloak-${KEYCLOAK_VER}.zip
ln -s keycloak-${KEYCLOAK_VER} keycloak-server

export PATH=/usr/local/java-11-openjdk/bin:${PATH}
export LAB_DOMAIN=my.awesome.lab
export BASTION_HOST=10.11.12.10
keytool -genkeypair -keystore /usr/local/keycloak/keystore.jks -deststoretype pkcs12 -storepass password -keypass password -alias jetty -keyalg RSA -keysize 4096 -validity 5000 -dname "CN=keycloak.${LAB_DOMAIN}, OU=okd4-lab, O=okd4-lab, L=City, ST=State, C=US" -ext "SAN=DNS:keycloak.${LAB_DOMAIN},IP:${BASTION_HOST}" -ext "BC=ca:true"

keytool -importkeystore -srckeystore /usr/local/keycloak/keystore.jks -destkeystore /usr/local/keycloak/keystore.jks -deststoretype pkcs12 -srcstorepass password
rm -f /usr/local/keycloak/keystore.jks.old

mv /usr/local/keycloak/keycloak-server/conf/keycloak.conf /usr/local/keycloak/keycloak-server/conf/keycloak.conf.orig
cat << EOF > /usr/local/keycloak/keycloak-server/conf/keycloak.conf
hostname=keycloak.${LAB_DOMAIN}
http-enabled=false
https-key-store-file=/usr/local/keycloak/keystore.jks
https-port=7443
EOF

mkdir -p /usr/local/keycloak/home
groupadd keycloak
useradd -g keycloak -d /usr/local/keycloak/home keycloak
chown -R keycloak:keycloak /usr/local/keycloak
su - keycloak
KEYCLOAK_ADMIN=admin KEYCLOAK_ADMIN_PASSWORD=admin /usr/local/keycloak/keycloak-server/bin/kc.sh start

cat <<EOF > /etc/init.d/keycloak
#!/bin/sh /etc/rc.common

START=99
STOP=80
SERVICE_USE_PID=0

start() {
   service_start /usr/bin/su - keycloak -c 'PATH=/usr/local/java-11-openjdk/bin:\${PATH} /usr/local/keycloak/keycloak-server/bin/kc.sh start > /dev/null 2>&1 &'
}

restart() {
   /usr/bin/su - keycloak -c 'PATH=/usr/local/java-11-openjdk/bin:\${PATH} /usr/local/keycloak/keycloak-server/bin/kc.sh start > /dev/null 2>&1 &'
}

stop() {
   /usr/bin/su - keycloak -c 'PATH=/usr/local/java-11-openjdk/bin:\${PATH} /usr/local/keycloak/keycloak-server/bin/kc.sh stop'
}
EOF

export APICURIO_URL=https://apicurio.${LAB_DOMAIN}:9443
wget -O ${OKD_LAB_PATH}/keycloak-realm.json https://raw.githubusercontent.com/Apicurio/apicurio-studio/master/distro/quarkus/openshift/auth/realm.json
sed -i "s|APICURIO_UI_URL|${APICURIO_URL}|g" ${OKD_LAB_PATH}/keycloak-realm.json

```

## Install Apicurio

```bash
mkdir -p /usr/local/apicurio/home
cd /usr/local/apicurio
APICURIO_VER=$(basename $(curl -Ls -o /dev/null -w %{url_effective} https://github.com/Apicurio/apicurio-studio/releases/latest))
wget -O apicurio-studio-${APICURIO_VER}-quickstart.zip https://github.com/Apicurio/apicurio-studio/releases/download/${APICURIO_VER}/apicurio-studio-${APICURIO_VER}-quickstart.zip
unzip apicurio-studio-${APICURIO_VER}-quickstart.zip
ln -s apicurio-studio-${APICURIO_VER}-quickstart apicurio-studio

groupadd apicurio
useradd -g apicurio -d /usr/local/apicurio/home apicurio
chown -R apicurio:apicurio /usr/local/apicurio

su - apicurio
export LAB_DOMAIN=my.awesome.lab
export PATH=/usr/local/java-11-openjdk/bin:${PATH}
./bin/standalone.sh -c standalone-apicurio.xml -Djboss.bind.address=10.11.12.10 -Djboss.socket.binding.port-offset=1000 -Dapicurio.kc.auth.rootUrl="https://keycloak.${LAB_DOMAIN}:7443" -Dapicurio.kc.auth.realm="apicurio" -Dapicurio-ui.hub-api.url="https://apicurio.${LAB_DOMAIN}:9443/hub-api"


cat <<EOF > /etc/init.d/apicurio
#!/bin/sh /etc/rc.common

START=99
STOP=80
SERVICE_USE_PID=0

start() {
   service_start /usr/bin/su - apicurio -c 'PATH=/usr/local/java-11-openjdk/bin:\${PATH} /usr/local/apicurio/apicurio-studio/bin/standalone.sh -c standalone-apicurio.xml -Djboss.bind.address=10.11.12.10 -Djboss.socket.binding.port-offset=1000 -Dapicurio.kc.auth.rootUrl=https://keycloak.${LAB_DOMAIN}:7443/auth > /dev/null 2>&1 &'
}

stop() {
   /usr/bin/su - apicurio -c 'kill \$(ps -x | grep apicurio | grep java | cut -d" " -f2)'
}
EOF
```
