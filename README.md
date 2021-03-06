# mysql-operator

## Overview

This guide is broken into 2 stages. 

1. build a mysql operator
2. build a wordpress operator

## The approach

We are going to start with a simple hello world example. And incrementally modify it into a fully functional mysql operator. 


We will start with implementing quick+dirty techniques, but then gradually improve and build on that to incorporate best practice and achieve level 1 maturity. 





mysql kubernetes operator built using the operator-sdk

```
brew install operator-sdk
```

```
$ operator-sdk version
operator-sdk version: "v0.15.1", commit: "e35ec7b722ba095e6438f63fafb9e7326870b486", go version: "go1.13.6 darwin/amd64"
```

```
operator-sdk new mysql-operator --repo=github.com/Sher-Chowdhury/mysql-operator
cd mysql-operator
```


```
operator-sdk add api --api-version=cache.codingbee.net/v1alpha1 --kind=MySQL  # kind needs to start with uppercase
```

This would have created a crd and a sample cr file, that you can try deploying at this stage. They look like this:

```
$ cat deploy/crds/cache.codingbee.net_mysqls_crd.yaml 
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: mysqls.cache.codingbee.net
spec:
  group: cache.codingbee.net
  names:
    kind: MySQL
    listKind: MySQLList
    plural: mysqls
    singular: mysql
  scope: Namespaced
  subresources:
    status: {}
  validation:
    openAPIV3Schema:
      description: MySQL is the Schema for the mysqls API
      properties:
        apiVersion:
          description: 'APIVersion defines the versioned schema of this representation
            of an object. Servers should convert recognized schemas to the latest
            internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
          type: string
        kind:
          description: 'Kind is a string value representing the REST resource this
            object represents. Servers may infer this from the endpoint the client
            submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
          type: string
        metadata:
          type: object
        spec:
          description: MySQLSpec defines the desired state of MySQL
          type: object
        status:
          description: MySQLStatus defines the observed state of MySQL
          type: object
      type: object
  version: v1alpha1
  versions:
  - name: v1alpha1
    served: true
    storage: true
```

NOTE: You must never make changes to the crd file directly. Intead you should edit it by first editing the mysql_types.go file and then run the `openshift generate k8s` and `openshift generate crds` commands. 

and the sample custom resource (cr):

```
$ cat deploy/crds/cache.codingbee.net_v1alpha1_mysql_cr.yaml
apiVersion: cache.codingbee.net/v1alpha1
kind: MySQL
metadata:
  name: example-mysql
spec:
  # Add fields here
  size: 3
```

Added custom settings to pkg/apis/cache/v1alpha1/mysql_types.go - https://github.com/Sher-Chowdhury/mysql-operator/commit/a9ae8d85f8ddda4e3c6d7b3713f1dee03b5fc5f5#

Note: you can never add any comments to this file, since all the comments have speacial meaning which are then used for updating the crd file. 



Also in particular notice this json-bit on this line: https://github.com/Sher-Chowdhury/mysql-operator/commit/a9ae8d85f8ddda4e3c6d7b3713f1dee03b5fc5f5#diff-bbc388b9f979f725f3962a950d7b75b4R26

This json-bit is used to to specify json related metadata - 
https://stackoverflow.com/questions/10858787/what-are-the-uses-for-tags-in-go
https://www.sohamkamani.com/blog/golang/2018-07-19-golang-omitempty/




For these changes to get propagated to the rest of the repo, run:

```
operator-sdk generate k8s
```

here's the resulting change - https://github.com/Sher-Chowdhury/mysql-operator/commit/10250ab5ee20d87a27f67dea6baff73600862f35

Next run the following to get the changes reflected in the crd file:

```
operator-sdk generate crds
```

Here's the resulting change - https://github.com/Sher-Chowdhury/mysql-operator/commit/551f703ecd5315c592081ab4b86e34c25fe81f44

Now let's create our operator's controller:

