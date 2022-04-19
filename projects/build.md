# ビルド手順

[che-buildkit](https://github.com/mikoto2000/docker-images/tree/master/che-buildkit) を利用する。

```sh
docker compose run -it --rm che-buildkit
```


# 各イメージを amd64, arm64 用にビルドしてプッシュ

## コンテナレジストリにログイン

ビルドしたものをそのまま push できるようにログインしておく。

```sh
docker login
```

## devfile-registry

### devfile-registry のビルド・プッシュ

```sh
./build.sh -t 7.45.0 -o mikoto2000 -r docker.io
```


## plugin-registry

### che-plugin-registry のビルド

```sh
./build.sh -t 7.45.0 -o mikoto2000 -r docker.io
```

## che-plugin-broker

### Docker イメージのビルド

```sh
# che-plugin-artifacts-broker
docker buildx build --platform linux/amd64,linux/arm64 --push -t mikoto2000/che-plugin-artifacts-broker:v3.4.0 -f build/artifacts/Dockerfile .

# che-plugin-metadata-broker
docker buildx build --platform linux/amd64,linux/arm64 --push -t mikoto2000/che-plugin-metadata-broker:v3.4.0 -f build/metadata/Dockerfile .
```

## che-server

### Maven プロジェクトのビルド

`Che Multiuser :: PostgreSQL Tck` で失敗するため、動作に必要なモジュールを別途個別にビルドしている。

「これで十分かよくわからないけど動いているからヨシ！」という状態。

```sh
mvn -B clean install -U -Pintegration -Dmaven.test.skip=true -Dfindbugs.skip=true -Dskip-validate-sources
cd infrastructures
mvn -B install -U -Pintegration -Dmaven.test.skip=true -Dfindbugs.skip=true -Dskip-validate-sources
cd ../assembly
mvn -B install -U -Pintegration -Dmaven.test.skip=true -Dfindbugs.skip=true -Dskip-validate-sources
```

### Docker イメージのビルド

```sh
cd che-server/dockerfiles/
./build.sh --tag:7.45.0 --organization:mikoto2000 endpoint-watcher che
```


## postgres

### s2i-base-container のビルド(根本のイメージ)

```sh
docker buildx build --platform linux/amd64,linux/arm64 --push -t mikoto2000/s2i-core-centos7:arm64 -f core/Dockerfile .
```

### postgresql-13-centos7 のビルド(che-postgres が直接依存しているイメージ)

```sh
cd postgresql-container-for-arm/13
docker buildx build --platform linux/amd64,linux/arm64 --push -t mikoto2000/postgresql-13-centos7:arm64 .
```


### che-postgres のビルド

che-server 内に Dockerfile があるので、それを利用してビルドする。

```sh
cd che-server/dockerfiles/postgres
./build.sh --tag:7.45.0 --organization:mikoto2000
```


## keycloak

### keycloak のビルド(che-keycloak が依存しているイメージ)

```sh
cd keycloak-containers/server
docker buildx build --platform linux/amd64,linux/arm64 -t mikoto2000/keycloak:arm64 .
```

### che-keycloak のビルド

che-server 内に Dockerfile があるので、それを利用してビルド。

```sh
cd $WORKDIR/che-server/dockerfiles/keycloak
./build.sh --tag:7.45.0 --organization:mikoto2000
```


## che-dashboard

```sh
docker buildx build --platform linux/amd64,linux/arm64 --push -f build/dockerfiles/Dockerfile -t mikoto2000/che-dashboard:7.45.0 .
```

