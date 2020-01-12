# DEX configuration

The dex deployment is available at `dex.192.168.1.110.xip.io:32000`

Set the TLS secrets.

**Remember to Update the CN section before generate your clients (from
`dex.example.com` to something else)**.

Here your `gencert.sh` script: https://github.com/dexidp/dex/blob/master/examples/k8s/gencert.sh

## Bring up your cluster

In this example we will use kind

```sh
kind create cluster --config kind.yaml
```

## Prepare Dex

```sh
kubectl create secret tls dex.example.com.tls --cert=ssl/cert.pem --key=ssl/key.pem
```

Create the Github secret

```sh
export GITHUB_CLIENT_ID=something
export GITHUB_CLIENT_SECRET=other_secrets

kubectl create secret \
generic github-client \
--from-literal=client-id=$GITHUB_CLIENT_ID \
--from-literal=client-secret=$GITHUB_CLIENT_SECRET
```

Now we are ready to deploy Dex

```sh
kubectl apply -f dex.yaml
NAME                   READY   STATUS    RESTARTS   AGE
dex-77bcfc5588-8rvhd   1/1     Running   0          3h8m
dex-77bcfc5588-njb46   1/1     Running   0          3h8m
dex-77bcfc5588-wdpwr   1/1     Running   0          3h8m
```

## Login from the CLI

First of all you need the `oidc-login` plugin
(https://github.com/int128/kubelogin) (grab the binary and put it into the bin
folder)

Then update your `~/.kube/config` configuration file with a new user (_users
section_)

```yaml
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=ISSUER_URL
      - --oidc-client-id=CLIENT_ID
      - --oidc-client-secret=OIDC_SECRET
      - --certificate-authority=CA_ABSOLUTE_PATH
      - --oidc-extra-scope=email
      - --oidc-extra-scope=profile
      - --oidc-extra-scope=groups
      command: kubectl
      env: null
```

Then set this new user as the current user

```sh
kubectl config set-context --user oidc --current
```

And try to use the cluster

```sh
kubectl get pods
Open http://localhost:8000 for authentication

NAME                   READY   STATUS    RESTARTS   AGE
dex-77bcfc5588-8rvhd   1/1     Running   0          3h8m
dex-77bcfc5588-njb46   1/1     Running   0          3h8m
dex-77bcfc5588-wdpwr   1/1     Running   0          3h8m
```

or try to get our deployments

```sh
kubectl get deployments
Error from server (Forbidden): deployments.apps is forbidden: User "walter.dalmut@gmail.com" cannot list resource "deployments" in API group "apps" in the namespace "default"
```

## Login App

The login app is exposed at: `https://loginapp.192.168.1.110.xip.io:5555`

```sh
docker run \
--add-host="dex.192.168.1.110.xip.io:192.168.1.110" \
-p 5555:5555 \
-v `pwd`/ssl/:/ssl \
-v `pwd`/config-loginapp-docker.yaml:/config-loginapp-docker.yaml \
quay.io/fydrah/loginapp:2.7.0 serve /config-loginapp-docker.yaml
```

---

Setup reference: https://github.com/dexidp/dex/blob/master/Documentation/kubernetes.md#configuring-the-openid-connect-plugin

Here the project: https://github.com/fydrah/loginapp

## Play with RBAC

Just apply Roles and RoleBindings, for example

```sh
kubectl apply -f examples/role.yaml
```

then try to get deployments or things outside the `default` namespace

```sh
kubectl get deployments --as=walter.dalmut@gmail.com
Error from server (Forbidden): deployments.apps is forbidden: User "walter.dalmut@gmail.com" cannot list resource "deployments" in API group "apps" in the namespace "default"
```

But if you try with pods everything works fine

```sh
kubectl get pods --as=walter.dalmut@gmail.com
NAME                   READY   STATUS    RESTARTS   AGE
dex-77bcfc5588-8rvhd   1/1     Running   0          3h8m
dex-77bcfc5588-njb46   1/1     Running   0          3h8m
dex-77bcfc5588-wdpwr   1/1     Running   0          3h8m
```
