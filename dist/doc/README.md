# CAS

## Installation
### Basic Installation Overview
1. Extract distribution bundle
1. Configure TLS key and truststore
1. Configure application properties
1. Install as a systemd service

### Java Installation and Distribution Bundle Extraction

1. Install the Java 11 JRE
1. If running as a non-root user, set create that user ```useradd cas```. You may also want to increase the number of file descriptors for that user.
1. If not running as a service, it is recommended to set the JAVA_HOME environment variable
1. Download either the zip or the tar.gz archive from the repository and copy it to the install location
1. Run ```unzip cas-XXX.zip``` or ```tar -xvf cas-XXX.tar.gz```
1. If extracted as the root user, file ownership will be incorrect. Update file permissions with chown.  Ex. ```chown -R user:user service_dir```


### Configure TLS

The following is an example of how to generate a CA signed TLS certificate.  Use SOP best practices when deploying to production.

THIS IS ONLY AN EXAMPLE

If you do not have a root certificate, create one:

Create a openssl config file, ca.conf.  Below is an example.  Update your values as needed.
```
[ req ]
default_bits = 2048
default_keyfile = ca.key
encrypt_key = no
prompt = no
utf8 = yes
distinguished_name = my_req_distinguished_name
req_extensions = my_extensions
x509_extensions = x509_ext

[ my_req_distinguished_name ]
C = US
ST = Colorado
L = Boulder
O  = NCEI
CN = root

[ my_extensions ]
basicConstraints = CA:TRUE
subjectKeyIdentifier = hash

[ x509_ext ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
basicConstraints = CA:TRUE
```

Generate a CA key and certificate
```bash
openssl req -new -x509 -out ca.crt -days 3650 -config ca.conf
```

Next create a truststore with any CA certificates needed.
```bash
cp "$JAVA_HOME/jre/lib/security/cacerts" truststore.jks
keytool -import -trustcacerts -file ca.crt -alias tlsca -keystore truststore.jks
```


Next, create the TLS private key and CSR:

Create a openssl config file, tls.conf.  Below is an example.  Update your values as needed.
```
[req]
default_bits = 2048
encrypt_key = no
prompt = no
utf8 = yes
default_md = sha256
x509_extensions = v3_req
distinguished_name = dn

[dn]
C = US
CN = localhost

[v3_req]
basicConstraints = critical, CA:FALSE
subjectAltName = @alt_names
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth

[alt_names]
DNS.1 = localhost
```

Generate a key and CSR
```bash
openssl req -new -out tls.csr -config tls.conf -keyout tls.key
```

And finally, sign the certificate
```bash
openssl x509 -req -sha256 -in tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt -days 365 -extensions v3_req -extfile tls.conf
```

The next steps will create the keystore to use for TLS encryption and the truststore to trust connections to other resources like the
HazEL Auth Service and database.

First create the keystore
```bash
openssl pkcs12 -export -inkey tls.key -in tls.crt -out keystore.p12 -name tls -password pass:password
```

By default, the service will look for the keystore in the config directory with the names keystore.p12.  Copy this file
here.  If the root CA certificate was not added to the man Java truststore, the application will need to be
configured to use a custom one.  The configuration for this is located in config/jvm.options.


### Configure Application
This service uses Spring Cloud Config for configuration.  This process is bootstrapped with the configuration
in config/bootstrap.properties.  Set the eureka.client.service-url.defaultZone property to a comma separated list of URLs
to the Eureka Discovery servers, ex. https://peer-1-server.com:9001/eureka,https://peer-2-server.com:9002/eureka,https://peer-3-server.com:9003/eureka.
The application will look up the config service from Eureka.

The config service reads configuration stored in a Git repository.  The file in Git should be named cas.properties.
A sample file is provided with this distribution.

The _${svc.home}_ placeholder can be used in properties files or environment variables and represents the absolute path to the service install location.


### Running the Service

To run the service manually, not as a systemd service, execute the _run.sh_ or _start-background.sh_ script in the directory where the application was extracted.


### Install As A Service

First, navigate to the svc directory.  Edit _install-service.properties_ and set the _USER_ and _JAVA_HOME_ properties.  Then run _install-service.sh_.

## Additional Configuration
### JVM Options
All JVM options passed to the application are located in _config/jvm.options_.  Lines starting with "#" are comments and will be ignored.
Sensible defaults have been selected.  The service will need to be restarted for changes to take effect.

### Logging
Logging is configured by the _config/log4j2.xml_ file.  Detailed configuration instructions can be found here: https://logging.apache.org/log4j/2.x/manual/configuration.html.

The most common changes would be adding loggers and changing levels though. 

To add a logger add the following to the _<Loggers>_ section:
```xml
    <Logger name="com.foo.bar" level="info">
      <AppenderRef ref="File"/>
    </Logger>
```
The name will be a package name (full or partial) or a class name.  Level will be one of "fatal", "error", "warn", "info", "debug", or "trace".

To change the level of all loggers not explicitly set, update the level of the following entry in the file:
```xml
    <Root level="warn">
      <AppenderRef ref="File"/>
    </Root>
``` 

The _${sys:svc.home}_ placeholder can be used in the logging configuration and represents the absolute path to the service install location.

Changes made to this file do not require a restart.  They will be picked up within 30 seconds.