# Docker


## docker와 vm의 차이
docker는 프로세스를 가상화해서 가벼움 하이퍼바이저 오버헤드가 줄어 훨씬 좋은 성능
도커가 왜 떴는가 -> 규격화된 컨테이너로 환경에 차이 영향이 없다. 컨테이너간 격리되어 인프라에 신경 쓸게 줄어들고 빠르게 배포 할 수 있습니다.

## Dockerfile

EXPOSE
일반적으로 dockerfile을 작성하는 사람과 컨테이너를 직접 실행할 사람 사이에서 공개할 포트를 알려주기 위해 문서 유형으로 작성할 때 사용 됩니다.
실제로 포트를 열기 위해선 container run 에서 -p 옵션을 사용해야 합니다.

VOLUME
컨테이너 안에 있는 데이터는 컨테이너를 삭제하면 모든 데이터가 같이 삭제(휘발성 데이터) 되기 때문에 데이터를 보존하기 위해 VOLUME을 사용 하고, VOLUME 명령은 설정한 컨테이너의 데이터를 호스트 OS에 저장하거나, 컨테이너들간의 데이터를 공유가 가능합니다.

RUN
RUN 명령은 아래 예시처럼 패키지 설치 등에 사용 되는건데, 좀 더 특정지어 말한다면 RUN 명령은 image layer를 만들어냅니다. (image layer는  [https://nirsa.tistory.com/63?category=868315](https://nirsa.tistory.com/63?category=868315)  참고)
```
# nignx 설치 (/bin/sh 예시)
# RUN yum -y install nginx

# nignx 설치 (exec 형식 예시)
# RUN ["/bin/bash", "-c", "yum -y install nginx"]
```

CMD, ENTRYPOINT
일단 두가지의 명령 모두 이미지를 바탕으로 생성된 ~컨테이너에서 사용~ 됩니다. (docker run로 생성/실행 또는 docker start으로 실행 시)
CMD와 ENTRYPOINT의 차이는 docker run 을 사용하며 새로운 명령을 지정한 경우 이 명령이 실행 되는지, 안되는지의 차이 입니다.
CMD : 컨테이너가 실행될 때 명령어 및 인자값 전달하여 실행, 단 docker run 명령에 쉘 명령어 및 인자값 전달할 경우 CMD에 작성된 명령어와 인자값은 무시 됩니다. (다른 명령어와 인자값을 사용한 예는 아래 이미지 확인)
즉, 우선 순위는 docker run > CMD
ENTRYPOINT : 컨테이너가 실행될 때 명령어 및 인자값 전달하여 실행, 단 docker run 명령에 쉘 명령어 및 인자값 전달할 경우 쉘 명령어는 영향을 받지 않으며, 인자값만 영향을 받습니다.
즉, 우선 순위는 ENTRYPOINT (인자값) < docker run < ENTRYPOINT (명령어)
CMD는 명령과 인자값이 변경될 수 있고, 컨테이너에서 명령 설정하지 않을 시 CMD에 기재된 명령을 기본값으로 실행됩니다. ENTRYPOINT는 명령 수정이 불가능하여 사용자에 의해 변경되지 않고 고정적으로 실행될 명령은 ENTRYPOINT를 사용하는것이 좋습니다.

COPY,ADD
COPY의 경우 호스트OS에서 컨테이너 안으로 복사만 가능하지만, ADD의 경우 원격 파일 다운로드 또는 압축 해제 등과 같은 기능을 갖고 있습니다. 즉, 호스트OS에서 컨테이너로 단순히 복사만을 처리할 때 COPY를 사용 합니다.

WORKDIR
WORKDIR은 명령을 실행하기 위한 디렉토리를 지정 합니다.

#docker