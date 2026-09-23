# Creating and Consuming ConfigMap in Kubernetes

This lab demonstrates how to create a Kubernetes **ConfigMap** and consume its configuration data inside a Pod.

The lab covers:

* Creating a ConfigMap using the imperative approach
* Creating a ConfigMap using a declarative YAML manifest
* Verifying the ConfigMap
* Injecting ConfigMap values as environment variables
* Using `envFrom` with `configMapRef`
* Using `env` with `configMapKeyRef`
* Verifying the injected environment variables from inside the Pod

---

## 📌 What is a ConfigMap?

A ConfigMap is a Kubernetes object used to store configuration data as key-value pairs.

It helps keep configuration separate from application/container images. This means application configuration can be changed without modifying the application image itself.

Typical configuration data can include:

* Environment variables
* Application parameters
* Command-line arguments
* Configuration file contents

A ConfigMap can be consumed by a Pod in different ways, including environment variables or mounted files.

---

# 🔄 How ConfigMap Works

A simple way to look at the relationship is:

```text
                ConfigMap
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
    Environment           Volume Mount
     Variables              / Files
          │                   │
          └─────────┬─────────┘
                    ▼
                   Pod
                    │
                    ▼
              Application
```

For this lab, the ConfigMap will be consumed as environment variables.

---

# 🛠️ Imperative vs Declarative Approach

Kubernetes resources such as ConfigMaps can generally be created using two approaches.

### Imperative

The resource is created directly using `kubectl` commands.

This is convenient for:

* Quick testing
* Lab environments
* Temporary configuration
* Experimentation

### Declarative

The desired configuration is defined in a YAML manifest and then applied to the cluster.

This approach is more suitable for production workflows because the configuration can be:

* Version controlled
* Reviewed
* Reused
* Reproduced consistently

Both approaches ultimately create the same Kubernetes resource; the main difference is how the desired state is defined and managed.

---

# 🎯 Lab Objective

Create a ConfigMap named:

```text
db-config
```

with the following configuration values:

```text
MYSQL_ROOT_PASSWORD=abc123
MYSQL_USER=user1
MYSQL_PASSWORD=user1@mydb
```

Then consume these values inside a MySQL Pod named:

```text
my-db
```

---

# 1. Create the ConfigMap — Imperative Approach

The general syntax is:

```bash
kubectl create configmap <config-name> \
  --from-literal=<key>=<value> \
  --from-literal=<key>=<value>
```

For this lab:

```bash
kubectl create configmap db-config \
  --from-literal=MYSQL_ROOT_PASSWORD=abc123 \
  --from-literal=MYSQL_USER=user1 \
  --from-literal=MYSQL_PASSWORD=user1@mydb
```

This creates a ConfigMap named `db-config` containing the specified key-value pairs.

---

# 2. Verify the ConfigMap

List the ConfigMaps:

