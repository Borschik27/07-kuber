# 07-kuber

Задачи реализованны на основе поднятого кластера kubernetes в репозитории 06-kuber

# Задача 1

Манифест для pv:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: /mnt/data_nfs
```
![image](https://github.com/user-attachments/assets/b98d2886-74fb-467d-a69e-9d9e6a1cb3d1)


Манифест pvc: 
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1G
  storageClassName: local-storage
```
![image](https://github.com/user-attachments/assets/23028a18-89e6-4fcf-a1c4-e525c0b0be95)

Манифест deployment :
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox-multitool
  template:
    metadata:
      labels:
        app: busybox-multitool
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c", "while true; do echo $(date) >> /data/shared-file.txt; sleep 5; done"]
        volumeMounts:
        - name: shared-data
          mountPath: /data
      - name: multitool
        image: praqma/network-multitool
        command: ["/bin/sh", "-c", "tail -f /data/shared-file.txt"]
        volumeMounts:
        - name: shared-data
          mountPath: /data
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: local-pvc
```

Зайдем в mutltitool и проверим что пишется в /data:
![image](https://github.com/user-attachments/assets/aeec5af4-1d6f-4625-927f-90e7b75ad27f)

Зайдем в bb и проверим что его задача тоже работает:
![image](https://github.com/user-attachments/assets/1402b54d-a618-4684-93e1-5ab1280951ce)

Зайдем на worker01 и посмотрик есть ли каталог /mnt/data_nfs:

![image](https://github.com/user-attachments/assets/74022427-ddb6-444d-8c07-d091aebe4089)

После удаления pv он зависает в состоянии Terminating это связано с тем пытается очистить PV, И PVC еще активно
Удалив pvc удаляется и pv при этом файл остался на хосте worker01
![image](https://github.com/user-attachments/assets/a903ec5f-228d-4d42-a5d5-4a6a5a480010)


# Задача 2

Установили NSF сервер на kuber и настроили его:
![image](https://github.com/user-attachments/assets/ad8e3895-f363-48e0-a110-796274a0f30c)

Содадим тестовый файл для проверки что на нодах виден файл:
![image](https://github.com/user-attachments/assets/00009162-bf97-4cc8-bad9-a4571cb2800b)
![image](https://github.com/user-attachments/assets/eebd811f-dca0-402f-bf31-75d8699bac9d)

Workers:
![image](https://github.com/user-attachments/assets/f37a8345-fa6a-40d8-87de-0947b69a7ebb)

![image](https://github.com/user-attachments/assets/fa525842-4bef-435f-b1c9-470eda2fa29e)

Создадим манифесты:
PV:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  nfs:
    path: /nfs
    server: 10.1.1.31
```
![image](https://github.com/user-attachments/assets/63610329-8e5c-4161-a061-c599c93323eb)

PVC:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: manual
```
![image](https://github.com/user-attachments/assets/e1833bae-29c2-4c83-a69a-c1b1ca1a9eb3)

Deployment:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox-multitool
  template:
    metadata:
      labels:
        app: busybox-multitool
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c", "while true; do echo $(date) >> /data/shared-file.txt; sleep 20; done"]
        volumeMounts:
        - name: shared-data
          mountPath: /data
      - name: multitool
        image: praqma/network-multitool
        command: ["/bin/sh", "-c", "tail -f /data/shared-file.txt"]
        volumeMounts:
        - name: shared-data
          mountPath: /data
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: nfs-pvc
```
![image](https://github.com/user-attachments/assets/06edf01f-1881-479f-af93-401be827078a)

ПРоверим на сервере NFS (kuber01) Что запись в NFS идет:
![image](https://github.com/user-attachments/assets/2e7fe9dc-cbf2-46c0-bd5f-41dc7adc3459)


