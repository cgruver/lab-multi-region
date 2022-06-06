# Install Apicurio Studion on Raspberry Pi

## Install JDK 11
```bash
ssh bastion.${LAB_DOMAIN}
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

## Install KeyCloak

```bash
mkdir -p /usr/local/keycloak
cd /usr/local/keycloak
KEYCLOAK_VER=$(basename $(curl -Ls -o /dev/null -w %{url_effective} https://github.com/keycloak/keycloak/releases/latest))
wget -O keycloak-${KEYCLOAK_VER}.zip https://github.com/keycloak/keycloak/releases/download/${KEYCLOAK_VER}/keycloak-${KEYCLOAK_VER}.zip
unzip keycloak-${KEYCLOAK_VER}.zip
ln -s keycloak-${KEYCLOAK_VER} keycloak-server

export PATH=/usr/local/java-11-openjdk/bin:${PATH}
keytool -genkeypair -keystore /usr/local/keycloak/keystore.jks -deststoretype pkcs12 -storepass password -keypass password -alias jetty -keyalg RSA -keysize 4096 -validity 5000 -dname "CN=keycloak.${LAB_DOMAIN}, OU=okd4-lab, O=okd4-lab, L=City, ST=State, C=US" -ext "SAN=DNS:keycloak.${LAB_DOMAIN},IP:${BASTION_HOST}" -ext "BC=ca:true"

keytool -importkeystore -srckeystore /usr/local/keycloak/keystore.jks -destkeystore /usr/local/keycloak/keystore.jks -deststoretype pkcs12 -srcstorepass password
rm -f /usr/local/keycloak/keystore.jks.old

mv /usr/local/keycloak/keycloak-server/conf/keycloak.conf /usr/local/keycloak/keycloak-server/conf/keycloak.conf.orig
cat << EOF > /usr/local/keycloak/keycloak-server/conf/keycloak.conf
hostname=keycloak
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
```

## Install Apicurio

```bash
mkdir -p /usr/local/apicurio
cd /usr/local/apicurio
APICURIO_VER=$(basename $(curl -Ls -o /dev/null -w %{url_effective} https://github.com/Apicurio/apicurio-studio/releases/latest))
wget -O apicurio-studio-${APICURIO_VER}-quickstart.zip https://github.com/Apicurio/apicurio-studio/releases/download/${APICURIO_VER}/apicurio-studio-${APICURIO_VER}-quickstart.zip
unzip apicurio-studio-${APICURIO_VER}-quickstart.zip
ln -s apicurio-studio-${APICURIO_VER}-quickstart apicurio-studio


export PATH=/usr/local/java-11-openjdk/bin:${PATH}
./bin/standalone.sh -c standalone-apicurio.xml -Djboss.bind.address=10.10.10.10 -Djboss.socket.binding.port-offset=1000 -Dapicurio.kc.auth.rootUrl=https://localhost:8080/auth 
```
