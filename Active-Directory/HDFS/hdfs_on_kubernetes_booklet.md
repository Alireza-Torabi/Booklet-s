
# راهنمای نصب HDFS روی Kubernetes

## معرفی HDFS و کاربرد آن
HDFS (Hadoop Distributed File System) یک سیستم فایل توزیع‌شده قدرتمند برای ذخیره‌سازی و دسترسی به داده‌های بسیار بزرگ است. این سیستم با طراحی خاص خود امکان ذخیره‌سازی فایل‌های پتابایتی را روی سخت‌افزار معمولی فراهم می‌کند و الگوی دسترسی به داده‌ها را به صورت **write-once/read-many** (یکبار نوشتن و چندبار خواندن) فراهم می‌نماید.

در معماری HDFS یک **NameNode (نام‌گره)** به عنوان گره اصلی مسئول متادیتای فایل‌ها (موقعیت بلوک‌ها، نام‌ها و…) است و چندین **DataNode (گره داده)** مسئول نگهداری بلوک‌های واقعی داده می‌باشند. هر فایل بزرگ به بلوک‌های کوچک تقسیم می‌شود و روی چندین DataNode قرار می‌گیرد؛ همچنین بلوک‌ها چندین بار روی DataNodeهای مختلف تکثیر می‌شوند تا در صورت خرابی یک گره، داده‌ها حفظ شوند.

## معرفی Kubernetes و مفاهیم پایه
Kubernetes یک پلتفرم متن‌باز برای خودکارسازی «استقرار، مقیاس‌دهی و مدیریت برنامه‌های کانتینری» است. در این محیط، کوچک‌ترین واحد اجرایی **Pod** نام دارد که شامل یک یا چند کانتینر مرتبط است.

برای دسترسی پایدار به Podها از **Service** استفاده می‌شود. برای برنامه‌های حالت‌دار مانند HDFS، Kubernetes از **StatefulSet** استفاده می‌کند که برای هر Pod هویتی ثابت (نام، شبکه، فضای ذخیره‌سازی) فراهم می‌سازد.

**Helm** نیز ابزار مدیریت بسته Kubernetes است که نصب و حذف برنامه‌ها را بسیار ساده‌تر می‌کند.

## پیش‌نیازها
- Kubernetes Cluster (مثلاً با Minikube)
- kubectl
- Helm
- StorageClass فعال

## نصب HDFS با StatefulSet
1. ایجاد Namespace:
```bash
kubectl create namespace bigdata
```

2. ایجاد ConfigMap تنظیمات Hadoop:
```bash
kubectl -n bigdata create configmap hadoop-config --from-file=core-site.xml --from-file=hdfs-site.xml
```

3. تعریف Service برای NameNode و سپس StatefulSet آن.

4. تعریف StatefulSet برای DataNode.

5. اعمال YAMLها:
```bash
kubectl apply -f <file>.yaml
```

6. بررسی وضعیت:
```bash
kubectl get pods -n bigdata
```

## نصب HDFS با Helm
```bash
helm repo add gradiant-bigdata https://gradiant.github.io/bigdata-charts
helm repo update
helm install my-hdfs gradiant-bigdata/hdfs --version 0.1.10
```

## تست عملکرد
```bash
kubectl exec -n bigdata -it namenode-0 -- hdfs dfsadmin -report
kubectl exec -n bigdata -it namenode-0 -- hdfs dfs -mkdir /test
kubectl exec -n bigdata -it namenode-0 -- hdfs dfs -put localfile.txt /test
kubectl exec -n bigdata -it namenode-0 -- hdfs dfs -ls /test
```

## پاک‌سازی منابع
```bash
helm uninstall my-hdfs
kubectl delete statefulset namenode datanode -n bigdata
kubectl delete pvc --all -n bigdata
kubectl delete namespace bigdata
```