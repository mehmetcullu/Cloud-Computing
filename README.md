# Kubernetes Nginx Deployment & Autoscaling Projesi

Bu proje, Kubernetes üzerinde temel yapı taşlarının (**Pod**, **Deployment**, **Service** ve **Horizontal Pod Autoscaler - HPA**) deklaratif yöntemle (YAML dosyaları kullanılarak) nasıl ayağa kaldırılacağını ve yönetileceğini içerir.

## 📁 Proje Dosya Yapısı

Depoda yer alan yapılandırma dosyaları:

* `nginx_pod.yaml`: Tekil bir Nginx Pod tanımı.
* `deployment.yaml`: 3 kopyalı (replica), CPU kaynak limitli Nginx Deployment tanımı.
* `service.yaml`: Uygulamayı dış dünyaya açan `NodePort` servisi.
* `hpa.yaml`: CPU kullanımına göre Pod sayısını otomatik ölçeklendiren (1-10 arası) HPA tanımı.

## 🛠️ Ön Koşullar

Bu projeyi yerel ortamınızda çalıştırmak için aşağıdaki araçların kurulu olması gerekir:

* **Docker Desktop** (Settings > Kubernetes menüsünden *"Enable Kubernetes"* aktif edilmiş) veya **Minikube**
* **kubectl** (Kubernetes komut satırı aracı)

### 1. macOS için `kubectl` Kurulumu

Terminal üzerinden Homebrew ile kurulum:

```bash
brew install kubectl
```

### 2. Windows için `kubectl` Kurulumu

PowerShell veya Komut İstemi (CMD) üzerinden **Winget** veya **Chocolatey** kullanarak kurabilirsiniz (Docker Desktop kuruluysa `kubectl` genellikle beraberinde hazır gelir):

```powershell
# Winget ile kurulum (Önerilen)
winget install -e --id Kubernetes.kubectl

# Veya Chocolatey ile kurulum
choco install kubernetes-cli
```

### Küme Doğrulaması (macOS & Windows)

Kurulum sonrası kümenin çalıştığını doğrulamak için:

```bash
kubectl cluster-info
kubectl get nodes
```

## 🚀 Kurulum ve Çalıştırma Adımları

### 1. Tekil Pod Oluşturma (Opsiyonel Test)

Sadece tek bir Nginx pod'u ayağa kaldırmak için:

```bash
kubectl apply -f nginx_pod.yaml
kubectl get pods
```

### 2. Deployment Oluşturma

Uygulamayı 3 kopya (replica) olarak çalıştırmak ve self-healing (kendi kendini iyileştirme) özelliğini aktif etmek için:

```bash
kubectl apply -f deployment.yaml
```

Durumu kontrol etmek için:

```bash
kubectl get deployments
kubectl get pods
```

### 3. Servis (NodePort) ile Dışarıya Açma

Pod'ları sabit bir port üzerinden dış dünyaya açmak ve yük dengeleme (load balancing) sağlamak için:

```bash
kubectl apply -f service.yaml
kubectl get services
```

Uygulamaya tarayıcı üzerinden erişmek için:

* **Docker Desktop (macOS & Windows):** `http://localhost:30007`
* **Minikube:** `minikube service nginx-service --url`

## 📈 Horizontal Pod Autoscaler (HPA) Kurulumu

HPA'nın CPU kullanımını okuyabilmesi için kümede **Metrics Server** kurulu olmalıdır.

### 1. Metrics Server Kurulumu (Yerel Küme İçin)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Yerel geliştirme ortamında TLS sertifika hatasını aşmak için işletim sisteminize uygun yamayı (patch) uygulayın:

**macOS / Linux Terminali için:**
```bash
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

**Windows (PowerShell / CMD) için:**
```powershell
kubectl patch deployment metrics-server -n kube-system --type="json" -p="[{\"op\": \"add\", \"path\": \"/spec/template/spec/containers/0/args/-\", \"value\": \"--kubelet-insecure-tls\"}]"
```

Metriklerin geldiğini doğrulayın (aktif olması 30-60 saniye sürebilir):

```bash
kubectl top nodes
```

### 2. HPA'yı Aktif Etme

CPU kullanımı %80'i geçtiğinde pod sayısını maksimum 10'a kadar çıkaran, yük olmadığında minimum 1'e düşüren HPA'yı başlatın:

```bash
kubectl apply -f hpa.yaml
kubectl get hpa
```

### 3. Yük Testi (Autoscaling Testi)

Otomatik ölçeklemeyi test etmek için bir terminalde HPA'yı izlemeye alın:

```bash
kubectl get hpa -w
```

Yeni bir terminal (macOS Terminal veya Windows PowerShell) penceresi açarak servise sürekli istek gönderen geçici bir yük pod'u başlatın:

```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://nginx-service:8080; done"
```

## 📄 YAML Dosya İçerikleri

Depodaki dosyaların içerikleri aşağıdaki gibidir:

**`nginx_pod.yaml`**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: webserver
    image: nginx:latest
    ports:
    - containerPort: 80
```

**`deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
```

**`service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 8080
    targetPort: 80
    nodePort: 30007
```

**`hpa.yaml`**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

## 🧹 Temizlik

Oluşturulan tüm kaynakları kümeden silmek için:

```bash
kubectl delete -f hpa.yaml
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
kubectl delete -f nginx_pod.yaml
```