```
operator-sdk add controller --api-version=cache.codingbee.net/v1alpha1 --kind=MySQL
```

This ends up creating the file `pkg/controller/add_mysql.go` and the folder `pkg/controller/mysql/` along with all it's content - https://github.com/Sher-Chowdhury/mysql-operator/commit/a29ecf7c5247b448d47e68e412bfc9877199df9e


Next update controller to make use of the mysql env vars - https://github.com/Sher-Chowdhury/mysql-operator/blob/6e4610c2931bb7ff5dfb140b3a8b8feaec484fe7/pkg/controller/mysql/mysql_controller.go#L150-L166 and 
https://github.com/Sher-Chowdhury/mysql-operator/blob/6e4610c2931bb7ff5dfb140b3a8b8feaec484fe7/pkg/controller/mysql/mysql_controller.go#L137



Now deploy the crd (you can also deploy the example cr too if you want too):

```
kubectl apply -f deploy/crds/cache.codingbee.net_mysqls_crd.yaml
```

The above is bit like creating a new table for storing mysql cr data in etcd. Let's check out etcd now has this crd:

```
$ kubectl get customresourcedefinitions                      
NAME                         CREATED AT
mysqls.cache.codingbee.net   2020-02-12T22:30:00Z
```



After that you can list your mysql instances:

```
$ kubectl get mysql
No resources found in default namespace.
```

Now you can 

```
$ kubectl apply -f deploy/crds/my-mysql-db-cr.yaml  

mysql.cache.codingbee.net/my-mysql-db created
```

This simply creates a new entry in the new table created inside etcd. 

```
$ kubectl get mysql                                                      
NAME          AGE
my-mysql-db   22s


$ kubectl get mysql my-mysql-db -o yaml
apiVersion: cache.codingbee.net/v1alpha1
kind: MySQL
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"cache.codingbee.net/v1alpha1","kind":"MySQL","metadata":{"annotations":{},"name":"my-mysql-db","namespace":"default"},"spec":{"environment":{"mysql_database":"wordpressDB","mysql_password":"wpPassword","mysql_root_password":"wpAdminPassword","mysql_user":"wpuser"}}}
  creationTimestamp: "2020-02-15T13:36:06Z"
  generation: 1
  name: my-mysql-db
  namespace: default
  resourceVersion: "154981"
  selfLink: /apis/cache.codingbee.net/v1alpha1/namespaces/default/mysqls/my-mysql-db
  uid: 9b3601a6-de87-4f0c-b849-5cd385472ee4
spec:
  environment:
    mysql_database: wordpressDB
    mysql_password: wpPassword
    mysql_root_password: wpAdminPassword
    mysql_user: wpuser
```


Update the controller to use the mysql image, rather than the busybox image - https://github.com/Sher-Chowdhury/mysql-operator/commit/0bc124b3447dea2d53a16bd42f1e084abd306b83




Now deploy the operator. Theres 2 ways to do that. deploy it as a pod, or run it locally. 

### Approach 1 - Deploy operator as a container (production approach)

This involves creating an image with your operator installed inside it, then push that image up to a registry, e.g. dockerhub, quay.io. 

First build an image that has your controller baked in:

```
export account=sher_chowdhury
export image_name=mysql-operator
export tag_version=v0.0.1
docker login --username ${account} quay.io
operator-sdk build quay.io/${account}/${image_name}:${tag_version}
docker push quay.io/${account}/${image_name}:${tag_version}
sed -i "" "s|REPLACE_IMAGE|quay.io/${account}/${image_name}:${tag_version}|g" deploy/operator.yaml
```

In this example, we have made our operator image publicly available. Extra steps are needed if you want your image to be private. 




Now we do the deployment (assuming crd was already deployed, see above):


```
kubectl apply -f deploy/role.yaml
kubectl apply -f deploy/role_binding.yaml
kubectl apply -f deploy/service_account.yaml
kubectl apply -f deploy/operator.yaml
```

