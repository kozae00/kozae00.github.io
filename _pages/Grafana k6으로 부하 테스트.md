---
title: Grafana k6으로 부하 테스트
author: kozae00
category: Jekyll
layout: post
---

>🔖 **Ref** : [Grafana k6으로 부하 테스트하고 시각화하기](https://velog.io/@heka1024/Grafana-k6%EC%9C%BC%EB%A1%9C-%EB%B6%80%ED%95%98-%ED%85%8C%EC%8A%A4%ED%8A%B8%ED%95%98%EA%B8%B0)

### 부하 테스트가 필요한 Reason ???

서비스를 개발하고, 부하 테스트는 일반적으로 실제 요구 부하를 서버가 견뎌낼 수 있는지 확인하는 작업 중 하나이다. 이 작업을 통해 내 어플리케이션의 건강성을 판단해볼 수 있다.

---

### 설치

**✔️ 최신 k6 바이너리 다운로드**
```bash
# k6 다운로드
choco install k6

# 설치 확인
k6 version
```

**🐋Docker 방식**
```sh
docker pull grafana/k6
```

--- 
### 실행

✔️ **`script.js` 파일을 통해 실행**
```js
import http from 'k6/http';
import { sleep } from 'k6';

export default function () {
  http.get('https://test.k6.io');
  sleep(1);
}
```

**실행**
```bash
k6 run script.js
```

![[k6 실행 이미지.png]]

해당 **🔖Ref** 에서는 설정값 변경, `--vus=10`(virtual users)과 `--duration=10s`를 주고 실행.

![[k6 실행 설정값 변경 후 실행- (2).png]]

위 옵션 전체를 `script.js` 파일에 넣을 수 도 있다.
```js
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  vus: 10,
  duration: '10s',
};

export default function () {
  http.get('https://test.k6.io');
  sleep(1);
}
```

위 옵션을 좀 더 세밀하게 지정하면 다음과 같은 시나리오 구성
```js
export const options = {
  stages: [
    { duration: '30s', target: 20 },
    { duration: '1m30s', target: 10 },
    { duration: '20s', target: 0 },
  ],
};
```

---

>💡 `intellij` 에서 `k6` 플러그인을 활용해 결과를 볼 수 있다.

![[intellij 에서 확인.png]]
![[intellij에서 확인(2).png]]

---
### Grafana를 이용한 시각화

해당 내용에서는 `influxdb`, `grafana`를 이용해 k6의 결과를 시각화 함.

---

### 환경 Settings

다음과 같은 `docker-compose.yml` 파일을 만들어 실행 준비를 한다. 
준비 사항으로는 ...
- `grafana`
- `influxdb`

```yaml
version: "3.7"

services:
  influxdb:
    image: bitnami/influxdb:1.8.5
    container_name: influxdb
    ports:
      - "8086:8086"
      - "8085:8088"
    environment:
      - INFLUXDB_ADMIN_USER_PASSWORD=bitnami123
      - INFLUXDB_ADMIN_USER_TOKEN=admintoken123
      - INFLUXDB_HTTP_AUTH_ENABLED=false
      - INFLUXDB_DB=myk6db
  granafa:
    image: bitnami/grafana:latest
    ports:
      - "3000:3000"
```

`docker-compose up` 을 통해 `grafana`와 `influxdb`를 준비하자.

다음 명령어를 통해 준비 여부를 확인.
```bash
influx ping
```
![[influx ping 명령어 실행.png]]

>⚠️ `influx` 명령어를 찾을 수 없다고 나올 때는, `choco install influxdb` 설치.

---
### Grafana Datasource 추가

![[grafana datasource 추가.png]]

해당 이미지와 같이 **url**과 **database**를 설정. 

두 컨테이너 모두 docker 위에서 돌아가고 있으므로 `http://localhost:8086`이 아니라 `http://influxdb:8086`으로 설정.

![[grafana datasource 설정 확인 메세지.png]]

해당 메세지 `Save & test`를 클릭하여 위와 같은 표시가 나오면 성공.

그리고 대시보드를 import 하자. ([https://grafana.com/grafana/dashboards/2587-k6-load-testing-results/](https://grafana.com/grafana/dashboards/2587-k6-load-testing-results/)) 해당 페이지의 대시보드를 가져옴.

다음과 같이 `k6`의 결과를 `influxdb`로 export하면 시각화를 할 수 있다.

**실행문** 
```null
k6 run \
  --out influxdb=http://localhost:8086/myk6db \
  script.js
```

**결과**
![[grafana 대시보드 결과 이미지.png]]