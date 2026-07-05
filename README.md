# Домашнее задание к занятию «Хранение в K8s»

## Задание 1. Volume: обмен данными между контейнерами в поде
### Задача

Создать Deployment приложения, состоящего из двух контейнеров, обменивающихся данными.

### Шаги выполнения
1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.
2. Настроить busybox на запись данных каждые 5 секунд в некий файл в общей директории.
3. Обеспечить возможность чтения файла контейнером multitool.


### Что сдать на проверку
- Манифесты:
  - `containers-data-exchange.yaml`
- Скриншоты:
  - описание пода с контейнерами (`kubectl describe pods data-exchange`)
  - вывод команды чтения файла (`tail -f <имя общего файла>`)

------

## Задание 2. PV, PVC
### Задача
Создать Deployment приложения, использующего локальный PV, созданный вручную.

### Шаги выполнения
1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool, использующего созданный ранее PVC
2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде.
3. Продемонстрировать, что контейнер multitool может читать данные из файла в смонтированной директории, в который busybox записывает данные каждые 5 секунд. 
4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему. (Используйте команду `kubectl describe pv`).
5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV.  Продемонстрировать, что произошло с файлом после удаления PV. Пояснить, почему.


### Что сдать на проверку
- Манифесты:
  - `pv-pvc.yaml`
- Скриншоты:
  - каждый шаг выполнения задания, начиная с шага 2.
- Описания:
  - объяснение наблюдаемого поведения ресурсов в двух последних шагах.

------

## Задание 3. StorageClass
### Задача
Создать Deployment приложения, использующего PVC, созданный на основе StorageClass.

### Шаги выполнения

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool, использующего созданный ранее PVC.
2. Создать SC и PVC для подключения папки на локальной ноде, которая будет использована в поде.
3. Продемонстрировать, что контейнер multitool может читать данные из файла в смонтированной директории, в который busybox записывает данные каждые 5 секунд.

### Что сдать на проверку
- Манифесты:
  - `sc.yaml`
- Скриншоты:
  - каждый шаг выполнения задания, начиная с шага 2
---
## Шаблоны манифестов с учебными комментариями
### 1. Deployment (containers-data-exchange.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-exchange
spec:
  replicas: # ЗАДАНИЕ: Укажите количество реплик
  selector:
    matchLabels:
      app: # ДОПОЛНИТЕ: Метка для селектора
  template:
    metadata:
      labels:
        app: # ПОВТОРИТЕ: Метка из selector.matchLabels
    spec:
      containers:
      - name: # ДОПОЛНИТЕ: Имя первого контейнера
        image: busybox
        command: ["/bin/sh", "-c"] 
        args: ["echo $(date) > путь_к_файлу; sleep 3600"] # КЛЮЧЕВОЕ: Команда записи данных в файл в директории из секции volumeMounts контейнера
        volumeMounts:
        - name: # ДОПОЛНИТЕ: Имя монтируемого раздела. Должно совпадать с именем эфемерного хранилища, объявленного на уровне пода.
          mountPath: # КЛЮЧЕВОЕ: Путь монтирования эфемерного хранилища внутри контейнера 1
      - name: # ДОПОЛНИТЕ: Имя второго контейнера
        image: busybox
        command: ["/bin/sh", "-c"]
        args: ["tail -f путь_к_файлу"] # КЛЮЧЕВОЕ: Команда для чтения данных из файла, расположенного в директории, указанной в volumeMounts контейнера
        volumeMounts:
        - name: # ДОПОЛНИТЕ: Имя монтируемого раздела. Должно совпадать с именем эфемерного хранилища, объявленного на уровне пода
          mountPath: # КЛЮЧЕВОЕ: Путь монтирования эфемерного хранилища внутри контейнера 2
      volumes:
      - name: # ДОПОЛНИТЕ: Имя монтируемого раздела эфемерного хранилища
        emptyDir: {} # ИНФОРМАЦИЯ: Определяем эфемерное хранилище, которое работает только внутри пода
