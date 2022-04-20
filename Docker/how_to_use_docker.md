# docker image란?
- docker container를 만들기 위해 필요한 설정이나 종속성을 갖고 있는 소프트웨어 패키지
- 직접 만들거나 도커 허브에 이미 만들어진 것을 이용할 수 있음
- 직접 만든 것을 도커 허브에 올려서 쓰는 것도 가능함


# miniconda image 받아서 쓰기

1. docker container 받기

```bash
# anaconda 최소 패키지
docker pull continuumio/miniconda3
```

2. container 실행해보기

```bash
# docker run -itd --name {your_container_name} --restart=always {your_image_name}
# docker exec -it {your_container_name} bash
docker run -itd --name miniconda --restart=always continuumio/miniconda3
docker exec -it miniconda bash
```

3. container에서 python 실행해보기

```bash
python3 -c "print('a')" # 정상적으로 실행됨
python3 -c "import numpy as np" # numpy를 설치하지 않았으므로 에러 발생
```

4. container에서 package 설치해보기

```bash
pip install numpy
python3 -c "import numpy as np" # 정상적으로 실행됨
```

5. 종료: exit를 입력하거나 control + D를 누르면 종료됨


## image commit 하기

1. image로 만들 container id 확인하기 

```bash
docker ps -a
```

2. image commit하기

```bash
# docker commit {container_name or container_id} {repository_name}:{tag_name}
docker commit miniconda ml_study:init
```

3. image가 잘 만들어졌는지 확인

```bash
docker images
```

4. 새로운 image로 container 실행해보기

```bash
# docker run -itd --name {your_container_name} --restart=always {your_image_name}
# docker exec -it {your_container_name} bash
docker run -itd --name ml_study --restart=always ml_study:init
docker exec -it ml_study bash

# numpy를 설치했던 image를 이용해 container를 만들었으므로 정상적으로 실행됨
python3 -c "import numpy as np"
```


## image, container 관리

```bash
# container 프로세스 목록 조회
docker ps -a

# container 삭제
# docker stop {container_name or container_id}
# docker rm {container_name or container_id}
docker stop miniconda
docker rm miniconda

# image 목록 조회
docker images -a

# image 삭제
# docker rmi {repository or image_id}
docker rmi continuumio/miniconda3
```


# Dockerfile 만들기

## 기본 익히기

- https://javacan.tistory.com/entry/docker-start-7-create-image-using-dockerfile

1. Dockerfile 파일 작성 (파일명: Dockerfile)

```vim
FROM alpine:3.10

ENTRYPOINT ["echo", "hello"]
```

2. Dockerfile 빌드

- 자동으로 Dockerfile 이름을 스캔해서 빌드해줌
- 다른 이름을 쓰고 싶으면 --file(-f) 옵션을 주면 됨

```bash
docker build --tag echoalpine:1.0 .
```

3. 새로 만든 image로 container 실행해보기

```bash
docker run --rm echoalpine:1.0
```


## miniconda + jupyterlab

- https://movingjin.tistory.com/m/26

1. Dockerfile, docker-compose.yml 파일 작성

- Dockerfile
```vim
FROM continuumio/miniconda3

LABEL maintainer="ejhan <nicedw17@gmail.com>"
LABEL version="0.1"
LABEL description="Debugging Jupyter Lab"

WORKDIR /jup

RUN conda install -c conda-forge jupyterlab
RUN conda install -c conda-forge nodejs
RUN jupyter labextension install @jupyterlab/debugger
RUN conda install xeus-python -c conda-forge

EXPOSE 8888

ENTRYPOINT ["jupyter", "lab","--ip=0.0.0.0","--allow-root"]
```

- docker-compose.yml
```vim
version: "3.7"
services:
  miniconda:
    build: .
    ports:
      - "8888:8888"
    volumes:
      - type: bind
        source: ./mount/config/jupyter_lab_config.py
        target: /root/.jupyter/jupyter_lab_config.py
      - type: volume
        source: jupyter_lab
        target: /jup
volumes:
  jupyter_lab:
```

- config 파일 생성
```bash
mkdir ./mount
mkdir ./mount/config
vim ./mount/config/jupyter_lab_config.py
```

2. build image

```bash
docker-compose build
```

- nodejs 버전 에러 발생

```bash
 => ERROR [5/6] RUN jupyter labextension install @jupyterlab/debugger                                                      1.7s
------
 > [5/6] RUN jupyter labextension install @jupyterlab/debugger:
#8 1.673 An error occurred.
#8 1.673 ValueError: Please install nodejs >=12.0.0 before continuing. nodejs may be installed using conda or directly from the nodejs website.
#8 1.673 See the log file for details:  /tmp/jupyterlab-debug-3vjeo7ia.log
------
executor failed running [/bin/sh -c jupyter labextension install @jupyterlab/debugger]: exit code: 1
ERROR: Service 'miniconda' failed to build : Build failed
```

- Dockerfile에 버전 업그레이드를 추가해서 해결

```vim
RUN conda upgrade -c conda-forge nodejs
```

3. container 실행

- Docker Compose는 여러 개의 컨테이너(container)로 구성된 애플리케이션을 관리하기 위한 간단한 오케스트레이션(Orchestration) 도구
	+ https://www.daleseo.com/docker-compose/

```bash
docker-compose up -d
```

4. 비밀번호 설정

```bash
docker exec -it {container_name} bash

# python 실행해서 패스워드 해시값을 얻어둠
python3
from notebook.auth import passwd; passwd()

# config file 생성: /root/.jupyter/jupyter_lab_config.py로 생성
jupyter lab --generate-config

# vim 설치
apt-get update
apt-get install vim

# config file 수정: /root/.jupyter/jupyter_lab_config.py에 아래 값 추가
c.NotebookApp.password = '위에서 얻은 해시값'
```

- 비밀번호 설정 완료 후, 아래 주소로 접속

http://localhost:8888/lab

- 접속이 안 될때 : 포트가 겹치는지 확인

```bash
jupyter lab list
jupyter lab stop 8888
```