```bash
kubectl get configmap
```
![configMap](https://github.com/cloudybdone/ConfigMap-In-K8s/blob/main/configmap0001.png)
To inspect the details:

```bash
kubectl describe configmap db-config
```
![ocnfigMap02](https://github.com/cloudybdone/ConfigMap-In-K8s/blob/main/configMap0002.png)
The `describe` command allows us to inspect the configuration data stored inside the ConfigMap.

---

# 3. Create the ConfigMap — Declarative Approach

Create a file named:

```text
config-map.yaml
```

with the following content:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  MYSQL_ROOT_PASSWORD: abc123
  MYSQL_USER: user1
  MYSQL_PASSWORD: user1@mydb
```

Apply the manifest:

```bash
kubectl create -f config-map.yaml
```

This creates the ConfigMap based on the YAML definition.

> **Note:** If the `db-config` ConfigMap was already created using the imperative method, use a separate namespace or delete the existing object before creating the same resource again.

---

# 4. Consume ConfigMap as Environment Variables

Now we will use the ConfigMap inside a MySQL Pod.

The process is straightforward:

```text
Create ConfigMap
       ↓
Reference ConfigMap from Pod
       ↓
Kubernetes injects values
       ↓
Environment variables available inside container
```

Kubernetes provides two approaches for this.

---

# 5. Method 1 — Using `envFrom` with `configMapRef`

This approach imports **all key-value pairs** from the ConfigMap as environment variables.

Create:

```text
pod-definition.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-db
  labels:
    name: my-db
spec:
  containers:
    - name: my-db
      image: mysql
      envFrom:
        - configMapRef:
            name: db-config
```

Here:

```yaml
envFrom:
  - configMapRef:
      name: db-config
```

tells Kubernetes to load all keys from `db-config` as environment variables inside the container.

This is useful when the application needs most or all of the configuration values stored in the ConfigMap.

Apply the Pod:

```bash
kubectl apply -f pod-definition.yaml
```

---

# 6. Method 2 — Using `env` with `configMapKeyRef`

Sometimes the application only needs specific values from a ConfigMap.

In that situation, individual keys can be referenced using `configMapKeyRef`.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-db
  labels:
    name: my-db
spec:
  containers:
    - name: my-db
      image: mysql
      env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: MYSQL_ROOT_PASSWORD

        - name: MYSQL_USER
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: MYSQL_USER

        - name: MYSQL_PASSWORD
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: MYSQL_PASSWORD
```

This method provides more granular control because individual ConfigMap keys can be selected and mapped to environment variables.

---

# 7. Verify the ConfigMap

Check the ConfigMap:

```bash
kubectl get configmap
```

You should see:

```text
NAME        DATA   AGE
db-config   3      ...
```

The `DATA` column indicates the number of data entries stored in the ConfigMap.

---

# 8. Verify the Pod

Check the Pod:

```bash
kubectl get pod
```

Expected state:

```text
NAME    READY   STATUS    RESTARTS   AGE
my-db   1/1     Running   0          ...
```

The exact output will depend on the cluster and Pod startup time.

---

# 9. Verify Environment Variables Inside the Pod

To verify that the ConfigMap values were injected into the container:

```bash
kubectl exec -it my-db -- sh
```

Once inside the container:

```bash
env
```

The environment should contain the ConfigMap values:

```text
MYSQL_ROOT_PASSWORD=abc123
MYSQL_USER=user1
MYSQL_PASSWORD=user1@mydb
```

This confirms that the ConfigMap data has been successfully exposed to the container as environment variables.

---

# 🧠 Key Takeaways

The main idea behind this lab is **separation of configuration from the application image**.

Instead of embedding environment-specific configuration directly into a container image:

```text
Application
     +
Configuration
     ↓
Container Image
```

we can keep the configuration separately:

```text
Container Image
       +
   ConfigMap
       ↓
      Pod
       ↓
 Application
```

This makes the same application image easier to reuse across different environments.

For example:

```text
Development
    ↓
ConfigMap: dev configuration

Testing
    ↓
ConfigMap: test configuration

Production
    ↓
ConfigMap: production configuration
```

The application image can remain unchanged while the environment-specific configuration changes independently.

---

# ⚠️ Important Note

The example in this lab uses values such as:

```text
MYSQL_ROOT_PASSWORD
MYSQL_PASSWORD
```

as ConfigMap data because that is the configuration pattern used by the lab.

In a real production environment, **sensitive credentials should not be stored in a ConfigMap**. Kubernetes provides `Secret` resources for sensitive values.

ConfigMap is better suited for non-sensitive configuration such as:

* Application mode
* Service endpoints
* Feature flags
* Port numbers
* Non-sensitive application settings

---

# 📋 Complete Lab Flow

```text
                    ┌─────────────────┐
                    │   ConfigMap     │
                    │   db-config     │
                    └────────┬────────┘
                             │
                 ┌───────────┴───────────┐
                 │                       │
                 ▼                       ▼
             envFrom              configMapKeyRef
                 │                       │
                 └───────────┬───────────┘
                             ▼
                       ┌───────────┐
                       │  my-db    │
                       │   Pod     │
                       └─────┬─────┘
                             │
                             ▼
                    Environment Variables
```

---

# 🏁 Conclusion

This lab demonstrates how Kubernetes ConfigMaps can be used to keep application configuration separate from container images and make that configuration available to workloads at runtime.

The complete workflow is:

```text
Create ConfigMap
       ↓
Verify ConfigMap
       ↓
Reference from Pod
       ↓
Inject as Environment Variables
       ↓
Verify Inside Container
```

The practical benefit is straightforward: **the application image does not need to change every time environment-specific configuration changes.**

That separation becomes increasingly useful as the same application moves across development, testing, staging, and production environments.