```
### 2. Deployment (pv-pvc.yaml)
```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: # ДОПОЛНИТЕ: Имя хранилища
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: # КЛЮЧЕВОЕ: Путь к директории на ноде (хосте, на котором развёрнут кластер)
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: # ДОПОЛНИТЕ: Имя PVC
spec:
  volumeName: # ДОПОЛНИТЕ: Имя PV, к которому будет привязан PVC, должен совпадать с созданным ранее PV
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: # ДОПОЛНИТЕ: Какой объём хранилища вы хотите передать в контейнер. Должно быть меньше или равно параметру storage из PV
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-exchange-pvc
spec:
  replicas: # ЗАДАНИЕ: Укажите количество реплик
  selector:
    matchLabels:
      app: # ДОПОЛНИТЕ: Метка для селектора
  template:
    metadata:
      labels:
        app: # ПОВТОРИТЕ: Метка из selector.matchLabels
    spec:
      containers:
      - name: # ДОПОЛНИТЕ: Имя первого контейнера
        image: busybox
        command: ["/bin/sh", "-c"] 
        args: ["echo $(date) > путь_к_файлу; sleep 3600"] # КЛЮЧЕВОЕ: Команда записи данных в файл в директории из секции volumeMounts контейнера 
        volumeMounts:
        - name: # ДОПОЛНИТЕ: Имя монтируемого раздела. Должно совпадать с именем хранилища, объявленного на уровне пода
          mountPath: # КЛЮЧЕВОЕ: Путь монтирования хранилища внутри контейнера 1
      - name: # ДОПОЛНИТЕ: Имя второго контейнера
        image: busybox
        command: ["/bin/sh", "-c"]
        args: ["tail -f путь_к_файлу"] # КЛЮЧЕВОЕ: Команда для чтения данных из файла, расположенного в директории, указанной в volumeMounts контейнера
        volumeMounts:
        - name: # ДОПОЛНИТЕ: Имя монтируемого раздела. Должно совпадать с именем хранилища, объявленного на уровне пода
          mountPath: # КЛЮЧЕВОЕ: Путь монтирования хранилища внутри контейнера 2
      volumes:
      - name: # ДОПОЛНИТЕ: Имя монтируемого раздела хранилища
        persistentVolumeClaim:
          claimName: # КЛЮЧЕВОЕ: Совпадает с именем PVC объявленного ранее
```
### 3. Deployment (sc.yaml)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: # ДОПОЛНИТЕ: Имя StorageClass
provisioner: kubernetes.io/no-provisioner # ИНФОРМАЦИЯ: Нет автоматического развёртывания
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: # ДОПОЛНИТЕ: Имя PVC
spec:
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: # ДОПОЛНИТЕ: Какой объем хранилища вы хотите передать в контейнер. Должно быть меньше или равно параметру storage из PV
  storageClassName: # ДОПОЛНИТЕ: Имя StorageClass. Должно совпадать с объявленным ранее
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-exchange-sc
spec:
  replicas: # ЗАДАНИЕ: Укажите количество реплик
  selector:
    matchLabels:
      app: # ДОПОЛНИТЕ: Метка для селектора
  template:
    metadata:
      labels:
        app: # ПОВТОРИТЕ: Метка из selector.matchLabels
    spec:
      containers:
      - name: # ДОПОЛНИТЕ: Имя первого контейнера
        image: busybox
        command: ["/bin/sh", "-c"] 
        args: ["echo $(date) > путь_к_файлу; sleep 3600"] # КЛЮЧЕВОЕ: Команда для чтения данных из файла, расположенного в директории, указанной в volumeMounts контейнера
        volumeMounts:
        - name: # ДОПОЛНИТЕ: Имя монтируемого раздела. Должно совпадать с именем хранилища, объявленного на уровне пода
          mountPath: # КЛЮЧЕВОЕ: Путь монтирования хранилища внутри контейнера 1
      - name: # ДОПОЛНИТЕ: Имя второго контейнера
        image: busybox
        command: ["/bin/sh", "-c"]
        args: ["tail -f путь_к_файлу"] # КЛЮЧЕВОЕ: Команда для чтения данных из файла, расположенного в директории, указанной в volumeMounts контейнера
        volumeMounts:
        - name: # ДОПОЛНИТЕ: Имя монтируемого раздела. Должно совпадать с именем хранилища, объявленного на уровне пода
          mountPath: # КЛЮЧЕВОЕ: Путь монтирования хранилища внутри контейнера 2
      volumes:
      - name: # ДОПОЛНИТЕ: Имя монтируемого раздела хранилища
        persistentVolumeClaim:
          claimName: # КЛЮЧЕВОЕ: Совпадает с именем PVC объявленного ранее
```

