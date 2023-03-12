+++
title = "Bringing down Gitea by prodding around with secrets"
date = "2023-03-12"

[taxonomies]
tags = ["kubernetes"]
+++

As I was preparing to try to set up OIDC authentication for Harbor with Gitea as the OpenID provider, I decided to require manual approval for Gitea. This involved a config change, which ended up bringing Gitea down.
<!-- more -->

The following change in the Gitea helm chart caused the gitea container to go into `CrashLoopBackOff`.
```diff
diff --git a/platform/gitea/values.yaml b/platform/gitea/values.yaml
index 15dd9fc..85160f4 100644
--- a/platform/gitea/values.yaml
+++ b/platform/gitea/values.yaml
@@ -24,6 +24,8 @@ gitea:
         ROOT_URL: https://git.atte.cloud
       webhook:
         ALLOWED_HOST_LIST: private
+      service:
+        REGISTER_MANUAL_CONFIRM: true
```
Odd, since the change was very small and shouldn't cause a crash by itself. 
First, we check out what's going on in the gitea namespace:
```
atte@vk1 $ kubectl get all -n gitea
NAME                                  READY   STATUS                  RESTARTS        AGE
pod/gitea-0                           0/1     Init:CrashLoopBackOff   6 (3m49s ago)   10m
```
Let's inspect this pod a bit more closely
```
atte@vk1 $ kubectl --namespace gitea describe pod gitea-0
Normal   Started    37m (x4 over 38m)    kubelet            Started container configure-gitea
Warning  BackOff    12m (x113 over 37m)  kubelet            Back-off restarting failed container
```
The logs for gitea-0 were empty. Let's see the logs of the `configure-gitea` container:
```
kubectl --namespace gitea logs gitea-0 -c configure-gitea
Failed to run app with [/usr/local/bin/gitea admin user change-password 
--username gitea_admin --password ]: password is required
```
Looks like the container is getting an empty password for some reason. 

In this lab, Gitea is set up to  obtain its admin username and password from a secret called `gitea-admin-secret` which is managed by [external-secrets](https://external-secrets.io/v0.7.2/) and pulls the secret from a self-hosted [https://www.vaultproject.io/](Vault) instance.

Let's verify that the secret has an empty password using `kubectl get secrets -n gitea gitea-admin-secret -o yaml`.
```yaml
apiVersion: v1
data:
  password: ""
  username: "REDACTED"
immutable: false
kind: Secret
[...]
```
Since external-secrets was not reporting any problems, I decided to look into Vault. 
After investigating in Vault, I realized this was because I had prodded around manually after accidentally creating another secret version. I had deleted the most recent version of the secret which seems to have caused external-secrets to fetch an empty password. Fixed by adding a new version of the secret with some data, and forcing external-secrets to sync the secret with
```
kubectl annotate es gitea-admin-secret force-sync=(date +%s) --overwrite -n gitea
```