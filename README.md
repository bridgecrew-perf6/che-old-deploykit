# eclipse che old deploy kit

Eclipse Che 7.45.0 を helm でデプロイするための手順。


## デプロイ手順

### 証明書作成

#### 証明書作成環境作成

```sh
cd ${WORK_DIR}
mkdir che_demo_ca
cd .\che_demo_ca

Invoke-WebRequest https://raw.githubusercontent.com/mikoto2000/docker-images/2fb7b9f119f65aec12ccc8db364bebeba8201090/openssl/docker-compose.yml -OutFile .\docker-compose.yml
mkdir usr_lib_ssl
Invoke-WebRequest https://raw.githubusercontent.com/mikoto2000/docker-images/2fb7b9f119f65aec12ccc8db364bebeba8201090/openssl/usr_lib_ssl/openssl.cnf -OutFile .\usr_lib_ssl\openssl.cnf
Invoke-WebRequest https://raw.githubusercontent.com/mikoto2000/docker-images/2fb7b9f119f65aec12ccc8db364bebeba8201090/openssl/usr_lib_ssl/v3_ca.txt -OutFile .\usr_lib_ssl\v3_ca.txt
docker-compose run --rm openssl bash
```

`che_demo_ca` をどこかで管理しましょう。


#### 証明書作成

```sh
# 必要な情報整理

## ドメイン定義
DOMAIN="FIXME!!!"

## SAN 設定作成
echo "subjectAltName = @alt_names" >> ./usr_lib_ssl/v3.txt
echo "" >> ./usr_lib_ssl/v3.txt
echo "[ alt_names ]" >> ./usr_lib_ssl/v3.txt
echo "DNS.1 = "'*'".${DOMAIN}" >> ./usr_lib_ssl/v3.txt

## 各種生成物定義
CA_KEY_FILE="/ca/private/cakey.pem"
CA_CSR_FILE="/ca/ca.csr"
CA_CRT_FILE="/ca/cacert.pem"

CA_COUNTRY="JP"
CA_STATE="KEN"
CA_LOCALITY="KU"
CA_ORGANIZATION="KOJIN"
CA_ORGANIZATION_UNIT="NONE"
CA_COMMON="${DOMAIN}"
CA_EMAIL="mikoto2000@gmail.com"


# CA 初期化
cd /ca

## 各種ディレクトリ作成
mkdir cert private crl newcerts
chmod 700 private

## シリアル初期化
echo "01" | tee -a serial

## crlnumber 初期化
echo "01" | tee -a crlnumber

## インデックス初期化
touch index.txt
echo "unique_subjec = yes" > index.txt.attr


# CA 鍵と CA 証明書作成

## CA 用秘密鍵作成
openssl genrsa -out ${CA_KEY_FILE} 2048

## CA 証明書要求作成
CA_SUBJECT="/C=${CA_COUNTRY}/ST=${CA_STATE}/L=${CA_LOCALITY}/O=${CA_ORGANIZATION}/OU=${CA_ORGANIZATION_UNIT}/CN=${CA_COMMON}"
openssl req -new -key ${CA_KEY_FILE} -out ${CA_CSR_FILE} -subj "${CA_SUBJECT}"

## CA 証明書要求確認
openssl req -noout -text -in ${CA_CSR_FILE}

## CA 証明書作成
openssl x509 -days 365 -req -in ${CA_CSR_FILE} -signkey ${CA_KEY_FILE} -out ${CA_CRT_FILE} -extfile /usr/lib/ssl/v3_ca.txt

## CA 証明書確認
openssl x509 -text -noout -in ${CA_CRT_FILE}

## CA 証明書を out ディレクトリへコピー
cp ${CA_CRT_FILE} /client/ca.crt


# サーバー証明書発行
SERVER_COMMON="${DOMAIN}"
SERVER_KEY_FILE="/client/che-${SERVER_COMMON}.key"
SERVER_CSR_FILE="/client/che-${SERVER_COMMON}.csr"
SERVER_CRT_FILE="/client/che-${SERVER_COMMON}.crt"

SERVER_COUNTRY="JP"
SERVER_STATE="KEN"
SERVER_LOCALITY="KU"
SERVER_ORGANIZATION="KOJIN"
SERVER_ORGANIZATION_UNIT="NONE"
SERVER_COMMON="che-${DOMAIN}"
SERVER_EMAIL="mikoto2000@gmail.com"

## サーバー秘密鍵作成
openssl genrsa -out ${SERVER_KEY_FILE} 2048

## サーバー証明書要求作成
SERVER_SUBJECT="/C=${SERVER_COUNTRY}/ST=${SERVER_STATE}/L=${SERVER_LOCALITY}/O=${SERVER_ORGANIZATION}/OU=${SERVER_ORGANIZATION_UNIT}/CN=${SERVER_COMMON}"
openssl req -new -key ${SERVER_KEY_FILE} -out ${SERVER_CSR_FILE} -subj "${SERVER_SUBJECT}"

## クライアント証明書要求確認
openssl req -noout -text -in ${SERVER_CSR_FILE}


# サーバー証明書作成
cd /client
yes | openssl ca -out ${SERVER_CRT_FILE} -extfile /usr/lib/ssl/v3.txt -infiles ${SERVER_CSR_FILE}
```

### Kubernetes へ証明書を設定


```sh
kubectl create namespace che
kubectl create secret generic self-signed-certificate --from-file=ca.crt -n che
kubectl create secret generic che-tls --from-file=ca.crt=ca.crt --from-file=tls.key=che-${DOMAIN}.key --from-file=tls.crt=che-${DOMAIN}.crt -n che
```

### Che デプロイ

```sh
cd helm/che
helm dependency update
helm upgrade --install che --namespace che --create-namespace --set global.ingressDomain=${DOMAIN} ./
```