check the operator pod is now up:

```
kubectl get pods
```

To delete your operator from the cluster, do:

```
$ kubectl delete -f deploy/
```




### Approach 2 - Run it locally (quicker developmental approach)

This approach is better for developing your operator. Because it's faster.  


```
export OPERATOR_NAME=mysql-operator
operator-sdk run --local --namespace=default
```
With this approach, you only need to deploy the crd before running the above command. 

You don't need to create the role, serviceaccount, and rolebinding, since it's using your own privileges, since it's using your local kubeconfig details. 


## Create CR


After that we can deploy our example cr:

```
kubectl apply -f deploy/crds/cache.codingbee.net_mysqls_crd.yaml
kubectl apply -f deploy/crds/my-mysql-db-cr.yaml
```

Then check if if worked:

```
kubectl get mysql
kubectl get pods
```

if it didn't work. then take a look at the operator's pod's log:

```
kubectl log mysql-operator-xxxx
```


if you find the problem, fix it. Then redeploy newer version of the operator and retest, do:
```
export account=sher_chowdhury
export image_name=mysql-operator
export tag_version=v0.0.1
operator-sdk build quay.io/${account}/${image_name}:${tag_version}
docker push quay.io/${account}/${image_name}:${tag_version}
kubectl replace -f deploy/operator.yaml
kubectl replace -f deploy/crds/my-mysql-db-cr.yaml
```

Once your mysqldb pod is present, you can test it:

You can test your pod by running:

```
kubectl exec -it <mysql-pod-name> -- bash
mysql -u root -h localhost -p$MYSQL_ROOT_PASSWORD
```

Then in the msyql prompt, run:

```
show databases;
```

This will end up listing the "wordpressDB" database. 



## Now add a service object

here we've added a service object - https://github.com/Sher-Chowdhury/mysql-operator/commit/a6f8df1a8bbde96a0f041ab137672d4d4361f3e8

The main changes have been done to the pkg/controller/mysql/mysql_controller.go file. We had to add 3 sections:

1. A new watch block for the service. This watch block will in trigger the reconciler if and when it notices a change has occured. It does that by putting something onto a queue. 
2. A new block in reconcile function - this calls the third function, then starts the actual loop. Inside this loop it tries to create the object. The loop exits once the object in question is created. Otherwise it keeps calling itself from inside the loop.  
3. created the new newServiceForCR. This function just generates the yaml file that will get used to create the object. 




Now perform a retest. First we redeploy the updated operator and recreate the cr:

```
export account=sher_chowdhury
export image_name=mysql-operator
export tag_version=v0.0.1
operator-sdk build quay.io/${account}/${image_name}:${tag_version}
docker push quay.io/${account}/${image_name}:${tag_version}
kubectl replace -f deploy/operator.yaml
kubectl replace -f deploy/crds/my-mysql-db-cr.yaml
```


Now we can test this new mysqldb service by running:

```
kubectl run -it --rm --image=mysql:latest client -- bash
export MYSQL_ROOT_PASSWORD=wpAdminPassword
mysql -u root -h my-mysql-db-service -p$MYSQL_ROOT_PASSWORD
```

This gives us the mysql cmd prompt:

```
mysql>
```


Then in the msyql prompt, run:

```
SHOW databases;
USE wordpressDB;
SHOW TABLES;
CREATE TABLE customers (userID INT, userFirstName char(25), userLastName char(25), userEmailAddress char(50));
SHOW TABLES;
DESCRIBE customers;
INSERT INTO customers (userID,userFirstName,userLastName,userEmailAddress) VALUES (1,"Peter","Parker","spiderman.gmail.com");
INSERT INTO customers (userID,userFirstName,userLastName,userEmailAddress) VALUES (2,"Tony","Stark","ironman.gmail.com");
INSERT INTO customers (userID,userFirstName,userLastName,userEmailAddress) VALUES (3,"Steve","Rogers","captain_america.gmail.com");
INSERT INTO customers (userID,userFirstName,userLastName,userEmailAddress) VALUES (4,"Bruce","Banner","the_hull.gmail.com");
SELECT * FROM customers;
exit
```

