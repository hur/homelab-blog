+++
title = "Integrating Tekton with Harbor"
date = "2023-03-13"

[taxonomies]
tags = ["kubernetes", "tekton", "harbor", "kaniko"]
+++

This post covers the steps I took to get Tekton to correctly push built container images to Harbor in order to automate building and deployment of this blog.
<!-- more -->

This homelab is based off of [khuedoan/homelab](https://homelab.khuedoan.com/), which at the time of this post includes Tekton and Harbor, but Tekton's CI jobs can't push to Harbor. I set out to fix that.

Its important that this setup would be automated. This means that necessary credentials should be automatically provisioned, and that Tekton could automatically authenticate to Harbor using those provisioned credentials. 

### Step 1: Provisioning harbor robot accounts. 

Since I am planning on enabling OIDC authentication for Harbor, I can't create any normal accounts on the Harbor instance. However, Harbor has [robot accounts](https://goharbor.io/docs/2.2.0/administration/robot-accounts/) meant for automated admin tasks, which are perfect for CI.

Harbor provides an API we can use to automate the creation of a robot account.
As I wasn't aware of any better way to approach this, I extended Khue Doan's [hacks script](https://github.com/khuedoan/homelab/blob/d3de308e5418c29e489808cf3d6e15d5fcb6d89c/scripts/hacks) which is executed as part of the initial deployment. When I have more time, I might implement this as a CronJob or find a better way to achieve this in the cluster.

I had implemented auto-generation of Harbor admin account password as part of another change, and the first step in this is to retrieve the harbor endpoint URL and the admin credential from the kubernetes secret.

```go
harbor_host = client.NetworkingV1Api().read_namespaced_ingress(
  'harbor-ingress', 
  'harbor'
).spec.rules[0].host

harbor_pass = base64.b64decode(
  client.CoreV1Api().read_namespaced_secret(
    'harbor-secrets', 
    'harbor'
  ).data['HARBOR_ADMIN_PASSWORD']
).decode("utf-8")

harbor_url = f"https://{harbor_host}"
```
Then, we can perform API requests to provision the account using the `/api/v2.0/robots` endpoint.
Please note that there is another legacy endpoint `/api/v2.0/projects/{project_name_or_id}/robots` which does not work the same way. I wasted some time attempting to use the legacy endpoint as I completely missed the new endpoint. Switching to the new endpoint fixed all of my problems.

This script is idempotent and safe to run in a CronJob. The script first checks if the desired robot account already exists:
```go
resp = requests.get(
  url=f"{harbor_url}/api/v2.0/robots?q=" + requests.utils.quote(f"exact match(name=robot${name})"),
  auth=HTTPBasicAuth('admin', harbor_pass)
)
if resp.status_code != 200:
  print(f"Error checking for existing Harbor robot account for Tekton ({resp.status_code})")
  print(resp.content)
  sys.exit(1)
robots = resp.json()
```
and if not, attempts to create it. Even if the script would attempt to create the account twice, this would not be a problem, the API would simply refuse the second registration and not overwrite any data. 
```go
if not any(robot['name'] == f"robot${name}" for robot in robots):
  // create robot account for tekton
  resp = requests.post(
    url=f"{harbor_url}/api/v2.0/robots",
    headers={
      'Content-Type': 'application/json'
      },
    auth=HTTPBasicAuth('admin', harbor_pass),
    data=json.dumps({
      "name": name,
      "duration": -1,  # no expiry
      "description": "Robot account for tekton pipelines",
      "disable": False,
      "level": "system", 
      "permissions": [
        {
        "namespace": "*",  # TODO: finer grained permissions
        "kind": "project",
        "access": [
          {
            "resource": "repository",
            "action": "push"
          },
          // More permissions...
          ]
        }
      ]
    })
```
If the account is succesfully created, the username and its secret are automatically stored in Vault. 
```go
  // continuing ... 
  )
  if resp.status_code == 201:
    create_vault_secret(
      f"harbor/tekton-robot-account",
        {
          'name': resp.json()['name'],
          'secret': resp.json()['secret']
        }
    )
  else:
    print(f"Error creating Harbor robot account for Tekton ({resp.status_code})")
    print(resp.content)
    sys.exit(1)
```
Now, we have a Harbor robot account whose credentials we can obtain from Vault. 

### Step 2: Correctly mounting the secret and using it in Kaniko
In our Tekton pipelines, we use [Kaniko](https://github.com/GoogleContainerTools/kaniko) to build and push images inside the Kubernetes cluster. Kaniko expects that a Docker `config.json` is mounted at `/kaniko/.docker/config.json` so that it can authenticate to the private registry (in our case, Harbor). This bit took me a while to figure out as we are using an experimental Tekton feature, [workflows](https://github.com/tektoncd/community/blob/main/teps/0098-workflows.md), and this is my first time working with both Tekton and Kaniko.

In our Workflow, we define a workspace that contains the `config.json`, since this is what the Kaniko integration for Tekton expects.
```yaml
  workspaces:
    # ...
    # for kaniko to be able to push to harbor
    - name: harbor-auth 
      secret:
        secretName: tekton-harbor-auth # we will create this secret soon
        items:
          - key: .dockerconfigjson
            path: config.json
```
Then, in the blog's `.ci/master.yaml`, we must refer to this both in `spec.workspaces` as well as `spec.tasks.[].workspaces`, and the workspace must be given the name `dockerconfig` as this is what Kaniko expects the credentials to be in. 
```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: master
spec:
  workspaces:
    # ...
    - name: harbor-auth
  tasks:
    # ...
    - name: docker
      # ...
      workspaces:
        # ...
        - name: dockerconfig
          workspace: harbor-auth
```

Now, we must also add the secret `tekton-harbor-auth` that the workspace obtains the `config.json` contents from. We can use an [ExternalSecret](https://external-secrets.io/v0.7.2/api/externalsecret/) to obtain the credentials from Vault. However, since Kaniko expects a file with Docker's `config.json` format and our credentials are stored in Vault as just the username and password, we must do a bit of templating magic to convert the secret into the desired format.

In the ExternalSecret's spec, we can specify a `Template` which can use the secret's contents and Go templating to create a correct secret of the `kubernetes.io/dockerconfigjson` type.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: tekton-harbor-auth
  namespace: tekton-workflows
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: vault
  target:
    name: tekton-harbor-auth
    template:
      engineVersion: v2
      type: kubernetes.io/dockerconfigjson
      data:
        .dockerconfigjson: |
          {
            "auths": {
              "harbor.atte.cloud": {
                "auth": "{{ printf "%s:%s" (.data | fromYaml).name (.data | fromYaml).secret | b64enc }}"
              }
            }
          }
  data:
  - secretKey: data
    remoteRef:
      key: /harbor/tekton-robot-account
```

With these changes, our pipelines can now authenticate to Harbor to push this blog's container image, which is then picked up by a deployment, resulting in this blog being live.
