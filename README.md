[![Join Slack](https://img.shields.io/badge/Join%20us%20on-Slack-e01563.svg)](https://godaddy-oss-slack.herokuapp.com/) [![Greenkeeper badge](https://badges.greenkeeper.io/godaddy/kubernetes-external-secrets.svg)](https://greenkeeper.io/)
[![js-standard-style](https://img.shields.io/badge/code%20style-standard-brightgreen.svg)](http://standardjs.com)

# 💂 Kubernetes External Secrets

Kubernetes External Secrets allows you to use external secret management systems (*e.g.*, [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)) to securely add secrets in Kubernetes. Read more about the design and motivation for Kubernetes External Secrets on the [GoDaddy Engineering Blog](https://godaddy.github.io/2019/04/16/kubernetes-external-secrets/).

## How it works

The project extends the Kubernetes API by adding a `ExternalSecrets` object using [Custom Resource Definition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) and a controller to implement the behavior of the object itself.

An `ExternalSecret` declares how to fetch the secret data, while the controller converts all `ExternalSecrets` to `Secrets`.
The conversion is completely transparent to `Pods` that can access `Secrets` normally.

## System architecture

![Architecture](architecture.png)

1. `ExternalSecrets` are added in the cluster (e.g., `kubectly apply -f external-secret-example.yml`)
1. Controller fetches `ExternalSecrets` using the Kubernetes API
1. Controller uses `ExternalSecrets` to fetch secret data from external providers (e.g, AWS Secrets Manager)
1. Controller upsert `Secrets`
1. `Pods` can access `Secrets` normally

## How to use it

### Install

To create the necessary resource and install the controller run:

```sh
kubectl apply -f https://raw.githubusercontent.com/godaddy/kubernetes-external-secrets/master/external-secrets.yml
```

This creates all the necessary resources and a `Deployment` in the `kubernetes-external-secrets` namespace.

### Add a secret

Add your secret data to your backend. For example, AWS Secrets Manager:

```
aws secretsmanager create-secret --name hello-service/password --secret-string "1234"
```

and then create a `hello-service-external-secret.yml` file:

```yml
apiVersion: 'kubernetes-client.io/v1'
kind: ExternalSecret
metadata:
  name: hello-service
secretDescriptor:
  backendType: secretsManager
  data:
    - key: hello-service/password
      name: password
```

Save the file and run:

```sh
kubectl apply -f hello-service-external-secret.yml
```

Wait a few minutes and verify that the associated `Secret` has been created:

```sh
kubectl get secret hello-service -o=yaml
```

The `Secret` created by the controller should look like:

```yml
apiVersion: v1
kind: Secret
metadata:
  name: hello-service
type: Opaque
data:
  password: MTIzNA==
```

## Backends

kubernetes-external-secrets supports only AWS Secrets Manager.

### AWS Secrets Manager

kubernetes-external-secrets supports both JSON objects ("Secret
key/value" in the AWS console) or strings ("Plaintext" in the AWS
console). Using JSON objects is useful when you need to atomically
update multiple values. For example, when rotating a client
certificate and private key.

When writing an ExternalSecret for a JSON object you must specify the
properties to use. For example, if we add our hello-service
credentials as a single JSON object:

```
aws secretsmanager create-secret --region us-west-2 --name hello-service/credentials --secret-string '{"username":"admin","password":"1234"}'
```

We can declare which properties we want from hello-service/credentials:

```yml
apiVersion: 'kubernetes-client.io/v1'
kind: ExternalSecret
metadata:
  name: hello-service
secretDescriptor:
  backendType: secretsManager
  data:
    - key: hello-service/credentials
      name: password
      property: password
    - key: hello-service/credentials
      name: username
      property: username
```

## Development

[Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) is a tool that makes it easy to run a Kubernetes cluster locally.

Start minikube and the daemon. This creates the `CustomerResourceDefinition`, and starts to process `ExternalSecrets`:

```sh
minikube start

npm run nodemon
```