Next exit out. 

Then delete the pod:


```
$ kubectl delete pod my-mysql-db-pod
```

A new pod comes up in it's place. Let's go back take a look inside it:

```
kubectl run -it --rm --image=mysql:latest client -- bash
export MYSQL_ROOT_PASSWORD=wpAdminPassword
mysql -u root -h my-mysql-db-service -p$MYSQL_ROOT_PASSWORD
```

Then on the mysql prompt, we get

```

SHOW databases;
USE wordpressDB;
SHOW TABLES;
```

This outputs:

```
mysql> SHOW databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| wordpressDB        |
+--------------------+
5 rows in set (0.01 sec)

mysql> USE wordpressDB;
Database changed
mysql> SHOW TABLES;
Empty set (0.00 sec)

mysql>
```



To make this persistent in case the pod fails, use pvc. 




## Organise our code into go packages

Our mysql_operator.go file is getting quite big. We can break down this file by organise the code into separate files. I'll do this by [creating packages](https://github.com/Sher-Chowdhury/gsg_child_packages). To do this I'll move the NewPodForCR function intos it's own .go file - https://github.com/Sher-Chowdhury/mysql-operator/commit/698670a8de4ebbd24a4bbb168034de6f5fbf3f96

Note:
- A package's public function needs to start with a capital letter. 
- sometimes vs code complains about false errors, in which case try restarting vscode. 
- I needed to create the packages folder in the same directory as the mysql_controller.go. Although I think this is unnecessary.
- vscode seems to updated the import block on it's own. And listed the packages in alphabetical order



I did the same thing for the NewServiceForCR function too - https://github.com/Sher-Chowdhury/mysql-operator/commit/179d5044604c986296238d3f8eeb540cb1de078a







## Making data persistent. 

If you're mysql pod dies, then all the data stored in it's database get's lost too. Let's demo this:

```
kubectl apply -f deploy/crds/my-mysql-db-cr.yaml
```

Let's open a test pod:

```
kubectl exec -it my-mysql-db-pod -- bash
```

Then start a sql session:

```
mysql -u root -h localhost -p$MYSQL_ROOT_PASSWORD
```

And see what tables are in our database:

```
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| wordpressDB        |
+--------------------+
5 rows in set (0.00 sec)

mysql> USE wordpressDB;
Database changed
mysql> SHOW TABLES;
Empty set (0.00 sec)

```


Now let's create a table and put some dummy data inside it:


```
CREATE TABLE customers (
  userID INT, 
  userFirstName char(25), 
  userLastName char(25), 
  userEmailAddress char(50)
);
```

Let's confirm that table exists:


```
mysql> SHOW TABLES;
+-----------------------+
| Tables_in_wordpressDB |
+-----------------------+
| customers             |
+-----------------------+
1 row in set (0.01 sec)
```

Which at the moment doesn't contain any entries:

```
mysql> SELECT * FROM customers;
Empty set (0.00 sec)

mysql> 
```


Now lets populatee this table with some data:

```
INSERT INTO customers (userID,userFirstName,userLastName,userEmailAddress) VALUES (1,"Peter","Parker","spiderman.gmail.com");
INSERT INTO customers (userID,userFirstName,userLastName,userEmailAddress) VALUES (2,"Tony","Stark","ironman.gmail.com");
INSERT INTO customers (userID,userFirstName,userLastName,userEmailAddress) VALUES (3,"Steve","Rogers","captain_america.gmail.com");
INSERT INTO customers (userID,userFirstName,userLastName,userEmailAddress) VALUES (4,"Bruce","Banner","the_hull.gmail.com");
```

