# 📦 클라우드 보안 아키텍처 관점에서의 Private EC2 Jenkins 환경 구축

## ✅ 프록시 기반 Private EC2 Jenkins 설치

- **Private EC2**는 보안 아키텍처 상 **외부 인터넷과 직접 통신할 수 없음**.
- 따라서 Public EC2에 설치한 `apt-cacher-ng`를 **APT 프록시 서버**로 활용하여,  
  Private EC2가 패키지 설치 시 해당 프록시를 통해 간접적으로 외부와 통신하게 구성함.

---

### 1. 🌐 Public EC2 (APT 프록시 서버) 설정

```bash
# 패키지 목록 업데이트 및 프록시 서버 설치
sudo apt update
sudo apt install apt-cacher-ng -y
```

📌 **보안 그룹 설정**:  
Public EC2 인스턴스에 대해 **포트 3142 (apt-cacher-ng 포트)** 를 허용해야 합니다.

---

### 2. 🔒 Private EC2에서 APT 프록시 사용 설정

```bash
# Public EC2의 IP를 프록시로 지정
echo 'Acquire::http::Proxy "http://<proxy_ip>:3142";' | sudo tee /etc/apt/apt.conf.d/01proxy

# 프록시를 통해 패키지 설치 테스트
sudo apt update
```

📝 **왜 필요한가?**  
Private EC2는 외부로 직접 연결되지 않으므로, APT 명령어를 사용하려면  
반드시 중간 프록시 서버를 거쳐야 설치 및 업데이트가 가능함.

---

### 3. 🐳 Private EC2에 Docker 설치 및 Docker 자체 프록시 설정

```bash
sudo apt install docker.io -y

sudo mkdir -p /etc/systemd/system/docker.service.d
cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://<proxy_ip>:3142/"
Environment="HTTPS_PROXY=http://<proxy_ip>:3142/"
Environment="NO_PROXY=localhost,127.0.0.1"
EOF

sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart docker
```

🔍 Docker는 자체 데몬(`dockerd`)을 통해 이미지를 다운로드하므로, 시스템 수준 프록시 외에도 **Docker 전용 프록시 설정이 별도로 필요**합니다.

---

### 4. 🚀 Docker Jenkins 실행 (Private EC2 내부)

```bash
sudo docker run -d \
  -p 8081:8080 \
  --name jenkins \
  jenkins/jenkins:lts-jdk17
```

---

### 5. 🌐 Public EC2에 NGINX 리버스 프록시로 Jenkins 외부 노출

- 외부에서 Jenkins에 접속하기 위해 Public EC2에 NGINX를 설치하고 프록시 구성 필요  
- (이 부분은 추후 보안 설정과 함께 포함 가능)

---

### 6. ⚙️ Jenkins 자체 프록시 설정 (`proxy.xml`)

```bash
cat <<EOF > /var/jenkins_home/proxy.xml
<proxy>
  <name><proxy_ip></name>
  <port>3142</port>
  <userName></userName>
  <password></password>
  <noProxyHost>localhost,127.0.0.1</noProxyHost>
</proxy>
EOF

docker restart jenkins
```

📌 Jenkins 자체 HTTP 클라이언트는 환경 변수 외에도 `proxy.xml` 설정이 있어야  
**플러그인 설치, 업데이트 서버 연결 등**이 제대로 작동함.

---

### 7. ⚒️ Gradle 프록시 설정 (`gradle.properties`)

```bash
mkdir -p ~/.gradle
cat <<EOF > ~/.gradle/gradle.properties
systemProp.http.proxyHost=<proxy_ip>
systemProp.http.proxyPort=3142
systemProp.https.proxyHost=<proxy_ip>
systemProp.https.proxyPort=3142
systemProp.http.nonProxyHosts=localhost,127.0.0.1
EOF
```

📌 Jenkins 내에서 `./gradlew build` 같은 Gradle Wrapper 명령을 실행할 때,  
Gradle은 자체적으로 프록시 설정이 되어 있어야 정상적으로 작동합니다.

---

## ✅ 결론

- Private EC2 환경에서는 **인터넷을 직접 사용할 수 없기 때문에**,  
  **설치 도구마다 별도로 프록시를 설정**해야 하는 번거로움이 존재합니다.
- Jenkins, Docker, Gradle 등은 각각 독립적으로 외부에 연결되기 때문에  
  환경 변수 외에 **전용 설정 파일(proxy.xml, gradle.properties 등)**이 필요합니다.
- 이러한 구조를 구성하고 유지하는 비용을 고려하면,  
  **NAT Gateway를 통한 아키텍처 구성**이 더 현실적이고 관리가 쉬운 대안이 될 수 있습니다.
