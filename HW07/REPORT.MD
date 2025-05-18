# Домашнее задание 7

## Postgres в minikube  

Задание выполняется на компьютере с ОС Windows 10, с установленным Hyper-V.

Установлен minikube. Запускаем minikube:

```console
PS C:\Windows\system32> minikube start --hyperv-use-external-switch --container-runtime=docker 
* minikube v1.31.2 на Microsoft Windows 10 Pro 10.0.19045.5854 Build 19045.5854
* Используется драйвер hyperv на основе конфига пользователя
* Запускается control plane узел minikube в кластере minikube
* Creating hyperv VM (CPUs=2, Memory=6000MB, Disk=20000MB) ...
* Подготавливается Kubernetes v1.27.4 на Docker 24.0.4 ...
  - Generating certificates and keys ...
  - Booting up control plane ...
  - Configuring RBAC rules ...
* Configuring bridge CNI (Container Networking Interface) ...
* Компоненты Kubernetes проверяются ...
  - Используется образ gcr.io/k8s-minikube/storage-provisioner:v5
* Включенные дополнения: storage-provisioner, default-storageclass
* Готово! kubectl настроен для использования кластера "minikube" и "default" пространства имён по умолчанию


PS C:\Windows\system32> minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

PS C:\Windows\system32> minikube ip
192.168.0.196
```

Подключимся:

```console
ssh -i ~/.minikube/machines/minikube/id_rsa docker@192.168.0.196
или
PS C:\Windows\system32> minikube ssh

                         _             _
            _         _ ( )           ( )
  ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __
/' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
| ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
(_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)


```

### 1. Установка через манифест
  
Создадим манифест :  
<details>
  <summary>postgres.yml</summary>

```yaml
  #Том 2ГБ с единоличным использованием подом и требование к нему
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres
spec:
  accessModes:
    - ReadWriteOncePod
  storageClassName: standard
  capacity:
    storage: 2Gi
  hostPath:
    path: /data/postgres
  
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres
spec:
  accessModes:
    - ReadWriteOncePod
  resources:
    requests:
      storage: 2Gi
  volumeName: postgres
---
#Секрет с паролем УЗ postgres, type=opaque - тип для произвольных данных
apiVersion: v1
kind: Secret
metadata:
  name: postgres
type: Opaque
data:
  POSTGRES_USER: cG9zdGdyZXM=     # echo postgres | base64
  POSTGRES_PASSWORD: c2VjcmV0    # echo secret | base64
---
#Stateful под с одним контейнером для прямого сетевого доступа
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  name: postgres
  labels:
    app: postgres
    role: service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
      role: service
  serviceName: postgres
  template:
    metadata:
      labels:
        app: postgres
        role: service
    spec:
      containers:
        - name: postgres
          image: postgres:14
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  key: POSTGRES_USER
                  name: postgres
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: POSTGRES_PASSWORD
                  name: postgres
          ports:
            - containerPort: 5432
              name: postgres
              protocol: TCP
          volumeMounts:
            - name: postgres
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: postgres
          persistentVolumeClaim:
            claimName: postgres
---
#Простой сетевой доступ через балансировщик
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
    role: service
spec:
  selector:
    app: postgres
    role: service
  type: LoadBalancer
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
      protocol: TCP
```

</details>

Создадим ресурсы

```console
PS E:\_otus\HW07> kubectl.exe --context=minikube apply -f ./postgres.yml
persistentvolume/postgres created
persistentvolumeclaim/postgres created
secret/postgres created
statefulset.apps/postgres created
service/postgres created
```

Тестируем с хоста

```console
PS E:\_otus\HW07> kubectl.exe --context=minikube get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP      10.96.0.1      <none>        443/TCP          10h
postgres     LoadBalancer   10.111.73.27   <pending>     5432:31740/TCP   25s
```

>Балансировщик использует IP 10.111.73.27  

```console
PS C:\Program Files\PostgreSQL\14\scripts> .\runpsql.bat
Server [localhost]: 10.111.73.27
Database [postgres]:
Port [5432]:
Username [postgres]:
Пароль пользователя postgres:
psql (14.4, сервер 14.18 (Debian 14.18-1.pgdg120+1))
ПРЕДУПРЕЖДЕНИЕ: Кодовая страница консоли (866) отличается от основной
                страницы Windows (1251).
                8-битовые (русские) символы могут отображаться некорректно.
                Подробнее об этом смотрите документацию psql, раздел
                "Notes for Windows users".
Введите "help", чтобы получить справку.

postgres=#
```