This results in:

```
mysql> SELECT * FROM customers;
+--------+---------------+--------------+---------------------------+
| userID | userFirstName | userLastName | userEmailAddress          |
+--------+---------------+--------------+---------------------------+
|      1 | Peter         | Parker       | spiderman.gmail.com       |
|      2 | Tony          | Stark        | ironman.gmail.com         |
|      3 | Steve         | Rogers       | captain_america.gmail.com |
|      4 | Bruce         | Banner       | the_hull.gmail.com        |
+--------+---------------+--------------+---------------------------+
4 rows in set (0.00 sec)

```

Now if we exit out of this pod, delete it then go into the newly provisioned pod:

```
mysql> exit
Bye
root@my-mysql-db-pod:/# exit
exit 

$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
my-mysql-db-pod                  1/1     Running   0          18m
mysql-operator-74676f9d4-js2k8   1/1     Running   0          18m
$ kubectl delete pods my-mysql-db-pod 
pod "my-mysql-db-pod" deleted

$ kubectl get pods                   
NAME                             READY   STATUS    RESTARTS   AGE
my-mysql-db-pod                  1/1     Running   0          44s
mysql-operator-74676f9d4-js2k8   1/1     Running   0          20m


$ kubectl exec -it my-mysql-db-pod -- bash

root@my-mysql-db-pod:/# mysql -u root -h localhost -p$MYSQL_ROOT_PASSWORD
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.19 MySQL Community Server - GPL

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| wordpressDB        |
+--------------------+
5 rows in set (0.00 sec)

mysql> USE wordpressDB
Database changed
mysql> SHOW TABLES;
Empty set (0.00 sec)

mysql> 
```

As you can see, our table and it's data has been wiped out. 





To prevent that from happening, we need to make use of Persitent Volumes. You can create it directly using a PV object. But it's better to create it indirectly using a PVC instead. That's because PVs created from a PVC can be retained even if you delete your custom resource, we'll explain how thats done later. 





To achieve this, we need to take the following steps:

1. Update types 
    1. update file to include new settings needed in order to create PVC - https://github.com/Sher-Chowdhury/mysql-operator/blob/e3f4c3bb6b3a0ca42e49197d4672fe86da0d93b9/pkg/apis/cache/v1alpha1/mysql_types.go#L36 and https://github.com/Sher-Chowdhury/mysql-operator/blob/e3f4c3bb6b3a0ca42e49197d4672fe86da0d93b9/pkg/apis/cache/v1alpha1/mysql_types.go#L19-L22
    1. Perform - `operator-sdk generate k8s`
    2. Updated crd - `operator-sdk generate crds`
2. update example cr file - https://github.com/Sher-Chowdhury/mysql-operator/blob/e3f4c3bb6b3a0ca42e49197d4672fe86da0d93b9/deploy/crds/my-mysql-db-cr.yaml#L11-L13
3. add new watch for pvc - https://github.com/Sher-Chowdhury/mysql-operator/blob/e3f4c3bb6b3a0ca42e49197d4672fe86da0d93b9/pkg/controller/mysql/mysql_controller.go#L56-L64
4. Add logic for pvc in reconcile function - https://github.com/Sher-Chowdhury/mysql-operator/blob/e3f4c3bb6b3a0ca42e49197d4672fe86da0d93b9/pkg/controller/mysql/mysql_controller.go#L125-L151 This calls the NewPvcForCR function we created in the next step. 
5. create new function for defining the pvc yaml definition. I've created this in the form of a package - https://github.com/Sher-Chowdhury/mysql-operator/blob/e3f4c3bb6b3a0ca42e49197d4672fe86da0d93b9/pkg/controller/mysql/resources/pvcs/mysql-pvc.go


