# Install Vault in Minikube

Running Vault on Kubernetes is generally the same as running it anywhere else. As a container orchestration engine, Kubernetes eases some of the operational burdens, and Helm charts provide the benefit of a refined interface when it comes to deploying Vault in various modes.

In this tutorial, you will set up Vault and its dependencies with a Helm chart. 

## Create a Vault lab directory

```bash
mkdir ~/vault_labs
cd $_
```



## Install Helm CLI

The `helm` command is used to install, manage, upgrade, and rollback applications on Kubernetes clusters. 

To install it, run: 

```bash
export VERIFY_CHECKSUM=false
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```



Confirm it was installed 

```bash
helm version
```





## Install the Vault Helm chart

Vault manages secrets for your applications, users, and systems. For this demonstration, Vault can be run in development mode to automatically handle initialization, unsealing, and setup of a KV secrets engine.

1. Add the HashiCorp Helm repository.

   ```shell-session
   helm repo add hashicorp https://helm.releases.hashicorp.com
   ```

   

2. Update all the repositories to ensure `helm` is aware of the latest versions.

   ```shell-session
   helm repo update
   ```

   Output:

   ```
   Hang tight while we grab the latest from your chart repositories...
   ...Successfully got an update from the "secrets-store-csi-driver" chart repository
   ...Successfully got an update from the "hashicorp" chart repository
   Update Complete. ⎈Happy Helming!⎈
   ```

   

3. To verify, search repositories for `vault` in charts.

   ```shell-session
   helm search repo hashicorp/vault
   ```

   Output:

   ```
   NAME            CHART VERSION   APP VERSION DESCRIPTION
   hashicorp/vault 0.20.0          1.10.3      Official HashiCorp Vault Chart
   ```

   

4. Install the latest version of the Vault Helm chart with Integrated Storage.

   Run the following command to create a file named `helm-vault-raft-values.yml` :

   ```shell-session
   cat > helm-vault-raft-values.yml <<EOF
   server:
     affinity: ""
     ha:
       enabled: true
       raft: 
         enabled: true
   EOF
   ```

   

   Run this command:

   ```shell-session
   helm install vault hashicorp/vault --values helm-vault-raft-values.yml
   ```

   

   **Example output:**

   ```plaintext
   NAME: vault
   LAST DEPLOYED: Wed May 18 20:19:15 2022
   NAMESPACE: default
   STATUS: deployed
   REVISION: 1
   NOTES:
   Thank you for installing HashiCorp Vault!
   
   Now that you have deployed Vault, you should look over the docs on using
   Vault with Kubernetes available here:
   
   https://www.vaultproject.io/docs/
   
   Your release is named vault. To learn more about the release, try:
   
     $ helm status vault
     $ helm get manifest vault
   ```

   This creates three Vault server instances with an Integrated Storage (Raft) backend.

5. Display all the pods within the default namespace.

   ```shell-session
   kubectl get pods
   ```

   Output

   ```
   NAME                                    READY   STATUS    RESTARTS   AGE
   vault-0                                 0/1     Running   0          2m12s
   vault-1                                 0/1     Running   0          2m12s
   vault-2                                 0/1     Running   0          2m12s
   vault-agent-injector-56b65c5cd4-k7lbt   1/1     Running   0          2m13s
   ```

   

