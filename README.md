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
