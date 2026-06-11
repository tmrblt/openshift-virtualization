### 1. Create users and groups (htpasswd)

#1. Create htpasswd file with admin user
```
htpasswd -c -B -b htpasswd-users.htpasswd admin redhat
```

#2. Add other users to htpasswd
```
htpasswd -B -b htpasswd-users.htpasswd g1-provisioner redhat
htpasswd -B -b htpasswd-users.htpasswd g1-user1 redhat
htpasswd -B -b htpasswd-users.htpasswd g1-user2 redhat

htpasswd -B -b htpasswd-users.htpasswd g2-provisioner redhat
htpasswd -B -b htpasswd-users.htpasswd g2-user1 redhat
htpasswd -B -b htpasswd-users.htpasswd g2-user2 redhat

htpasswd -B -b htpasswd-users.htpasswd g2-provisioner redhat
htpasswd -B -b htpasswd-users.htpasswd g2-user1 redhat
htpasswd -B -b htpasswd-users.htpasswd g2-user2 redhat
```

#3. Check contents of htpasswd file 
```
cat htpasswd-users.htpasswd
```

#4. Create secret from htpasswd file
```
oc create secret generic secret-htpasswd-users --from-file htpasswd=htpasswd-users.htpasswd -n openshift-config
```

#5. Get existing oauth from cluster
```
oc get oauth cluster -o yaml > oauth.yaml
```

#6. Update oauth file for htpasswd
```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
...output omitted...
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: secret-htpasswd-users
    mappingMethod: claim
    name: htpasswd-users
    type: HTPasswd
```

#7. Replace oauth.yaml
```
oc replace -f oauth.yaml
```

#8. Wait for authentication pods restart
```
watch oc get pod -n openshift-authentication
```

#9. Assign cluster-admin role to admin user of htpasswd 
```
oc adm policy add-cluster-role-to-user cluster-admin admin
```

#10. List users and identities
```
oc get users
oc get identity
```

#11. Crate groups
```
oc adm groups new group1
oc adm groups new group2
oc adm groups new group3
```

#12. Add users to groups
```
oc adm groups add-users group1 g1-provisioner
oc adm groups add-users group1 g1-user1
oc adm groups add-users group1 g1-user2

oc adm groups add-users group2 g2-provisioner
oc adm groups add-users group2 g2-user1
oc adm groups add-users group2 g2-user2

oc adm groups add-users group3 g3-provisioner
oc adm groups add-users group3 g3-user1
oc adm groups add-users group3 g3-user2
```

### 2. Cluster Resource Quotas

```
# group1-crq.yaml
apiVersion: quota.openshift.io/v1
kind: ClusterResourceQuota
metadata:
  name: group1-total-quota
spec:
  quota:
    hard:
      cpu: "50"
      memory: 200Gi
      pods: "100"
      count/virtualmachines.kubevirt.io: "10"
      requests.storage: "500Gi"
  selector:
    labels:
      matchLabels:
        local-group: g1-provisioner
```
```
oc create -f group1-crq.yaml
``` 
```
# group2-crq.yaml
apiVersion: quota.openshift.io/v1
kind: ClusterResourceQuota
metadata:
  name: group2-total-quota
spec:
  quota:
    hard:
      cpu: "50"
      memory: 200Gi
      pods: "100"
      count/virtualmachines.kubevirt.io: "10"
      requests.storage: "500Gi"
  selector:
    labels:
      matchLabels:
        local-group: g2-provisioner
```
```
oc create -f group2-crq.yaml
```         
```
# group3-crq.yaml
apiVersion: quota.openshift.io/v1
kind: ClusterResourceQuota
metadata:
  name: group3-total-quota
spec:
  quota:
    hard:
      cpu: "50"
      memory: 200Gi
      pods: "100"
      count/virtualmachines.kubevirt.io: "10"
      requests.storage: "500Gi"
  selector:
    labels:
      matchLabels:
        local-group: g3-provisioner
```
```
oc create -f group3-crq.yaml
```

### 3. Disable Self-Provisioning
#1. Disable self-provisioners 
```
oc patch clusterrolebinding self-provisioners -p '{"subjects": null}'

oc annotate clusterrolebinding self-provisioners rbac.authorization.kubernetes.io/autoupdate="false" --overwrite
```
#2. Check all bindings to verify noone has self-provisioners role
```
oc get clusterrolebinding -o json | jq -r '.items[] | select(.roleRef.name == "self-provisioner") | {binding_name: .metadata.name, subjects: .subjects}'
```

### 4. Set Custom Message
```
oc patch project.config.openshift.io cluster --type=merge -p '{"spec":{"projectRequestMessage":"Please use the Ansible Self-Service Portal to request a new project."}}'
```

### 5. Create provisioner ClusterRole
```
# provisioner-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: provisioner-role
rules:
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["create", "get", "list", "watch", "patch", "update"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["rolebindings"]
  verbs: ["create", "get", "list", "patch", "update", "delete"]
- apiGroups: ["", "kubevirt.io"]
  resources: ["resourcequotas", "limitranges", "virtualmachines", "persistentvolumeclaims"]
  verbs: ["create", "get", "list", "patch", "update", "delete"]
```
```
oc create -f provisioner-role.yaml
```

### 6. Role Assignment
```
oc adm policy add-cluster-role-to-user provisioner-role g1-provisioner
oc adm policy add-cluster-role-to-user provisioner-role g2-provisioner
oc adm policy add-cluster-role-to-user provisioner-role g3-provisioner
```

### 7. Create user-project-admin ClusterRole
```
# user-project-admin.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: user-project-admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list", "watch"] # Denies UPDATE and PATCH to lock labels/annotations
```
```
oc create -f user-project-admin.yaml
```

