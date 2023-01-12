# kubernetes-introduction

# 1. Installer Minikube et Kubectl
Suivre cette documentation : https://minikube.sigs.k8s.io/docs/start/

## Installer Minikube

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### Démarrer minikube

```
minikube start
```

## Installer Kubectl grâce a Minikube

```
minikube kubectl -- get po -A
alias kubectl="minikube kubectl --"
```

#  2. Pod Nginx
## a. Héberger un premier Pod Nginx

```
# kubectl run my-nginx --image=nginx --port=80
pod/my-nginx created
```

```
# kubectl get pod
NAME       READY   STATUS             RESTARTS   AGE
my-nginx   1/1     Running   0          34m
```

**Il est aussi possible d'héberger un pod grâce au fichier de configuration du service :** 

```
vim nginx.yml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

**Déployer le service :**
```
kubectl apply -f nginx.yml
```
**Vérifier le déploiement :** 
```
# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           4m32s
```

## b. A l’aide de la commande kubectl port-forward et d’un navigateur accéder à la page par défaut de votre pod Nginx

**Après un kubectl run :**
```
# kubectl port-forward my-nginx 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

**Après un kubectl apply :**
```
kubectl port-forward deployment/nginx-deployment 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

![](https://i.imgur.com/gnCkPrf.png)


# 3. Connexion entre plusieurs Pods
## a. A l’image du TP 1 sur Docker (question 7 et 8), héberger un Pod phpmyadmin et mysql, cette fois-ci en utilisant minikube et lier un service associé au pod mysql.

```
kubectl run mysql --image=mysql:latest
kubectl run phpmyadmin --image=phpmyadmin:latest
```

- Créer un fichier de déploiement mysql
```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```


- Créer un fichier de déploiement phpmyadmin

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpmyadmin-deployment
  labels:
    app: phpmyadmin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phpmyadmin
  template:
    metadata:
      labels:
        app: phpmyadmin
    spec:
      containers:
        - name: phpmyadmin
          image: phpmyadmin/phpmyadmin
          ports:
            - containerPort: 80
          env:
            - name: PMA_HOST
              value: mysql-service
            - name: PMA_PORT
              value: "3306"
            - name: MYSQL_ROOT_PASSWORD
              value: P@ssw0rd
            - name: MYSQL_USER
              value: toto
            - name: MYSQL_PASSW0RD
              value: P@ssw0rd
```

- Lancer le déploiement

```
kubectl apply -f mysql.yml
service/mysql created

kubectl apply -f phpmyadmin.yml
deployment.apps/phpmyadmin-deployment created


```

Vérifier le déploiement de chacun des services :

```
kubectl get deployment
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
mysql                   0/1     1            0           18s
nginx-deployment        1/1     1            1           38m
phpmyadmin-deployment   0/1     1            0           18m
```

## b. Connecter phpmyadmin avec le Service mysql et vérifier que vous pouvez administrer cette base de données

```
---
apiVersion: v1
kind: Service
metadata:
  name: phpmyadmin-service
spec:
  type: NodePort
  selector:
    app: phpmyadmin
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```
```
kubectl apply -f phpmyadmin.yml
```

## c. Avec la commande kubectl-port forward, vérifier que phpmyadmin arrive à contacter et administrer votre base de données mysql

- Créer une base de données en ajoutant l'environnement MYSQL_DATABASE dans le fichier mysql-deployment.yaml

```
env:
  - name: MYSQL_DATABASE
    value: ynov
```

- Appliquer les configurations pour le pod mysql


```
$ kubectl apply -f mysql-deployment.yaml
```

- Vérifier que phpmyadmin se connecte à l base de données avec l'utilisateur "toto"

```
$ kubectl port-forward phpmyadmin-deployment-7bd96d95d4-zbg7l 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

![](https://i.imgur.com/SV0xxGH.png) (User: toto - Pass: P@ssw0rd)

![](https://i.imgur.com/2B9ofua.png)





## d. Ajouter un Ingress pour accéder à phpmyadmin sans utiliser la commande kubectl port-forward

- Créer un fichier ingress.yaml

```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: phpmyadmin-http-ingress
spec:
  rules:
  - host: phpmyadmin.local
  - http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: phpmyadmin-service
            port:
              number: 80
```

- Activer ingress

```
$ minikube addons enable ingress
```

- Créer un DNS pour le host: phpmyadmin.local
```
$sudo vim /etc/hosts
192.168.49.2 phpmyadmin.local
L'IP de l'ingress s'obtient en faisant la commande suivante:
```
```
$ kubectl get ingress
NAME                      CLASS   HOSTS              ADDRESS        PORTS   AGE
phpmyadmin-http-ingress   nginx   phpmyadmin.local   192.168.49.2   80      6m2s
```
