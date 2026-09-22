
Readme · MD
🎬 Netflix Clone — DevSecOps Pipeline Projesi
Bu proje, bir React/TypeScript tabanlı Netflix klonu uygulamasını Azure bulut altyapısı üzerinde, uçtan uca bir DevSecOps hattı kurarak geliştirme, güvenlik taraması, sürekli entegrasyon/dağıtım (CI/CD), izleme ve container orkestrasyonu pratikleriyle production benzeri bir ortamda çalıştırma deneyimidir.

Amaç sadece "uygulamayı ayağa kaldırmak" değil; modern bir yazılım ekibinin kod yazdıktan sonra o kodu güvenli, izlenebilir ve otomatik şekilde canlıya taşımak için hangi araçları, neden ve nasıl kullandığını uçtan uca deneyerek öğrenmekti.

Kaynak proje: N4si/DevSecOps-Project (React + TypeScript + Vite, TMDB API kullanan bir Netflix klonu)

📐 Mimari Genel Bakış
Projede iki ayrı çalışma ortamı bir araya geldi:

Tek bir Azure Ubuntu VM — Docker, Jenkins, SonarQube, Trivy, Prometheus ve Grafana bu VM üzerinde bir arada çalıştı. Gerçek bir kurumsal ortamda bu servisler genelde ayrı sunucularda/servislerde barındırılır; biz öğrenme amaçlı hepsini tek makinede topladık.
Azure Kubernetes Service (AKS) — VM'den tamamen bağımsız, Azure'un yönetilen Kubernetes cluster'ı. Buraya VM üzerinden değil, Azure Cloud Shell üzerinden bağlanıldı.
mermaid
flowchart TD
    A[Geliştirici<br/>git push] --> B[Jenkins<br/>CI/CD Orkestrasyonu]
    B --> C[SonarQube<br/>Kod Kalitesi ve Statik Analiz]
    B --> D[Trivy<br/>Dosya Sistemi ve Container Taraması]
    B --> E[Docker<br/>Build ve Image Oluşturma]
    E --> F[(DockerHub<br/>Image Registry)]
    F --> G[Docker Container<br/>VM üzerinde, port 8081]
    F --> H[Kubernetes - AKS<br/>LoadBalancer ile Dış IP]
    B --> I[Prometheus + Grafana<br/>İzleme / Monitoring]
    B --> J[E-posta Bildirimi<br/>Gmail SMTP ile build sonucu]
🧰 Kullanılan Araçlar ve Neden Kullanıldıkları
Her araç, DevSecOps zincirinde belirli bir problemi çözmek için seçildi. "Ne işe yaradığını" değil, "hangi soruna cevap verdiğini" anlamak asıl amaçtı:

Docker
Sorun: "Benim makinemde çalışıyordu ama sunucuda çalışmadı" klasik sorunu. Çözüm: Uygulamayı ve tüm bağımlılıklarını (Node.js runtime, build çıktısı, nginx) tek bir taşınabilir pakete (image) koyduk. Bu image VM'de, Kubernetes'te veya başka herhangi bir Docker uyumlu ortamda birebir aynı şekilde çalışır. Container'lar sanal makinelerden farklı olarak işletim sistemi çekirdeğini paylaşır, bu da onları çok daha hafif ve hızlı başlatılabilir yapar.

Jenkins
Sorun: Kod her değiştiğinde "test et, tara, build al, deploy et" adımlarını elle tek tek yapmak hem yavaş hem hataya açık. Çözüm: Jenkins, bu adımların hepsini tanımlı bir "pipeline" (Groovy scripti) içinde otomatikleştirir. Bu projenin CI/CD omurgasıdır — her şey Jenkins'ten tetiklenir ve sırayla yürütülür.

SonarQube
Sorun: Kodun içinde göze çarpmayan kalite ve güvenlik sorunları (kötü pratikler, tekrar eden bloklar, potansiyel bug'lar, gömülü secret'lar) olabilir. Çözüm: SonarQube statik kod analizi yaparak bu sorunları otomatik tespit eder ve bir "Quality Gate" (kalite eşiği) tanımlayarak yetersiz kaliteli kodun ilerlemesini durdurabilir. Biz bu projede quality gate'i bilgilendirme amaçlı kullandık (pipeline'ı durdurmadı).