notes: Here we specified the storage class setting. 
https://github.com/Sher-Chowdhury/mysql-operator/blob/e3f4c3bb6b3a0ca42e49197d4672fe86da0d93b9/pkg/apis/cache/v1alpha1/mysql_types.go#L21
That is not a mandatory field but we have added it in for now because we'll use it as part of a demo later on. 



After that, you can retest this by running:


```
$ kubectl replace -f deploy/crds/my-mysql-db-cr.yaml --force
```

Ok this created the pvc, which in turn creates the pv:

```
$ kubectl get pvc

$ kubectl get pv
```


However we haven't made added code in order for the pod to make use of this PV. Let's do that now. 


Tip, before writing this function, it's a good idea to write out the yaml equivalent and then test it out. e.g. something like:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: my-mysql-db
  name: my-mysql-db-pod
spec:
  containers:
  - env:
    - name: MYSQL_ROOT_PASSWORD
      value: wpAdminPassword
    - name: MYSQL_DATABASE
      value: wordpressDB
    - name: MYSQL_USER
      value: wpuser
    - name: MYSQL_PASSWORD
      value: wpPassword
    image: docker.io/mysql:latest
    imagePullPolicy: Always
    name: mysqldb
    volumeMounts:                                ##  New section nested unter pod.spec.containers block
    - mountPath: /var/lib/mysql                  ##
      name: mysql-pvc-provisioned-volume         ##
  volumes:                                    ## New section neseted under pod.spec block
  - name: mysql-pvc-provisioned-volume        ##
    persistentVolumeClaim:                    ##
      claimName: my-mysql-db-pvc              ##
```

You should then test out this sample code to confirm it's working. After confirming it works, you can then codify it into golang code. 


This sample will also help you visualise what your yaml function would look like. For example, we have to write code to generate to different parts `pod.spec.containers.volumeMounts` and `pod.spec.volumes` and they are both linked together by their respective `name` settings. 

I used `/var/lib/mysql` as the mountpath as advised by the https://hub.docker.com/_/mysql


Here how this looks like - https://github.com/Sher-Chowdhury/mysql-operator/commit/2e8e78b2ea9e57306ad554914f1f4d2a44ddb5a8


The best way to understan how to write this is to break it into 2 parts. so first write out the code to generate `pod.spec block` then repeat the approach for the `pod.spec.containers` block. 


 Then see what you need and  list them out. This list outlines that we need 'a' which depends 'b' which depends on 'c'. once you have this list, reverse the order of that list and then work down that list. 


Then  retest. You should find the data persists even when the pod dies.  


However the data does get deleted if you delete the whole cr itself:


```
$ kubectl replace -f deploy/crds/my-mysql-db-cr.yaml --force
```

That's because This ends up deleting the PV as a whole. If you want to keep the PV, then that's possible by specifying a storageclass with a reclaimpolicy set to 'retain'.

```
$ kubectl get sc
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  6h27m
```

Ensure storage class with 'retain' option is enabled. This is in order to retain PV even if the PVC or the CR as a whole gets deleted- This storageclass is something that should get created at the of building the kubecluster itself. It's bad practice to create this as part of this mysql operator. The following can be used to create this sc in minikube:

```
kubectl apply -f deploy/minikube-storageclass.yaml
```

This ends up with:

```
$ kubectl get storageclasses -o wide                
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
retained-volumes     k8s.io/minikube-hostpath   Retain          Immediate           false                  6m21s
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  65m
```



Next you need to create a PV from this new sc. Since unfortunately a PVC can't rebind to a PV it earlier created. So need to use the volumeName+claimref technique - https://stackoverflow.com/a/55443675


Another use case, is that might not care what happens to your PV once the cr gets deleted. In that scenario it becomes an unnecessary burden to make it mandatory to fill out this storageclass section and instead make it default to the default storage class instead. 

E.g. let's replace our existing cr defintions with these 2. I shortened the cr.names too to make things simpler. 
- https://github.com/Sher-Chowdhury/mysql-operator/commit/e08e86ce2956d16010025f1e1cebe9d46951c97a#diff-0962a40d6cd398a5963b080e42cc727eL4
- https://github.com/Sher-Chowdhury/mysql-operator/commit/e08e86ce2956d16010025f1e1cebe9d46951c97a#diff-05ae5677ad12ba673677c0fb353160b0L4


Then I made StorageClass setting optional, by enabling the omitempty setting - https://github.com/Sher-Chowdhury/mysql-operator/commit/e08e86ce2956d16010025f1e1cebe9d46951c97a#diff-bbc388b9f979f725f3962a950d7b75b4L21

After running the `operator-sdk generate k8s` command, the sc gets removed from the required list - https://github.com/Sher-Chowdhury/mysql-operator/commit/e08e86ce2956d16010025f1e1cebe9d46951c97a#diff-7fda698534788ca007b527fa28af6e3dL63

Next I create the pvcSpec. That starts with a blank struct, which I then gradually populate with available settings - https://github.com/Sher-Chowdhury/mysql-operator/commit/e08e86ce2956d16010025f1e1cebe9d46951c97a#diff-44d953b0b10edde67687c5268139cb2cR17-R46

This ends up replacing the existing logic - https://github.com/Sher-Chowdhury/mysql-operator/commit/e08e86ce2956d16010025f1e1cebe9d46951c97a#diff-44d953b0b10edde67687c5268139cb2cL25-L35



Now both of the new cr's should work. 


```
$ kubectl apply -f deploy/crds/my-mysql-db-cr.yaml
mysql.cache.codingbee.net/mydb created

