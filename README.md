# Multiple containeirs / Single Pod
```
podman pod create --replace --name test -p 8080:80 -p 3306:3306
```
```
podman run --replace -it --rm --pod test \
  --name web docker.io/library/httpd
```
```
podman run --replace -it --rm --pod test \
  --name db -e MYSQL_ROOT_PASSWORD=test docker.io/library/mariadb
```
```
podman run --replace -it --rm --pod test \
  --name bastion --entrypoint= docker.io/library/alpine ash
apk add curl mysql-client
curl http://127.0.0.1
mysql -h 127.0.0.1 -uroot -ptest -e "show databases"
```
# Podman Multi Architecture Images
```
sudo apt install qemu-user-static podman buildah skopeo jq
sudo /etc/init.d/binfmt-support restart

podman ps

echo ok > index.html

cat <<'EOF'> Dockerfile
FROM docker.io/library/nginx:stable
COPY index.html /usr/share/nginx/html/index.html
RUN set -ex \
    && dpkg --print-architecture
EOF

podman build --platform=linux/amd64 --format=docker -t website:amd64 .
podman build --platform=linux/arm64/v8 --format=docker -t website:arm64 .

podman push website:amd64 docker-archive:website-amd64.tar
skopeo inspect docker-archive:website-amd64.tar | jq

podman push website:arm64 docker-archive:website-arm64.tar
skopeo inspect docker-archive:website-arm64.tar | jq

podman manifest create website:v1
podman manifest add website:v1 docker-archive:website-amd64.tar
podman manifest add website:v1 docker-archive:website-arm64.tar

podman manifest push --all website:v1 oci-archive:website.tar
skopeo inspect --raw oci-archive:website.tar | jq

podman load -i website.tar | \
  awk -F':' '{print $NF}' | \
  xargs -i podman tag {} website

podman run -it --rm website

podman run -d --rm --name registry -p 5000:5000 docker.io/library/registry:2

podman manifest push --all website:v1 127.0.0.1:5000/website:v1 --tls-verify=false
skopeo copy --all \
  oci-archive:website.tar \
  docker://localhost:5000/website:v1 \
  --dest-tls-verify=false

skopeo inspect --tls-verify=false --raw docker://localhost:5000/website:v1 | jq .

skopeo copy --all \
  oci-archive:website.tar \
  docker://localhost:5000/website:latest \
  --dest-tls-verify=false

skopeo copy --override-arch=amd64 \
  docker://localhost:5000/website:v1 \
  docker://localhost:5000/website:v1-amd64 \
  --dest-tls-verify=false \
  --src-tls-verify=false

skopeo copy --override-arch=arm64 \
  docker://localhost:5000/website:v1 \
  docker://localhost:5000/website:v1-arm64 \
  --dest-tls-verify=false \
  --src-tls-verify=false

skopeo inspect --tls-verify=false docker://localhost:5000/website:v1 | jq .

podman run -it --rm --tls-verify=false -p 8080:80 localhost:5000/website:v1

curl http://localhost:8080


# Test Docker
sudo apt install docker.io
sudo adduser $USER docker
sudo su - $USER

docker ps

skopeo copy --override-arch=amd64 \
  oci-archive:website.tar \
  docker-archive:website-amd64.tar

skopeo inspect docker-archive:website-amd64.tar | jq

skopeo copy --override-arch=arm64 \
  oci-archive:website.tar \
  docker-archive:website-arm64.tar

skopeo inspect docker-archive:website-arm64.tar | jq

docker load -i website-amd64.tar | \
  awk -F':' '{print $NF}' | \
  xargs -i docker tag {} website-amd64

docker run -it --rm website-amd64

docker load -i website-arm64.tar | \
  awk -F':' '{print $NF}' | \
  xargs -i docker tag {} website-arm64

docker run -it --rm website-arm64
```
# Podman Play Kube
```
cat <<'EOF'> podman_play_kube.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: test
  name: test
spec:
  containers:
  - args:
    - ""
    image: localhost/test:latest
    name: app0
    ports:
    - containerPort: 8080
      hostPort: 8080
    env:
    - name: server_port
      value: 8080
    securityContext:
      capabilities:
        drop:
        - CAP_MKNOD
        - CAP_NET_RAW
        - CAP_AUDIT_WRITE
  - args:
    - ""
    image: localhost/test:latest
    name: app1
    ports:
    - containerPort: 8081
      hostPort: 8081
    env:
    - name: server_port
      value: 8081
    securityContext:
      capabilities:
        drop:
        - CAP_MKNOD
        - CAP_NET_RAW
        - CAP_AUDIT_WRITE
  restartPolicy: Never
EOF
```
```
podman play kube podman_play_kube.yaml --down
podman play kube podman_play_kube.yaml
```
```
podman exec -it test-app0 bash
curl http://test:8080/actuator/health
curl http://test:8081/actuator/health
```