OWASP Dependency-Check
Sorun: SonarQube kendi yazdığınız koda bakar, ama siz üçüncü parti kütüphaneler (npm paketleri) de kullanıyorsunuz — onların içinde bilinen güvenlik açıkları olabilir. Çözüm: Bu araç, projenin bağımlılıklarını (package.json/yarn.lock) tarayıp bilinen CVE (Common Vulnerabilities and Exposures) veritabanıyla karşılaştırır. Not: Bu projede NVD API anahtarı gereksinimi ve veri güncelleme sorunları nedeniyle pipeline'dan çıkarmak zorunda kaldık — gerçek kullanımda ücretsiz bir NVD API key alınıp eklenmesi gerekir.

Trivy
Sorun: Hem bağımlılıklarda hem de container image'ının içindeki işletim sistemi paketlerinde (Alpine Linux gibi) güvenlik açığı olabilir. Çözüm: Trivy, OWASP'a benzer şekilde dosya sistemini tarar, ama ayrıca oluşturduğumuz Docker image'ının içine de bakıp "bu image'ı production'a göndermeden önce içinde bilinen bir açık var mı" sorusuna cevap verir. DevSecOps'taki "Sec" (güvenlik) kısmının en somut karşılığı budur.

Prometheus
Sorun: Sistemin ne durumda olduğunu (CPU, RAM, disk kullanımı, build sayıları) bilmeden bir sorunu teşhis etmek mümkün değildir. Çözüm: Prometheus, tanımlı hedeflerden (VM, Jenkins) periyodik olarak metrik toplayıp zaman serisi veritabanında saklar. "Neden yavaşladı?" sorusuna cevap verebilmenin ön koşulu bu veriyi toplamaktır.

