# Sorry Cypress Openshift Configuration

[Sorry cypress](https://sorry-cypress.dev/) is an open-source alternative to cypress dashboard and, this contains the Openshift configuration required to self-host sorry cypress. You can change the configuration values as you wish.

> There is also a [helm chart](https://github.com/sorry-cypress/charts) provided by the sorry cypress team for easier deployment. These configs were made by me as helm was not available.

## Getting started

1. Clone the repo

```bash
git clone git@github.com:akhilmhdh/sorry-cypress-openshift.git
```

2. Change the values described in next section

3. Apply the configuration to openshift via cli or oc ui.

## Dependency Components

These are the various components like db, object storage required for sorry-cypress.

### [Sorry cypress secret](./sorry-cypress-secret.yml)

These are the secret used in the configurations. You can add more secrets like monogodb username, password etc. Currently contains minio username and password.

```bash
oc create -f ./sorry-cypress-secret.yml
```

### [Mongo DB](<(./mongodb.yml)>)

Sorry-cypress requires a mongodb database and the mongodb.yml contains the stateful set and service for the mongodb. You can add `username` and `password` as enviroment variable.

```bash
oc create -f ./mongodb.yml
```

### [MinIO](./minio.yml)

[MinIO](https://min.io/) is used to save cypress test video recording in the openshift cluster itself. You can also choose AWS S3 by changing the driver from [minio to s3](https://docs.sorry-cypress.dev/configuration/director-configuration#remote-storage-configuration). There are mainly two containers running in this configuration by default.

1. MinIO object store -> 9000
2. MinIO dashboard -> 9001

```bash
oc create -f ./minio.yml
```

After the pods are running, login to minio dashboard and create a bucket with name `sorry-cypress` and policy public. This is where the videos and images be saved by sorry-cypress. This can be changed, but corresponding change needs to made in env of sorry cypress director.

## Core components

Sorry-cypress official docker image cannot be used to deploy in Openshift as the docker images expect `root` access, but Openshift doesn't allow root containers. Therefore the given configuration uses images from [quay.io](https://quay.io/akhilmhdh) built by myself in which the image runs as a non-root user. The dashboard and director docker source code are attached for reference.

Sorry-cypress mainly contains three components.

1. Sorry cypress director: Root component that handles parallel test execution and saving it.
2. Sorry cypress api: A graphql api to access cypress projects and test details
3. Sorry cypress dashboard: A dashboard panel that provides a ui to go through projects, test and there recordings

### [Sorry Cypress API](./sorry-cypress-api.yml)

API components doesn't require root access thus can be straight way deployed to openshift. All it needs is mongodb database.

```bash
oc create -f ./sorry-cypress-api.yml
```

### [Sorry Cypress Dashboard](./sorry-cypress-dashboard.yml)

By default the official image requires root access due to nginx enviroment variable injection. The image used here is modified by the official doc on [non-root conversion](https://docs.sorry-cypress.dev/cloud-setup/docker-images/running-sorry-cypress-dashboard-as-non-root-user). The major difference is, [sorry-cypress-dashboard-config](./sorry-cypress-dashboard-config.yml) config map is mounted to the default.conf file instead of modifying it to the image.

With this the env variable required for dashboard are to be change in the config map file. The most important one is `GRAPHQL_SCHEMA_URL` which expects external route to sorry-cypress api.

```bash
oc create -f ./sorry-cypress-dashboard-config.yml
```

Then create the dashboard and its service

```bash
oc create -f ./sorry-cypress-dashboard.yml
```

### [Sorry cypress Director](./sorry-cypress-director.yml)

The director requried root access because the pm2 inside director tries to create a `.pm2` folder at root. To avoid this, the [image is extended](./sorry-cypress-director) and the `./pm2` folder is created before hand and user is converted to non-root `node`.

This configuration requires the minio secrets and also the [director config file](./sorry-cypress-director-config.yml). Change the required values in the config file.

> Minio endpoint and Minio URL both are external IP. Difference is one should be complete URL other one just the domain name. Eg: MinioURL: http://my-minio.com:3000 Minio Endpoint: my-minio.com.

> The http and port are filled in through another env variable `MINIO_PORT`,`MINIO_USESSL`. I did try giving Minio end point as internal address but it was not working. Correct me if I am wrong :smile:. The url is used to build the video link in dashboard.

1. Create the config file

```bash
oc create -f ./sorry-cypress-director-config.yml
```

2. Create the director and its service

```bash
oc create -f ./sorry-cypress-director.yml
```

That's it. Now you have a running instance of sorry cypress in openshift.

## For k8/Openshift 4

All you have to do is adjust the ApiVersion of some components like the stateful set and it will work.

## Acknowledgment

1. [Mayur Deshmukh ](https://github.com/deshmukhmayur)
2. [Sorry Cypress](https://sorry-cypress.dev/)
