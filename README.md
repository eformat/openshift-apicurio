### Apicurio

Instructions for installing Apicurio on premise using OpenShift 3.7, RedHat SSO, EAP-7.1, MySQL-5.7 container images.

#### OpenShift Installation 

Clone this project and change directory:

```
cd openshift
```

Deploy using an EAP-7.1 image in OpenShift, import the image as a cluster amdin:

```
oc import-image registry.access.redhat.com/jboss-eap-7/eap71-openshift --confirm -n openshift
```

As a normal user, create a project and a binary build:

```
oc new-project apicurio  --description='Apicurio' --display-name='Apicurio'
oc new-build --strategy=source --name=apicurio --binary -l app=apicurio -i eap71-openshift
oc start-build apicurio --from-dir=. --follow
oc new-app apicurio
oc expose svc/apicurio
```

Create an SSO server:

```
oc create -f https://raw.githubusercontent.com/jboss-openshift/application-templates/master/secrets/sso-app-secret.json -n apicurio
oc create sa sso-service-account
oc policy add-role-to-user edit system:serviceaccount:$(oc project -q):sso-service-account
oc new-app sso71-mysql-persistent -p HTTPS_NAME=jboss -p HTTPS_PASSWORD=mykeystorepass -p SSO_ADMIN_USERNAME=ssoUser -p SSO_ADMIN_PASSWORD=ssoPassword
```

Create a standalone mysql database for apicurio to store our API artefacts:

```
oc new-app mysql-persistent -p MYSQL_USER="dbuser" -p MYSQL_PASSWORD="password" -p MYSQL_DATABASE="apicurio"
```

Once standalone MySQL is running, deploy the apicurio schema:

```
oc rsh <mysql pod name>
mysql -u $MYSQL_USER -p$MYSQL_PASSWORD -h $HOSTNAME $MYSQL_DATABASE
use apicurio;
-- cut and paste ddl from ./dbconfigs directory
```

Import SSO realm:

Login to SSO and import the apicurio realm

```
-- from the SSO user interface, import realm-file
./apicurio-local-realm.json
```

Login to Apicurio Studio using `demo/demo`

```
http://apicurio-apicurio.apps.example.com/studio
```

### Setup this repository from scratch

Download zip file from www.apicur.io

wget the apicurio database file from github - https://raw.githubusercontent.com/Apicurio/apicurio-studio/master/back-end/hub-core/src/main/resources/io/apicurio/hub/core/storage/jdbc/hub_mysql5.ddl

wget the secrets file for EAP from - https://raw.githubusercontent.com/jboss-openshift/application-templates/master/secrets/sso-app-secret.json

MySQL support for apicurio is documented here - https://apicurio-studio.readme.io/v0.2.1/docs/switching-to-mysql-or-postgresql

Download the mysql client jar from here - https://dev.mysql.com/downloads/connector/j/

Extract and copy standalone/deployment war files and standalone-apicurio.xml configuration into the appropriate folders (this layout is used by the EAP S2I image). Rename the apicurio EAP file to `standalone-openshift.xml`

```
openshift/
├── apicurio-local-realm.json
├── configuration
│   └── standalone-openshift.xml
├── dbscripts
│   ├── hub_mysql5.ddl
│   ├── upgrade-1_mysql5.ddl
│   └── upgrade-2_mysql5.ddl
├── deployments
│   ├── apicurio-studio-api.war
│   ├── apicurio-studio.war
│   ├── apicurio-studio-ws.war
│   └── mysql-connector-java-5.1.45-bin.jar
└── sso-app-secret.json
```

Edit the `standalone-openshift.xml` and replace `h2` config with `mysql` and configure the `keycloak` subsystem e.g.

```
-- properties section
    <system-properties>
        <property name="apicurio.kc.auth.rootUrl" value="http://sso-apicurio.apps.<your domain>/auth"/>
        <property name="apicurio.kc.auth.realm" value="apicurio-local"/>
        <property name="apicurio.hub.storage.jdbc.type" value="mysql5"/>
        <property name="apicurio.hub.storage.jdbc.init" value="false"/>
    </system-properties>

-- mysql datasource (use the proper credentials as per mysql template deployment)

		<datasource jndi-name="java:jboss/datasources/ApicurioDS" pool-name="ApicurioDS" enabled="true" use-java-context="true">
		  <connection-url>jdbc:mysql://mysql:3306/apicurio</connection-url>
		  <driver>mysql-connector-java-5.1.45-bin.jar_com.mysql.jdbc.Driver_5_1</driver>
		  <security>
		    <user-name>dbuser</user-name>
		    <password>password</password>
		  </security>
		</datasource>

-- keycloak section (match your realm config)

        <subsystem xmlns="urn:jboss:domain:keycloak:1.1">
            <realm name="apicurio-local">
                <realm-public-key>MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvq5bOyrTX9TBlPn0Sz5+QtSSEKTsyFXzdbHSBdDhoX8L2GaFOxf+jiFcAUay4Wrs6Kyv4ILXguVVqzJruIp8xd2KLj2lY46u81fjV091urTsQ76yCIu3Uf414PwHqTies1aCr0LsRCawakloXeX1mz9BSO5aBQg7PGrcahwHq8X/wu9JB44NWa7qCTLlUinpaYEI15BRrSViVQRorYajKDS0j8Rt5gcYtT6guqCKKREr0S0Dg3R/kByLDyElaN5B/d6JX+znp9DQpoXHjmGKbUyN1lkU2s549O+Fnl5RcitjXdYm7uKAeQygHn+tc8ku08/p3iOfivkcyiqtgw+o5wIDAQAB</realm-public-key>
                <auth-server-url>${apicurio.kc.auth.rootUrl}</auth-server-url>
                <ssl-required>NONE</ssl-required>
                <enable-cors>true</enable-cors>
                <principal-attribute>preferred_username</principal-attribute>
            </realm>
            <secure-deployment name="apicurio-studio-api.war">
                <realm>apicurio-local</realm>
                <resource>apicurio-api</resource>
                <bearer-only>true</bearer-only>
                <disable-trust-manager>true</disable-trust-manager>
                <credential name="secret">9e228b2a-8fde-438c-bcf7-6e8c630ab156</credential>
            </secure-deployment>
            <secure-deployment name="apicurio-studio.war">
                <realm>apicurio-local</realm>
                <resource>apicurio-studio</resource>
                <public-client>true</public-client>
                <disable-trust-manager>true</disable-trust-manager>
            </secure-deployment>
        </subsystem>
```

Create SSO configuration:

```
-- login to SSO server using credentials passed into template (ssoUser/ssoPassword)
-- create a new 'Apicurio-local' realm
-- turn off SSL for the realm for now
-- logout and login to SSO using http:// portal endpoint
-- create new client 'apicurio-api' bearer-only openidconnect
-- create new client 'apicurio-studio' public openid-connect
   set Valid Redirect URL's: http://apicurio-apicurio.apps.<your domain>/studio/*
   set Web Origins: http://apicurio-apicurio.apps.<your domain>
-- create a realm 'demo' user
```

You can export the realm config using:

```
oc scale --replicas=1 dc sso
oc env dc/sso -e "JAVA_OPTS_APPEND=-Dkeycloak.migration.action=export -Dkeycloak.migration.provider=singleFile -Dkeycloak.migration.realmName=Apicurio-local -Dkeycloak.migration.usersExportStrategy=REALM_FILE -Dkeycloak.migration.file=/tmp/apicurio-local-realm.json"
oc scale --replicas=1 dc sso
oc rsync sso-3-h2tqx:/tmp/apicurio-local-realm.json .
```