Тестируем с minikube

```console
PS E:\_otus\HW07> minikube service postgres  --url
http://192.168.0.196:31740

kubectl --context=minikube run pgbox --image=postgres:14 --rm -it --restart=Never -- bash -c "export PGPASSWORD=secret && 
             psql -v ON_ERROR_STOP=1 -P pager=off --single-transaction --host=192.168.0.196 --port=31740 \
                  --username=postgres --dbname=postgres \
                  --command='SELECT version();'"
                                                        version
-----------------------------------------------------------------------------------------------------------------------
 PostgreSQL 14.18 (Debian 14.18-1.pgdg120+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit
(1 row)

pod "pgbox" deleted
```

### 2. Установка с использованием HELM

Установим helm

```console
PS C:\Windows\system32> choco install kubernetes-helm
Chocolatey v2.2.2
Installing the following packages:
kubernetes-helm
...
 The install of kubernetes-helm was successful.
  Software installed to 'C:\ProgramData\chocolatey\lib\kubernetes-helm\tools'

Chocolatey installed 1/1 packages.
 See the log for details (C:\ProgramData\chocolatey\logs\chocolatey.log).
 
PS C:\Windows\system32> helm version
version.BuildInfo{Version:"v3.17.3", GitCommit:"e4da49785aa6e6ee2b86efd5dd9e43400318262b", GitTreeState:"clean", GoVersion:"go1.23.7"}

```

Добавим репозиторий

```console
PS C:\Windows\system32> helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
PS C:\Windows\system32> helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Создадим PV и PVC аналогичным образом как в основном задании:
<details>
  <summary>postgres-pv.yml</summary>

```yaml
  #Том 2ГБ с единоличным использованием подом и требование к нему
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres
spec:
  accessModes:
    - ReadWriteOncePod
  storageClassName: standard
  capacity:
    storage: 2Gi
  hostPath:
    path: /data/postgres
  
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres
spec:
  accessModes:
    - ReadWriteOncePod
  resources:
    requests:
      storage: 2Gi
  volumeName: postgres
```

</details>

Создадим ресурсы

```console
PS E:\_otus\HW07> kubectl.exe --context=minikube apply -f ./postgres-pv.yml
persistentvolume/postgres created
persistentvolumeclaim/postgres created
PS E:\_otus\HW07> kubectl.exe --context=minikube get pvc
NAME       STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgres   Bound    postgres   2Gi        RWOP           standard       15s
```

Выполняем установку приложения

```console
PS E:\_otus\HW07>helm install postgres-helm bitnami/postgresql --set auth.postgresPassword='secret'  --set primary.persistence.existingClaim=postgres  --set volumePermissions.enabled=true
NAME: postgres-helm
LAST DEPLOYED: Sun May 18 13:37:32 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 16.7.4
APP VERSION: 17.5.0
```

Для разнообразия в этот раз пробросим порт на созданный чартом сервис

```console
PS E:\_otus\HW07> Start-Job -ScriptBlock {kubectl port-forward --namespace default svc/postgres-helm-postgresql 5432:5432}

Id     Name            PSJobTypeName   State         HasMoreData     Location             Command
--     ----            -------------   -----         -----------     --------             -------
1      Job1            BackgroundJob   Running       True            localhost            kubectl port-forward -...
```

Проверим:

```console
PS C:\Program Files\PostgreSQL\14\scripts> .\runpsql.bat
Server [localhost]:
Database [postgres]:
Port [5432]:
Username [postgres]:
Пароль пользователя postgres:
psql (14.4, сервер 17.5)
ПРЕДУПРЕЖДЕНИЕ: psql имеет базовую версию 14, а сервер - 17.
                Часть функций psql может не работать.
ПРЕДУПРЕЖДЕНИЕ: Кодовая страница консоли (866) отличается от основной
                страницы Windows (1251).
                8-битовые (русские) символы могут отображаться некорректно.
                Подробнее об этом смотрите документацию psql, раздел
                "Notes for Windows users".
Введите "help", чтобы получить справку.

postgres=# SELECT version();
                                          version
-------------------------------------------------------------------------------------------
 PostgreSQL 17.5 on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit
(1 ёЄЁюър)


postgres=#
```