6. Initialize `vault-0` with one key share and one key threshold.

   ```shell-session
   kubectl exec vault-0 -- vault operator init \
       -key-shares=1 \
       -key-threshold=1 \
       -format=json > cluster-keys.json
   ```

   

   The [`operator init`](https://developer.hashicorp.com/vault/docs/commands/operator/init) command generates a root key that it disassembles into key shares `-key-shares=1` and then sets the number of key shares required to unseal Vault `-key-threshold=1`. These key shares are written to the output as unseal keys in JSON format `-format=json`. Here, the output is redirected to a file named `cluster-keys.json`.

   

7. Display the unseal key found in `cluster-keys.json`.

   ```shell-session
   jq -r ".unseal_keys_b64[]" cluster-keys.json
   ```

   

   **Insecure operation**

   Do not run an unsealed Vault in production with a single key share and a single key threshold. This approach is only used here to simplify the unsealing process for this demonstration.

8. Create a variable named `VAULT_UNSEAL_KEY` to capture the Vault unseal key.

   ```shell-session
   VAULT_UNSEAL_KEY=$(jq -r ".unseal_keys_b64[]" cluster-keys.json)
   ```

   

   After initialization, Vault is configured to know where and how to access the storage but does not know how to decrypt any of it. Unsealing is the process of constructing the root key necessary to read the decryption key to decrypt the data, allowing access to the Vault.

9. Unseal Vault running on the `vault-0` pod.

   ```shell-session
   kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
   ```

   

   **Insecure operation**

   Providing the unseal key with the command writes the key to your shell's history. This approach is only used here to simplify the unsealing process for this demonstration.

   The `operator unseal` command reports that Vault is initialized and unsealed.

   **Example output:**

   ```plaintext
   Key                     Value
   ---                     -----
   Seal Type               shamir
   Initialized             true
   Sealed                  false
   Total Shares            1
   Threshold               1
   Version                 1.10.3
   Storage Type            raft
   Cluster Name            vault-cluster-3e905d5c
   Cluster ID              74f3d9dc-27ce-7d9e-4382-a8fe5f670a6f
   HA Enabled              true
   HA Cluster              n/a
   HA Mode                 standby
   Active Node Address     <none>
   Raft Committed Index    31
   Raft Applied Index      31
   ```

   The Vault server is initialized and unsealed.

10. Join the `vault-1` pod to the Raft cluster.

    ```shell-session
    kubectl exec -ti vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
    ```

    

11. Join the `vault-2` pod to the Raft cluster.

    ```shell-session
    kubectl exec -ti vault-2 -- vault operator raft join http://vault-0.vault-internal:8200
    ```

    

12. Use the unseal key from above to unseal vault-1.

    ```shell-session
    kubectl exec -ti vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY
    Unseal Key (will be hidden):
    Key                Value
    ---                -----
    Seal Type          shamir
    Initialized        true
    Sealed             true
    Total Shares       1
    Threshold          1
    Unseal Progress    0/1
    Unseal Nonce       n/a
    Version            1.10.3
    Storage Type       raft
    HA Enabled         true
    ```

    

13. Use the unseal key from above to unseal vault-2.

    ```shell-session
    kubectl exec -ti vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY
    ```

## Set a secret in Vault

The web application that you deploy expects Vault to store a username and password at the path `secret/webapp/config`. To create this secret requires you to log in with the root token, enable the key-value secret engine, and store a secret username and password at that defined path.

Vault generated an initial root token when it was initialized.

1. Display the *root token* found in `cluster-keys.json`.

   ```shell-session
   jq -r ".root_token" cluster-keys.json
   ```

   Example output: 

   ```
   hvs.HJmsajgGlWPTx6YNHoUljuOO
   ```

   

2. Start an interactive shell session on the `vault-0` pod.

   ```shell-session
   kubectl exec --stdin=true --tty=true vault-0 -- /bin/sh
   ```

   

   Your system prompt is replaced with a new prompt `/ $`.

   Vault is now ready for you to log in with the initial root token.

   Vault requires clients to authenticate first before it allows any further actions. An unsealed Vault starts with the Token Auth Method enabled and generates an initial root token.

3. Login with the *root token* when prompted.

   ```shell-session
   vault login
   Token (will be hidden):
   ```

   Example output:

   ```
   Success! You are now authenticated. The token information displayed below
   is already stored in the token helper. You do NOT need to run "vault login"
   again. Future Vault requests will automatically use this token.
   
   Key                  Value
   ---                  -----
   token                hvs.HJmsajgGlWPTx6YNHoUljuOO
   token_accessor       JVsMJHVu6rTWbPLlYmWQTq1R
   token_duration       ∞
   token_renewable      false
   token_policies       ["root"]
   identity_policies    []
   policies             ["root"]
   ```

   

4. Enable an instance of the kv-v2 secrets engine at the path `secret`.

   ```shell-session
   vault secrets enable -path=secret kv-v2
   ```

   

5. Create a secret at path `secret/webapp/config` with a `username` and `password`.

   ```shell-session
   vault kv put secret/webapp/config username="static-user" password="static-password"
   ```

   Output:

   ```
   ====== Secret Path ======
   secret/data/webapp/config
   
   ======= Metadata =======
   Key                Value
   ---                -----
   created_time       2022-06-07T05:15:19.402740412Z
   custom_metadata    <nil>
   deletion_time      n/a
   destroyed          false
   version            1
   ```

   

6. Verify that the secret is defined at the path `secret/webapp/config`.

   ```shell-session
   vault kv get secret/webapp/config
   ```

   Output:

   ```
   ====== Secret Path ======
   secret/data/webapp/config
   
   ======= Metadata =======
   Key                Value
   ---                -----
   created_time       2022-06-07T05:15:19.402740412Z
   custom_metadata    <nil>
   deletion_time      n/a
   destroyed          false
   version            1
   
   ====== Data ======
   Key         Value
   ---         -----
   password    static-password
   username    static-user
   ```

   

   You successfully created the secret for the web application.

7. Exit the `vault-0` pod.

   ```shell-session
   exit
   ```

   

## Configure Kubernetes authentication

The initial root token is a privileged user that can perform any operation at any path. The web application only requires the ability to read secrets defined at a single path. This application should authenticate and be granted a token with limited access.

We recommend that root tokens be used only for the initial authentication method and policies setup. Afterwards, they should be revoked. This tutorial does not show you how to revoke the root token.

Vault provides a Kubernetes authentication method that enables clients to authenticate with a Kubernetes Service Account Token.

1. Start an interactive shell session on the `vault-0` pod.

   ```shell-session
   kubectl exec --stdin=true --tty=true vault-0 -- /bin/sh
   ```

   Your system prompt is replaced with a new prompt `/ $`.

   

2. Enable the Kubernetes authentication method.

   ```shell-session
   vault auth enable kubernetes
   ```

   

   Vault accepts this service token from any client within the Kubernetes cluster. During authentication, Vault verifies that the service account token is valid by querying a configured Kubernetes endpoint.

3. Configure the Kubernetes authentication method to use the location of the Kubernetes API.

   

   ```shell-session
   vault write auth/kubernetes/config \
       kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
   ```

   

   Successful output from this command resembles this example:

   ```plaintext
   Success! Data written to: auth/kubernetes/config
   ```

   The environment variable `KUBERNETES_PORT_443_TCP_ADDR` is defined and references the internal network address of the Kubernetes host.

   For a client to access the secret data defined at `secret/webapp/config`, it requires that the read capability be granted for the path `secret/data/webapp/config`. This is an example of a policy. A policy defines a set of capabilities.

4. Write out the policy named `webapp` that enables the `read` capability for secrets at path `secret/data/webapp/config`.

   ```shell-session
   vault policy write webapp - <<EOF
   path "secret/data/webapp/config" {
     capabilities = ["read"]
   }
   EOF
   ```

   

   Successful output from this command resembles this example:

   ```plaintext
   Success! Uploaded policy: webapp
   ```

   Define an auth method role that uses the `webapp` policy. A role binds policies and environment parameters together to create a login for the web application.

5. Create a Kubernetes authentication role named `webapp that connects the Kubernetes service account name and `webapp` policy.

   ```shell-session
   vault write auth/kubernetes/role/webapp \
           bound_service_account_names=vault \
           bound_service_account_namespaces=default \
           policies=webapp \
           ttl=24h
   ```

   

   Successful output from this command resembles this example:

   ```plaintext
   Success! Data written to: auth/kubernetes/role/webapp
   ```

   The role connects the Kubernetes service account, `vault`, and namespace, `default`, with the Vault policy, `webapp`. The tokens returned after authentication are valid for 24 hours.

6. Exit the `vault-0` pod.

   ```shell-session
   exit
   ```

   

## Launch a web application

We've created a web application, published it to DockerHub, and created a Kubernetes deployment to run the application in your existing cluster. The example web application performs the single function of listening for HTTP requests. It reads the Kubernetes service token during a request, logs into Vault, and then requests the secret.

1. Use your preferred text editor to create `deployment-01-webapp.yml`.

   

   deployment-01-webapp.yml

   ```yaml
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: webapp
     labels:
       app: webapp
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: webapp
     template:
       metadata:
         labels:
           app: webapp
       spec:
         serviceAccountName: vault
         containers:
           - name: app
             image: hashieducation/simple-vault-client:latest
             imagePullPolicy: Always
             env:
               - name: VAULT_ADDR
                 value: 'http://vault:8200'
               - name: JWT_PATH
                 value: '/var/run/secrets/kubernetes.io/serviceaccount/token'
               - name: SERVICE_PORT
                 value: '8080'
   ```

   The web application deployment defines a list of environment variables.

   - `JWT_PATH` sets the path of the JSON web token (JWT) issued by Kubernetes. This token is used by the web application to authenticate with Vault.
   - `VAULT_ADDR` sets the address of the Vault service. The Helm chart defined a Kubernetes service named `vault` that forwards requests to its endpoints (i.e. The pods named `vault-0`, `vault-1`, and `vault-2`).
   - `SERVICE_PORT` sets the port that the service listens for incoming HTTP requests.

2. Deploy the webapp in Kubernetes by applying the file `deployment-01-webapp.yml`.

   ```shell-session
   kubectl apply --filename deployment-01-webapp.yml
   ```

   

   The `webapp` runs as a pod within the default namespace.

3. Get all the pods within the default namespace.

   ```shell-session
   kubectl get pods
   ```

   Output:

   ```
   NAME                                    READY   STATUS              RESTARTS   AGE
   vault-0                                 1/1     Running             0          15m
   vault-1                                 1/1     Running             0          15m
   vault-2                                 1/1     Running             0          15m
   vault-agent-injector-659b4488df-4hhpz   1/1     Running             0          15m
   webapp-784b8d9979-vn57d                 0/1     ContainerCreating   0          47s
   ```

   

   The `webapp` pod is displayed here as the pod prefixed with `webapp`.

   The deployment of the service requires retrieving the web application container from Docker Hub. This displays the STATUS of `ContainerCreating`. The pod reports that it is not ready (`0/1`).

   Wait until the `webapp` pod is running and ready (`1/1`).

   The pod runs an HTTP service that is listening on port 8080.

4. In **another terminal**, port forward all requests made to `http://localhost:8080` to the `webapp` pod on port `8080`.

   ```shell-session
   kubectl port-forward \
       $(kubectl get pod -l app=webapp -o jsonpath="{.items[0].metadata.name}") \
       8080:8080
   ```

   

5. In the original terminal, perform a `curl` request at `http://localhost:8080`.

   ```shell-session
   curl http://localhost:8080
   ```

   

   ![Web application showing username and password secret](https://developer.hashicorp.com/_next/image?url=https%3A%2F%2Fcontent.hashicorp.com%2Fapi%2Fassets%3Fproduct%3Dtutorials%26version%3Dmain%26asset%3Dpublic%252Fimg%252Fvault-k8s%252Fminikube-vault-helm-webapp.png%26width%3D722%26height%3D182&w=1920&q=75)

   The web application running on port 8080 in the webapp pod:

   - authenticates with the Kubernetes service account token
   - receives a Vault token with the read capability at the `secret/data/webapp/config` path
   - retrieves the secrets from `secret/data/webapp/config` path
   - displays the secrets as JSON

## Congrats! 



## 