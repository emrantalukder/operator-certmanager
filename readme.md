# CloudFlare's PKI toolkit - Generate Certificates
```sh
brew install cfssl
```


## CA
```sh
cfssl gencert -initca ca-csr.json | cfssljson -bare certs/ca -
```

### validate
```sh
openssl x509 -in ca.pem -text -noout
```


## server certificate

cfssl gencert -ca=certs/ca.pem -ca-key=certs/ca-key.pem -config=ca-config.json -profile=server server.json | cfssljson -bare certs/server

openssl x509 -in certs/server.pem -text -noout

```sh
cfssl gencert -ca=certs/ca.pem -ca-key=certs/ca-key.pem -config=ca-config.json -profile=server server.json | cfssljson -bare certs/server
```

The output from above command: /tmp/server.pem & /tmp/server-key.pem. Use this file in fullchain: & privKey in helm charts respectively.


## components
```sh
cfssl gencert -ca=certs/ca.pem -ca-key=certs/ca-key.pem -config=ca-config.json -profile=server kafka.json | cfssljson -bare certs/kafka
```

## create keystore and truststore

## client

cfssl gencert -ca=certs/ca.pem -ca-key="certs/ca-key.pem" -config="ca-config.json" -profile=server "kafka.json" | cfssljson -bare "certs/kafka"

cfssl gencert -ca=certs/ca.pem -ca-key="certs/ca-key.pem" -config="ca-config.json" -profile=server "controlcenter.json" | cfssljson -bare "certs/controlcenter"

cfssl gencert -ca=certs/ca.pem -ca-key="certs/ca-key.pem" -config="ca-config.json" -profile=server "connect.json" | cfssljson -bare "certs/connect"

cfssl gencert -ca=certs/ca.pem -ca-key="certs/ca-key.pem" -config="ca-config.json" -profile=server "schemaregistry.json" | cfssljson -bare "certs/schemaregistry"

cfssl gencert -ca=certs/ca.pem -ca-key="certs/ca-key.pem" -config="ca-config.json" -profile=server "ksql.json" | cfssljson -bare "certs/ksql"

## fullchain

cat certs/controlcenter.pem certs/ca.pem | pbcopy
cat certs/connect.pem certs/ca.pem | pbcopy
cat certs/schemaregistry.pem certs/ca.pem | pbcopy
cat certs/ksql.pem certs/ca.pem | pbcopy

### create keystore

cfssl gencert -ca=certs/ca.pem -ca-key="certs/ca-key.pem" -config="ca-config.json" -profile=server "client.json" | cfssljson -bare "certs/client"

openssl pkcs12 -export \
	-in certs/client.pem \
	-inkey certs/client-key.pem \
	-out certs/pkcs.p12 \
	-name client \
	-passout pass:confluent

keytool -importkeystore \
	-deststorepass confluent \
	-destkeypass confluent \
	-destkeystore certs/keystore.jks \
	-deststoretype pkcs12 \
	-srckeystore certs/pkcs.p12 \
	-srcstoretype PKCS12 \
	-srcstorepass confluent

keytool -list -v -keystore certs/keystore.jks -storepass confluent


./bin/kafka-topics --bootstrap-server kafka-0.kafka.operator.svc.cluster.local:9071 --list --command-config ssl_operator.conf


### create truststore

keytool -import \
        -trustcacerts \
        -alias ca \
        -file certs/ca.pem \
	-keystore certs/truststore.jks \
	-deststorepass confluent \
  -deststoretype pkcs12 \
	-noprompt

helm install operator ./confluent-operator -f ./providers/private_mtls.yaml --namespace operator --set operator.enabled=true

helm install zookeeper ./confluent-operator -f ./providers/private_mtls.yaml --namespace operator --set zookeeper.enabled=true

helm install kafka ./confluent-operator -f ./providers/private_mtls.yaml --namespace operator --set kafka.enabled=true