## **Правила приёма работы**
1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, скриншоты результатов, пояснения.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

## **Критерии оценивания задания**
1. Зачёт: Все задачи выполнены, манифесты корректны, есть доказательства работы (скриншоты) и пояснения по заданию 2.
2. Доработка (на доработку задание направляется 1 раз): основные задачи выполнены, при этом есть ошибки в манифестах или отсутствуют проверочные скриншоты.
3. Незачёт: работа выполнена не в полном объёме, есть ошибки в манифестах, отсутствуют проверочные скриншоты. Все попытки доработки израсходованы (на доработку работа направляется 1 раз). Этот вид оценки используется крайне редко.

## **Срок выполнения задания**  
1. 5 дней на выполнение задания.
2. 5 дней на доработку задания (в случае направления задания на доработку).


---


# Выполнение домашнего задания

## Задание 1
Применяю манифест
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl apply -f containers-data-exchange.yaml
deployment.apps/data-exchange created
```

Узнаю имя пода
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl get po
NAME                                             READY   STATUS    RESTARTS      AGE
data-exchange-77895dc648-lddws                   2/2     Running   0             38s
```

Смотрю подробную информацию о поде
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl describe pod data-exchange-77895dc648-lddws
```

Вывод:
```bash
Name:             data-exchange-77895dc648-lddws
Namespace:        default
Priority:         0
Service Account:  default
Node:             vm/192.168.218.128
Start Time:       Sun, 05 Jul 2026 20:40:17 +0300
Labels:           app=data-exchange
                  pod-template-hash=77895dc648
Annotations:      cni.projectcalico.org/containerID: c556812223dd63813eabffb72d081c0ef5a3ea3eea977447e069c4f190b04d65
                  cni.projectcalico.org/podIP: 10.1.141.126/32
                  cni.projectcalico.org/podIPs: 10.1.141.126/32
Status:           Running
IP:               10.1.141.126
IPs:
  IP:           10.1.141.126
