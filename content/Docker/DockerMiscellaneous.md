---
title: "Docker: 기본 조작법 이것저것"
---
최종 업데이트 일시: `2026년 5월 9일 (일) 15시 (KST)`

---
## 기본적인 명령어들

```
# To stop a running container
docker stop {CONTAINER_ID}

# To remove a container:
docker rm {CONTAINER_ID}

# To remove an image
docker rmi {IMAGE_ID}

# To list available images on my local PC
docker images

# detached-mode로 이미지에서 컨테이너 시작하기
# -p    : port forwardning
# --name: 컨테이너 이름 지정 
docker run -d -p 8081:8000 --name mypy my-py-app:1.0 
```

## `Dockerfile`

#### 가장 간단한 예시

```dockerfile
FROM python:3.11-slim

WORKDIR /pyapp

COPY requirements.txt .
RUN pip install -r requirements.txt

# COPY <source> <destination>
COPY . .

EXPOSE 8000

CMD ["python", "mypyapp.py"]
```

#### `Dockerfile`에서 이미지 생성하기

```
docker build -t {이미지이름:태그} -f {DockerFile경로} .

# Build 진행상황을 표시
docker build --progress=plain -t {이미지이름:태그} -f {DockerFile경로} .
```

#### Image 분석하기: `inspect`

이미지에 입혀진 환경변수, 레이어(해시) 등 메타데이터를 확인하고 싶을 때 사용한다.

```
docker inspect {이미지이름}
```

예시: `python:3.11-slim` 분석하기
```
docker inspect my-py-app:1.0
```

이런 결과가 나온다. (전후 생략)
```json
"RootFS": {
    "Type": "layers",
    "Layers": [
        "sha256:79dd1f4c855cd061f687a994426634cf5f84c8ecdbc66c7a7d118e828dd93c99",
        "sha256:2e53cb234c597e3f6a8f896df5efafd6ceca83645632661f2e32cd7144b43738",
        "sha256:799edc77eb3d21542943122b97c5ba1e80c2a1eb3d222e23196717126398ef61",
        "sha256:94e760ec30748f73d19bcb5d4da140b8dde956299eac18e3372d89236f56c995",
        "sha256:596f07c5e348cccc87d6119557fda2cac78064a4ed0c5a9996def135c9152ef3",
        "sha256:e70eb102e5fc6c2b880a6993e09adb4f3ef85a89902b83d2befeee7afca5dcc3",
        "sha256:c92c8fc939a0c13c6d5d18c5915a5aedc119428785f48432e5fab47722e1389d",
        "sha256:ae99d8ff71f764d0bb7c71cb685431306a152bf1a884a4fa0db62d84743c7c34"
    ]
},
```

`python:3.11-slim` 이미지의 `Dockerfile`은 다음 URL에서 확인할 수 있다.
- [GitHub  링크](https://github.com/docker-library/python/blob/c859f2b9e567f72c94e00c969f916d3f92ae52a7/3.11/slim-trixie/Dockerfile)

Debian 이미지를 사용하고 있는 것을 알 수 있다.
```dockerfile
FROM debian:trixie-slim
(이하 생략)
```

Debian 이미지를 inspect 해 보면,
```
docker inspect debian:trixie-slim
```

다음과 같은 레이어 해시가 확인된다.
```json
"RootFS": {
    "Type": "layers",
    "Layers": [
        "sha256:79dd1f4c855cd061f687a994426634cf5f84c8ecdbc66c7a7d118e828dd93c99"
    ]
},
```

#### 컨테이너 환경변수 지정하기

방법 1: `Dockerfile`에 `ENV` 값으로 지정

```dockerfile
ENV MY_NAME=BLUE
```

방법 2: 컨테이너 생성(`docker run`) 시점에 지정: `-e` 플래그
- 이미지에 사용된 값이 있으면 재정의(overriding)

```
docker run -d -p 8081:8000 --name mypy -e MY_NAME=RED my-py-app:1.0
```