Grafana
Sorun: Ham sayısal veri (Prometheus'un topladığı) insan gözü için okunabilir değildir. Çözüm: Grafana, bu veriyi grafiklere ve dashboard'lara dönüştürür. Prometheus veriyi toplar, Grafana o veriyi anlamlı hale getirir — ikisi birbirini tamamlayan bir çift olarak kullanılır.

Jenkins Extended E-mail Notification + Gmail SMTP
Sorun: Pipeline'ı sürekli ekranda izlemek pratik değil; özellikle başarısız build'lerde hızlı haber almak gerekir. Çözüm: Build bittiğinde (başarılı ya da başarısız) otomatik e-posta gönderimi kurduk. "Sessiz kalan hata en tehlikeli hatadır" prensibiyle, insanı döngüye geri kattık.

Kubernetes (Azure Kubernetes Service — AKS)
Sorun: Docker tek bir container'ı çalıştırmak için yeterlidir, ama gerçek production'da trafik arttığında birden fazla kopya çalıştırmak, biri çökerse otomatik yeniden başlatmak ve yük dengelemesi yapmak gerekir. Çözüm: Kubernetes, container'ları orkestre eden bir katmandır. Biz burada kendi cluster'ımızı kurmak yerine Azure'un yönetilen servisi olan AKS'yi kullandık; böylece node yönetimi, güncellemeler gibi altyapı işleriyle kendimiz uğraşmadık.

TMDB API
Sorun: Uygulamanın kendi film/dizi veritabanı yok. Çözüm: The Movie Database (TMDB) API'sinden gerçek zamanlı olarak film/dizi bilgisi (poster, açıklama, puan) çekildi. Bu yüzden bir API key gerekiyordu — API key'ler build zamanında --build-arg ile Docker image'ına enjekte edildi.

🚀 Adım Adım Ne Yaptık
Faz 1 — Azure VM Kurulumu ve Docker ile İlk Deploy
Azure Portal'da Ubuntu 22.04 LTS, Standard_B2s (2 vCPU / 4 GB RAM) bir VM oluşturuldu.
Network Security Group (NSG) üzerinden gerekli portlar açıldı:
Port	Servis
22	SSH
80	HTTP
8080	Jenkins
8081	Uygulama (Docker)
9000	SonarQube
9090	Prometheus
3000	Grafana
SSH ile bağlanıldı:
bash
   ssh azureuser@<VM_PUBLIC_IP>
Docker kuruldu:
bash
   sudo apt-get update
   sudo apt-get install docker.io -y
   sudo usermod -aG docker $USER
   newgrp docker
   sudo chmod 777 /var/run/docker.sock
Proje klonlandı ve TMDB API key alındı:
bash
   git clone https://github.com/N4si/DevSecOps-Project.git
   cd DevSecOps-Project
Docker image build edilip container çalıştırıldı:
bash
   docker build --build-arg TMDB_V3_API_KEY=<API_KEY> -t netflix .
   docker run -d --name netflix -p 8081:80 netflix:latest
Dockerfile'ın mantığı (multi-stage build):

dockerfile
FROM node:16.17.0-alpine as builder      # 1. aşama: uygulamayı derle
WORKDIR /app
COPY ./package.json .
COPY ./yarn.lock .
RUN yarn install
COPY . .
ARG TMDB_V3_API_KEY
ENV VITE_APP_TMDB_V3_API_KEY=${TMDB_V3_API_KEY}
ENV VITE_APP_API_ENDPOINT_URL="https://api.themoviedb.org/3"
RUN yarn build

FROM nginx:stable-alpine                  # 2. aşama: sadece derlenmiş çıktıyı al
WORKDIR /usr/share/nginx/html
RUN rm -rf ./*
COPY --from=builder /app/dist .
EXPOSE 80
ENTRYPOINT ["nginx", "-g", "daemon off;"]
Bu multi-stage build tekniği önemlidir: ilk aşamada Node.js ile kodu derliyoruz, ama son image'da Node.js'e ihtiyacımız yok — sadece derlenmiş statik dosyaları nginx ile sunuyoruz. Bu, final image'ı çok daha küçük ve güvenli (daha az saldırı yüzeyi) yapar. Nitekim Trivy taramasında bu image'da 0 güvenlik açığı çıktı.

Faz 2 — Güvenlik Taraması: SonarQube ve Trivy
bash
# SonarQube (Docker container olarak)
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

# Trivy kurulumu
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install trivy -y
Bulgular:

Dosya sistemi taraması (trivy fs .): yarn.lock içinde 34 açık (21 HIGH, 11 MEDIUM, 2 LOW) — özellikle eski @xmldom/xmldom, brace-expansion, react-router versiyonlarında.
Docker image taraması (trivy image netflix:latest): 0 açık — minimal Alpine tabanlı image'ın avantajı.
Bu ikisi arasındaki fark önemli bir öğrenim: kod bağımlılıklarınız güncel olmayabilir, ama iyi tasarlanmış bir container image bu riski kısmen izole edebilir.

Faz 3 — Jenkins CI/CD Pipeline
Java, Jenkins ve gerekli plugin'ler (SonarQube Scanner, NodeJS, Docker Pipeline, OWASP Dependency-Check vb.) kuruldu. SonarQube token'ı ve DockerHub kimlik bilgileri Jenkins'e credential olarak eklendi.

Nihai pipeline (Jenkinsfile mantığı):

groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps { cleanWs() }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/N4si/DevSecOps-Project.git'
                // Node 16 ile bazı bağımlılıkların engine uyumsuzluğunu aşmak için:
                sh "sed -i 's/RUN yarn install/RUN yarn install --ignore-engines/' Dockerfile"
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage("quality gate") {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    script {
                        waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                    }
                }
            }
        }
        stage('Install Dependencies') {
            steps { sh "npm install" }
        }
        stage('TRIVY FS SCAN') {
            steps { sh "trivy fs . > trivyfs.txt" }
        }
        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build --build-arg TMDB_V3_API_KEY=<API_KEY> -t netflix ."
                        sh "docker tag netflix <dockerhub-user>/netflix:latest"
                        sh "docker push <dockerhub-user>/netflix:latest"
                    }
                }
            }
        }
        stage("TRIVY Image Scan") {
            steps { sh "trivy image <dockerhub-user>/netflix:latest > trivyimage.txt" }
        }
        stage('Deploy to container') {
            steps {
                sh 'docker stop netflix || true && docker rm netflix || true'
                sh 'docker run -d --name netflix -p 8081:80 <dockerhub-user>/netflix:latest'
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>Build Number: ${env.BUILD_NUMBER}<br/>URL: ${env.BUILD_URL}<br/>",
                to: '<email-adresiniz>',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
Her stage'in ne işe yaradığı:

clean workspace → önceki build'den kalan dosyaları temizler, her build'in temiz bir zeminde başlamasını sağlar.
Checkout from Git → kaynak kodu çeker; sed komutu ise Node 16'nın bazı modern paketlerle (engine gereksinimi >=18 olan) uyumsuzluğunu build'i bozmadan aşmak için Dockerfile'ı otomatik yamalar.
Sonarqube Analysis → statik kod analizini tetikler ve sonucu SonarQube sunucusuna yükler.
quality gate → SonarQube'un analizi bitirip bir "geçti/geçmedi" kararı vermesini bekler; timeout bloğu, SonarQube'un webhook ile geri dönmemesi durumunda pipeline'ın sonsuza kadar askıda kalmasını önler.
Install Dependencies → npm install ile bağımlılıkları kurar (sonraki analiz adımları için gerekli).
TRIVY FS SCAN → bağımlılıklardaki bilinen açıkları dosyaya yazar.
Docker Build & Push → image'ı build eder, DockerHub'a yükler (withDockerRegistry credential'ları güvenli şekilde enjekte eder).
TRIVY Image Scan → DockerHub'a gönderilen image'ın son halini tarar.
Deploy to container → eski container'ı temizler (|| true ile "container zaten yoksa hata verme" garantisi), yenisini ayağa kaldırır.
post { always { emailext ... } } → build sonucu ne olursa olsun (başarı/başarısız) bir e-posta gönderir, tarama raporlarını ek olarak yollar.
Faz 4 — İzleme: Prometheus ve Grafana
Prometheus ve Node Exporter, systemd servisleri olarak kuruldu (Docker container olarak değil — bunun nedeni sistem seviyesi metrikleri (CPU, disk, ağ) toplarken host makineye doğrudan erişimin gerekmesiydi).

Prometheus yapılandırması (prometheus.yml):

yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
        - localhost:9090

  - job_name: node_exporter
    static_configs:
      - targets:
        - localhost:9100

  - job_name: jenkins
    metrics_path: /prometheus
    static_configs:
      - targets:
        - localhost:8080
Bu dosya Prometheus'a "hangi adreslerden, hangi aralıklarla metrik topla" der. Üç farklı job_name — kendi kendini izleme (prometheus), sistem metrikleri (node_exporter) ve CI/CD metrikleri (jenkins) — tek bir dashboard'da birleştirilebilir hale geldi.

Grafana kurulup Prometheus veri kaynağı olarak eklendi, ardından hazır bir community dashboard (ID: 1860 — Node Exporter Full) import edilerek CPU, RAM, disk kullanımı gibi metrikler görselleştirildi.

Faz 5 — E-posta Bildirimi
Gmail hesabında 2 adımlı doğrulama açılıp bir App Password oluşturuldu (Gmail normal şifrenizle SMTP erişimine izin vermez, bu yüzden ayrı bir uygulama şifresi gerekir — hesap güvenliği için önemli bir pratiktir). Jenkins'in Extended E-mail Notification eklentisi bu bilgilerle SMTP sunucusuna (smtp.gmail.com:465, SSL) bağlandı ve pipeline'ın post bloğuna eklenen emailext adımı her build sonunda otomatik e-posta gönderdi.

Faz 6 — Kubernetes (Azure Kubernetes Service)
Bu faz, VM'den ayrı çalışır. Azure Portal üzerinden bir AKS cluster'ı (netflix-cluster) oluşturuldu, ardından Azure Cloud Shell (tarayıcı içi terminal, kubectl önceden kurulu) üzerinden bağlanıldı:

bash
az account set --subscription <subscription-id>
az aks get-credentials --resource-group <resource-group> --name netflix-cluster --overwrite-existing
kubectl get nodes
Proje içindeki hazır manifest dosyaları uygulandı:

bash
kubectl apply -f https://raw.githubusercontent.com/N4si/DevSecOps-Project/main/Kubernetes/deployment.yml
kubectl apply -f https://raw.githubusercontent.com/N4si/DevSecOps-Project/main/Kubernetes/service.yml
Karşılaşılan ve çözülen sorunlar:

Service tipi varsayılan olarak NodePort geldi, dış dünyadan erişim için LoadBalancer olması gerekiyordu:
bash
   kubectl patch svc netflix-app -p '{"spec": {"type": "LoadBalancer"}}'
Deployment, orijinal proje sahibinin DockerHub image'ını (nasi101/netflix:latest) referans veriyordu; kendi image'ımızla güncellendi:
bash
   kubectl set image deployment/netflix-app netflix-app=<dockerhub-user>/netflix:latest
Sonuç olarak Azure otomatik bir external IP atadı ve uygulama bu IP üzerinden tarayıcıdan erişilebilir hale geldi.

🧠 Bu Projeden Çıkan Öğrenimler
CI/CD pipeline'ları gerçek hayatta nadiren ilk denemede çalışır. Bu projede karşılaşılan hemen her hata (Java sürüm uyumsuzluğu, GPG key sorunları, Docker izin hataları, Node engine uyumsuzluğu, NVD API key eksikliği, Kubernetes service tipi/image adı uyumsuzlukları) gerçek dünyada da sıkça rastlanan sınıflardan. Hata mesajlarını okuyup adım adım izole etmek, "her şeyi yeniden kurmak"tan çok daha etkili.
Güvenlik taraması "tek seferlik" bir kontrol değil, sürekli bir süreç. SonarQube, Trivy ve OWASP farklı katmanlarda (kod, bağımlılık, image) çalışıyor — hiçbiri tek başına yeterli değil, üçü birlikte daha bütünsel bir güvence sağlıyor.
Container image tasarımı doğrudan güvenliği etkiler. Multi-stage build ve minimal base image (Alpine) kullanımı, image'daki açık sayısını doğrudan azaltıyor — bu proje bunu somut olarak gösterdi (yarn.lock'ta 34 açık varken final image'da 0).
İzleme (observability) olmadan operasyon kördür. Prometheus + Grafana kurulmadan önce sistemin sağlığı hakkında hiçbir görünürlük yoktu; kurulduktan sonra CPU/RAM/disk trendleri anlık olarak izlenebilir hale geldi.
Yönetilen bulut servisleri (AKS gibi) altyapı yükünü azaltıyor. Kendi Kubernetes cluster'ımızı VM üzerinde kurmak yerine AKS kullanmak, node yönetimi/güncelleme gibi karmaşık işleri Azure'a devretmemizi sağladı.
Secrets yönetimi kritik. API key'ler ve DockerHub/Gmail kimlik bilgileri Jenkins credential store üzerinden yönetildi; bu, hassas bilgilerin pipeline kodunun içine (ve dolayısıyla Git geçmişine) gömülmesini önledi.
🔮 Nasıl Geliştirilebilir?
Secrets için gerçek bir secret manager kullanılabilir. TMDB API key gibi değerler şu an pipeline script'inde açık şekilde duruyor; Azure Key Vault veya Jenkins Credentials Binding ile daha güvenli hale getirilebilir.
OWASP Dependency-Check yeniden aktif edilebilir. Ücretsiz bir NVD API key alınıp Jenkins credential olarak eklenirse bu tarama katmanı geri kazanılabilir.
Infrastructure as Code (IaC) kullanılabilir. Şu an VM, NSG kuralları ve AKS cluster'ı Azure Portal üzerinden manuel oluşturuldu; Terraform veya Bicep ile bu adımlar kod olarak versiyonlanıp tekrarlanabilir hale getirilebilir.
Pipeline, GitHub webhook ile otomatik tetiklenebilir. Şu an build'ler manuel "Build Now" ile başlatılıyor; bir GitHub webhook eklenerek her git push'ta pipeline otomatik tetiklenebilir.
Kubernetes deployment'ı Jenkins pipeline'ına entegre edilebilir. Şu an Kubernetes adımları manuel Cloud Shell üzerinden yapıldı; withKubeConfig adımıyla pipeline'ın kendisi de cluster'a deploy edebilir hale getirilebilir (orijinal projede bu zaten örnekleniyor).
ArgoCD ile GitOps yaklaşımına geçilebilir. Kubernetes manifest'lerindeki her değişiklik otomatik olarak cluster'a senkronize edilerek "declarative" bir deployment modeline geçilebilir.
Yatay ölçeklendirme (autoscaling) denenebilir. AKS üzerinde Horizontal Pod Autoscaler tanımlanarak trafiğe göre pod sayısının otomatik artıp azalması test edilebilir.
HTTPS/TLS eklenebilir. Şu an tüm servisler düz HTTP üzerinden erişiliyor; bir reverse proxy (nginx/Traefik) ve Let's Encrypt sertifikası ile trafik şifrelenebilir.
Log toplama merkezi hale getirilebilir. Prometheus metrik toplarken loglar dağınık durumda; ELK Stack (Elasticsearch, Logstash, Kibana) veya Loki eklenerek log ve metrik izlemesi bütünleşik hale getirilebilir.
🔗 Erişim Noktaları
Servis	Adres
Uygulama (Docker)	http://<VM_IP>:8081
Uygulama (Kubernetes)	http://<AKS_EXTERNAL_IP>
Jenkins	http://<VM_IP>:8080
SonarQube	http://<VM_IP>:9000
Prometheus	http://<VM_IP>:9090
Grafana	http://<VM_IP>:3000
📚 Kaynak
Bu proje, N4si/DevSecOps-Project reposundaki adımlar takip edilerek Azure üzerinde uçtan uca uygulandı.