Controlled By:  ReplicaSet/data-exchange-77895dc648
Containers:
  busybox:
    Container ID:  containerd://8336ccba539c7793c65fe43bfcabaa95326b9e0145ed535341c0d9afb0f5fc6a
    Image:         busybox
    Image ID:      docker.io/library/busybox@sha256:fd8d9aa63ba2f0982b5304e1ee8d3b90a210bc1ffb5314d980eb6962f1a9715d
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
      -c
      while true; do date >> /shared/data.txt; sleep 5; done
    State:          Running
      Started:      Sun, 05 Jul 2026 20:40:24 +0300
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /shared from shared-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-lhsq5 (ro)
  multitool:
    Container ID:   containerd://55462da9b3a4b07f8b1e55676a315894980df81279a007b179f5bf4ec7fe6d18
    Image:          wbitt/network-multitool
    Image ID:       docker.io/wbitt/network-multitool@sha256:db2810fe2c8d36db074eab5d98fbf861c8ed55e0786d648d3477b3de9135632e
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sun, 05 Jul 2026 20:40:25 +0300
    Ready:          True
    Restart Count:  0
    Environment:
      HTTP_PORT:  8080
    Mounts:
      /shared from shared-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-lhsq5 (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  shared-storage:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
  kube-api-access-lhsq5:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m54s  default-scheduler  Successfully assigned default/data-exchange-77895dc648-lddws to vm
  Normal  Pulling    2m51s  kubelet            Pulling image "busybox"
  Normal  Pulled     2m47s  kubelet            Successfully pulled image "busybox" in 4.188s (4.188s including waiting). Image size: 2236931 bytes.
  Normal  Created    2m47s  kubelet            Created container: busybox
  Normal  Started    2m47s  kubelet            Started container busybox
  Normal  Pulling    2m47s  kubelet            Pulling image "wbitt/network-multitool"
  Normal  Pulled     2m46s  kubelet            Successfully pulled image "wbitt/network-multitool" in 1.041s (1.041s including waiting). Image size: 96718848 bytes.
  Normal  Created    2m46s  kubelet            Created container: multitool
  Normal  Started    2m46s  kubelet            Started container multitool
```

Подключаеюсь к контейнеру `multitool` и читаю общий файл, в который busybox записывает данные
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl exec data-exchange-77895dc648-lddws -c multitool -- tail -f /shared/data.txt
Sun Jul  5 17:44:59 UTC 2026
Sun Jul  5 17:45:04 UTC 2026
Sun Jul  5 17:45:09 UTC 2026
Sun Jul  5 17:45:14 UTC 2026
Sun Jul  5 17:45:19 UTC 2026
Sun Jul  5 17:45:24 UTC 2026
Sun Jul  5 17:45:29 UTC 2026
Sun Jul  5 17:45:34 UTC 2026
Sun Jul  5 17:45:39 UTC 2026
Sun Jul  5 17:45:44 UTC 2026
Sun Jul  5 17:45:49 UTC 2026
```

## Задание 2

Манифест `pv-pvc.yaml`
```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-local
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/data/pv-shared"
  storageClassName: manual

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-local
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pvc-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: storage-test
  template:
    metadata:
      labels:
        app: storage-test
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ['sh', '-c', 'while true; do date >> /data/shared_data.txt; sleep 5; done']
          volumeMounts:
            - name: my-persistent-storage
              mountPath: /data
        - name: multitool
          image: wbitt/network-multitool
          env:
            - name: HTTP_PORT
              value: "8080"
          volumeMounts:
            - name: my-persistent-storage
              mountPath: /data
      volumes:
        - name: my-persistent-storage
          persistentVolumeClaim:
            claimName: pvc-local
```

Cоздание директории и включения `hostpath-storage`
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# mkdir -p /data/pv-shared

root@vm:/home/bogatyrevam/Desktop/5-storages# chmod 777 /data/pv-shared

root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s enablee hostpath-storage
'enablee' is not a valid MicroK8s subcommand.
Available subcommands are:
	add-node
	addons
	config
	ctr
	dashboard-proxy
	dbctl
	disable
	enable
	helm
	helm3
	images
	istioctl
	join
	kubectl
	leave
	linkerd
	refresh-certs
	remove-node
	reset
	start
	status
	stop
	version
	inspect
root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s enable hostpath-storage
Infer repository core for addon hostpath-storage
Enabling default storage class.
WARNING: Hostpath storage is not suitable for production environments.
         A hostpath volume can grow beyond the size limit set in the volume claim manifest.

deployment.apps/hostpath-provisioner created
storageclass.storage.k8s.io/microk8s-hostpath created
serviceaccount/microk8s-hostpath created
clusterrole.rbac.authorization.k8s.io/microk8s-hostpath created
clusterrolebinding.rbac.authorization.k8s.io/microk8s-hostpath created
Storage will be available soon.
```

Применение манифеста pv-pvc.yaml

```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# nano pv-pvc.yaml

root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl apply -f pv-pvc.yaml
persistentvolume/pv-local created
persistentvolumeclaim/pvc-local created
deployment.apps/pvc-deployment created
```

Вывод `kubectl get pv`,`pvc` и `kubectl get po`
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pv-local   1Gi        RWO            Retain           Bound    default/pvc-local   manual         <unset>                          2m12s

root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl get pvc
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
pvc-local   Bound    pv-local   1Gi        RWO            manual         <unset>                 2m19s

root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl get po
NAME                                             READY   STATUS    RESTARTS      AGE
pvc-deployment-5f9fdcff56-7dbcd                  2/2     Running   0             3m10s
```

Вывод `tail -f` из контейнера `multitool`
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl exec pvc-deployment-5f9fdcff56-7dbcd -c multitool -- tail -f /data/shared_data.txt
Sun Jul  5 18:00:15 UTC 2026
Sun Jul  5 18:00:20 UTC 2026
Sun Jul  5 18:00:25 UTC 2026
Sun Jul  5 18:00:30 UTC 2026
Sun Jul  5 18:00:35 UTC 2026
Sun Jul  5 18:00:40 UTC 2026
Sun Jul  5 18:00:45 UTC 2026
Sun Jul  5 18:00:50 UTC 2026
Sun Jul  5 18:00:55 UTC 2026
Sun Jul  5 18:01:00 UTC 2026
Sun Jul  5 18:01:05 UTC 2026
```

Удаляю `deployment` и `PVC`
Смотрю `kubectl describe pv`
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl delete deploy pvc-deployment
deployment.apps "pvc-deployment" deleted from default namespace

root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl delete pvc pvc-local
persistentvolumeclaim "pvc-local" deleted from default namespace

root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl describe pv pv-local
Name:            pv-local
Labels:          <none>
Annotations:     pv.kubernetes.io/bound-by-controller: yes
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    manual
Status:          Released
Claim:           default/pvc-local
Reclaim Policy:  Retain
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        1Gi
Node Affinity:   <none>
Message:         
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /data/pv-shared
    HostPathType:  
Events:            <none>
```

Смотрю файл на локальном диске ноды
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# cat /data/pv-shared/shared_data.txt 
Sun Jul  5 17:55:10 UTC 2026
Sun Jul  5 17:55:15 UTC 2026
Sun Jul  5 17:55:20 UTC 2026
Sun Jul  5 17:55:25 UTC 2026
Sun Jul  5 17:55:30 UTC 2026
Sun Jul  5 17:55:35 UTC 2026
Sun Jul  5 17:55:40 UTC 2026
Sun Jul  5 17:55:45 UTC 2026
Sun Jul  5 17:55:50 UTC 2026
Sun Jul  5 17:55:55 UTC 2026
Sun Jul  5 17:56:00 UTC 2026
Sun Jul  5 17:56:05 UTC 2026
Sun Jul  5 17:56:10 UTC 2026
Sun Jul  5 17:56:15 UTC 2026
Sun Jul  5 17:56:20 UTC 2026
Sun Jul  5 17:56:25 UTC 2026
Sun Jul  5 17:56:30 UTC 2026
Sun Jul  5 17:56:35 UTC 2026
```

Состояние `PV`
```
Наблюдаемое поведение:
После удаления deployment и PVC команда kubectl get pv показывает, что статус pv-local изменился с Bound на Released.

Пояснение:
Статус Released означает, что запрос (PVC), который удерживал этот объем, был удален. Поскольку в манифесте PV была указана политика persistentVolumeReclaimPolicy: Retain, Kubernetes не удаляет сам объем и данные в нем. Однако PV не возвращается автоматически в статус Available, так как он все еще содержит данные предыдущего пользователя (PVC) и требует ручного вмешательства администратора для повторного использования.
```
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM               STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pv-local   1Gi        RWO            Retain           Released   default/pvc-local   manual         <unset>                          16m
```

Сохраняемость файла на локальном диске ноды
```
Наблюдаемое поведение:
При проверке директории на хост-системе (/data/pv-shared/shared_data.txt) файл остается на месте и содержит все записанные ранее данные.

Пояснение:
Это результат работы политики Retain. Она гарантирует, что физические данные на ноде сохраняются независимо от жизненного цикла объектов PVC в кластере.
```

Удаление PV и состояние файла
```bash
Наблюдаемое поведение:
После удаления объекта PV с помощью kubectl delete pv pv-local, файл на локальном диске по адресу /data/pv-shared/shared_data.txt по-прежнему существует.

Пояснение:
Для типов томов hostPath Kubernetes управляет только объектом в API (самой записью о PV). При удалении ресурса PV из кластера, Kubernetes удаляет только "логическое" описание тома. Удаление фактических данных в локальной директории на ноде не производится. Это механизм защиты от случайной потери данных, требующий от администратора системы ручной очистки дискового пространства на физическом уровне.
```



## Задание 3

1. Манифест `sc.yaml`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path-sc
provisioner: microk8s.io/hostpath
reclaimPolicy: Delete
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sc-pvc
spec:
  storageClassName: local-path-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sc-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sc-test
  template:
    metadata:
      labels:
        app: sc-test
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ['sh', '-c', 'while true; do date >> /data/dynamic_data.txt; sleep 5; done']
          volumeMounts:
            - name: dynamic-storage
              mountPath: /data
        - name: multitool
          image: wbitt/network-multitool
          env:
            - name: HTTP_PORT
              value: "8080"
          volumeMounts:
            - name: dynamic-storage
              mountPath: /data
      volumes:
        - name: dynamic-storage
          persistentVolumeClaim:
            claimName: sc-pvc
```

2. Описание процесса и результаты
В данном задании был реализован механизм динамического выделения ресурсов:

  1. Создан StorageClass, использующий провиженер `microk8s.io/hostpath`.
  2. Создан PVC, который автоматически инициировал создание соответствующего PV (статус `Bound` подтверждает успешную связку).
  3. Развернут deployment с двумя контейнерами, которые используют общий динамический том для обмена данными.

3.1. Применение манифеста `sc.yaml`
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# nano sc.yaml

root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl apply -f sc.yaml
storageclass.storage.k8s.io/local-path-sc created
persistentvolumeclaim/sc-pvc created
deployment.apps/sc-deployment created
```

3.2. Вывод `kubectl get sc, pvc, pv` демонстрирующий автоматическое создание `PV`
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl get sc,pvc,pv
NAME                                                      PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/local-path-sc                 microk8s.io/hostpath   Delete          Immediate              false                  2m48s
storageclass.storage.k8s.io/microk8s-hostpath (default)   microk8s.io/hostpath   Delete          WaitForFirstConsumer   false                  29m

NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/sc-pvc   Bound    pvc-e8eb2fc9-931a-45b4-8b8b-31f792231c1e   500Mi      RWO            local-path-sc   <unset>                 2m47s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM               STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pv-local                                   1Gi        RWO            Retain           Released   default/pvc-local   manual          <unset>                          27m
persistentvolume/pvc-e8eb2fc9-931a-45b4-8b8b-31f792231c1e   500Mi      RWO            Delete           Bound      default/sc-pvc      local-path-sc   <unset>                          2m32s
```

3.3. Вывод `tail -f /data/dynamic_data.txt` из контейнера `multitool`, подтверждающий запись и чтение данных
```bash
root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl get po
NAME                                             READY   STATUS    RESTARTS      AGE
sc-deployment-b546dc667-cnfbf                    2/2     Running   0             5m5s

root@vm:/home/bogatyrevam/Desktop/5-storages# microk8s kubectl exec sc-deployment-b546dc667-cnfbf -c multitool -- tail -f /data/dynamic_data.txt
Sun Jul  5 18:24:55 UTC 2026
Sun Jul  5 18:25:00 UTC 2026
Sun Jul  5 18:25:05 UTC 2026
Sun Jul  5 18:25:10 UTC 2026
Sun Jul  5 18:25:15 UTC 2026
Sun Jul  5 18:25:20 UTC 2026
Sun Jul  5 18:25:25 UTC 2026
Sun Jul  5 18:25:30 UTC 2026
Sun Jul  5 18:25:35 UTC 2026
Sun Jul  5 18:25:40 UTC 2026
```