$ kubectl apply -f deploy/crds/my-mysql-db-cr-with-set-storageclass.yaml
mysql.cache.codingbee.net/mydb-with-set-sc created

$ kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
mydb-pod               1/1     Running   0          36s
mydb-with-set-sc-pod   1/1     Running   0          18s

$ kubectl get pvc
NAME                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
mydb-pvc               Bound    pvc-fc8c94fb-f918-443c-932c-3a5deeb22ff1   1Gi        RWO            standard           48s
mydb-with-set-sc-pvc   Bound    pvc-039a38c4-f48b-4f20-b439-0d1901740d64   1Gi        RWO            retained-volumes   30s
```




There is a third scenaro which specifies storageclass, but is left blank. e.g.:

```
$ cat tmp-pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-with-blank-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName:

$ kubectl apply -f tmp-pvc.yaml
persistentvolumeclaim/pvc-with-blank-storage created

$ kubectl get pvc pvc-with-blank-storage
NAME                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-with-blank-storage   Bound    pvc-f00fd8cc-b6d3-49b7-adf8-b29a3ce951c7   1Gi        RWO            standard       10s

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                            STORAGECLASS       REASON   AGE
pvc-f00fd8cc-b6d3-49b7-adf8-b29a3ce951c7   1Gi        RWO            Delete           Bound    default/pvc-with-blank-storage   standard                    30s

```

I created a similar cr to replicate this:

```
$ cat deploy/crds/my-mysql-db-cr-with-blank-storageclass.yaml
apiVersion: cache.codingbee.net/v1alpha1
kind: MySQL
metadata:
  name: mydb-with-blank-sc
spec:
  environment:
    mysql_database: wordpressDB
    mysql_password: wpPassword
    mysql_root_password: wpAdminPassword
    mysql_user: wpuser
  volume:
    storage_class:
    volume_size: 1Gi
