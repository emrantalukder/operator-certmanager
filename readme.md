# Confluent Operator mTLS Setup

## CloudFlare's PKI toolkit - Generate Certificates
```sh
brew install cfssl
```

### generate ca cert
cfssl gencert -initca ca-csr.json | cfssljson -bare certs/ca

### create intermediate ca 

cfssl gencert -initca intermediate-ca.json| cfssljson -bare certs/intermediate_ca

### sign intermediate ca

cfssl sign -ca certs/ca.pem -ca-key certs/ca-key.pem -config ca-config.json -profile intermediate_ca certs/intermediate_ca.csr | cfssljson -bare certs/intermediate_ca


cfssl gencert -ca=certs/intermediate_ca.pem -ca-key=certs/intermediate_ca-key.pem -config=ca-config.json -profile=server kafka.json | cfssljson -bare certs/kafka

cfssl gencert -ca=certs/intermediate_ca.pem -ca-key=certs/intermediate_ca-key.pem -config=ca-config.json -profile=server controlcenter.json | cfssljson -bare certs/controlcenter

cfssl gencert -ca=certs/intermediate_ca.pem -ca-key=certs/intermediate_ca-key.pem -config=ca-config.json -profile=server schemaregistry.json | cfssljson -bare certs/schemaregistry

cfssl gencert -ca=certs/intermediate_ca.pem -ca-key=certs/intermediate_ca-key.pem -config=ca-config.json -profile=server connect.json | cfssljson -bare certs/connect

cfssl gencert -ca=certs/intermediate_ca.pem -ca-key=certs/intermediate_ca-key.pem -config=ca-config.json -profile=server ksql.json | cfssljson -bare certs/ksql

cfssl gencert -ca=certs/intermediate_ca.pem -ca-key=certs/intermediate_ca-key.pem -config=ca-config.json -profile=server client.json | cfssljson -bare certs/client

### create keystore and truststore (for client cert)

#### keystore
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

#### truststore
keytool -import \
        -trustcacerts \
        -alias ca \
        -file certs/ca.pem \
	-keystore certs/truststore.jks \
	-deststorepass confluent \
  -deststoretype pkcs12 \
	-noprompt


## deploy

helm install operator ./confluent-operator -f ./providers/mtls.yaml --namespace operator --set operator.enabled=true

helm install zookeeper ./confluent-operator -f ./providers/mtls.yaml --namespace operator --set zookeeper.enabled=true

helm install kafka ./confluent-operator -f ./providers/mtls.yaml --namespace operator --set kafka.enabled=true

helm install schemaregistry ./confluent-operator -f ./providers/mtls.yaml --namespace operator --set schemaregistry.enabled=true

helm install connect ./confluent-operator -f ./providers/mtls.yaml --namespace operator --set connect.enabled=true

helm install ksql ./confluent-operator -f ./providers/mtls.yaml --namespace operator --set ksql.enabled=true

helm install controlcenter ./confluent-operator -f ./providers/mtls.yaml --namespace operator --set controlcenter.enabled=true