```

However that didn't work:

```
$ kubectl apply -f deploy/crds/my-mysql-db-cr-with-blank-storageclass.yaml
The MySQL "mydb-with-blank-sc" is invalid: spec.volume.storage_class: Invalid value: "null": spec.volume.storage_class in body must be of type string: "null"
```

This error message appears even when your operator pod isn't running. That means that problem is being reported by kubernetes itslef based on the crd spec. 

Therefore I made the following change - https://github.com/Sher-Chowdhury/mysql-operator/commit/f458f4827bb2869255b25da5f6bf5e2c79680570


I found out about `nullable` settings here - https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#specifying-a-structural-schema

To update our crd to enable this feature - we have to update our mysql_types.go file , and then ran `operator-sdk generate crds`-
- https://github.com/Sher-Chowdhury/mysql-operator/commit/3dde8b8581b7ff40ed1d4e1f3da55fea43f04fb6
I found what to set here:
- https://book.kubebuilder.io/reference/markers/crd-validation.html?highlight=null#crd-validation
- https://github.com/kubernetes-sigs/kubebuilder
- https://book-v1.book.kubebuilder.io/beyond_basics/generating_crd.html



So with this in place, all the following scenarios should now work:

1. yaml with no storageclass field at all. (should default to default sc)
2. yaml with storage class, but left blank (should default to default sc)
3. yaml with storage class set to empty string.  (should default to default sc)
4. yaml with storage class set to specific name  (should use to specified sc)

We have now nearly reached the Phase 1 of the Maturity Model - 

https://docs.openshift.com/container-platform/4.3/operators/olm-what-operators-are.html#olm-maturity-model_olm-what-operators-are

Things this outstanding to full reach phase 1. 

- more configuration ability for mysql, e.g make use of config maps. 
- better db login cred managment - using kubectl secrets. 
- Implement bast practice - e.g. resource cpu/memory requests and limits. 
- implement security best practice. 

https://medium.com/@cloudark/introducing-kubernetes-operator-maturity-model-for-multi-operator-platforms-952d2e637a82



















# setup up status info for our CR. 

Useful link: https://medium.com/@xcoulon/customizing-the-server-side-printing-of-your-kubernetes-custom-resources-1c84b7631686

At the moment `kubectl get mysql` only has 2 columns, name and age. We want to add more. 


```
$ kubectl get mysql my-mysql-db
NAME          AGE
my-mysql-db   2m52s

$ kubectl get mysql my-mysql-db -o yaml
apiVersion: cache.codingbee.net/v1alpha1
kind: MySQL
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"cache.codingbee.net/v1alpha1","kind":"MySQL","metadata":{"annotations":{},"name":"my-mysql-db","namespace":"default"},"spec":{"environment":{"mysql_database":"wordpressDB","mysql_password":"wpPassword","mysql_root_password":"wpAdminPassword","mysql_user":"wpuser"}}}
  creationTimestamp: "2020-02-16T10:33:19Z"
  generation: 1
  name: my-mysql-db
  namespace: default
  resourceVersion: "196765"
  selfLink: /apis/cache.codingbee.net/v1alpha1/namespaces/default/mysqls/my-mysql-db
  uid: b308c6e0-cede-482b-bf06-083dc50523be
spec:
  environment:
    mysql_database: wordpressDB
    mysql_password: wpPassword
    mysql_root_password: wpAdminPassword
    mysql_user: wpuser


$ kubectl describe mysql my-mysql-db
Name:         my-mysql-db
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"cache.codingbee.net/v1alpha1","kind":"MySQL","metadata":{"annotations":{},"name":"my-mysql-db","namespace":"default"},"spec...
API Version:  cache.codingbee.net/v1alpha1
Kind:         MySQL
Metadata:
  Creation Timestamp:  2020-02-16T10:33:19Z
  Generation:          1
  Resource Version:    196765
  Self Link:           /apis/cache.codingbee.net/v1alpha1/namespaces/default/mysqls/my-mysql-db
  UID:                 b308c6e0-cede-482b-bf06-083dc50523be
Spec:
  Environment:
    mysql_database:       wordpressDB
    mysql_password:       wpPassword
    mysql_root_password:  wpAdminPassword
    mysql_user:           wpuser
Events:                   <none>
```





# References

https://github.com/operator-framework/operator-sdk/blob/v0.15.1/doc/user-guide.md
