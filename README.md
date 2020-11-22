# 실행 조건:
1. k8s cluster 구성(1.17v 이하)
2. nginx-controller 구성
3. 사용 가능한 storage-class로 변경( /k8s-manifest/petclinic-db/pvc.yaml의 아래 부분 수정)<pre><code>storageClassName: local-storage</code></pre>  
4. default namespace 확인
5. worker node(pod가 기동될 노드)에 /logs 디렉토리 생성 확인


# 실행 방법
1. git clone
<pre>
<code>
git clone https://github.com/manjoong/spring-petclinic.git
</code>
</pre>

2. 기존 build 디렉토리 삭제
<pre>
<code>
./gradlew clean
</code>
</pre>
3. build 시작(build dir 생성 확인)
<pre>
<code>
./gradlew bootjar
</code>
</pre> 
4. jib plugin을 통한 이미지 배포(skekf123/petclinic:latest로 배포)
<pre>
<code>
./gradlew jib
</code>
</pre> 
5. k8s 리소스 배포(defualt namespace에 배포됨)
<pre>
<code>
  kubectl apply -f k8s-manifest/petclinic-db
  kubectl apply -f k8s-manifest/petclinic
</code>
</pre>


# 요구사항 관련
### 1. 어플리케이션의 ​log​는 ​host​의 ​/logs ​디렉토리에 적재되도록 한다​.
- 해결방안: hostpath volume mount를 통하여 /log가 host의 /logs 디렉토리에 저장되도록 설정
### 2.  정상 동작 여부를 반환하는 ​api​를 구현하며​, 10​초에 한번 체크하도록 한다​. 3​번 연속 체크에 실패하 면 어플리케이션은 ​restart ​된다​.
- 해결방안: liveness probe의 httpGet을 사용하여 10초 마다 정상여부 체크하며 3번의 error 발견 시, pod를 재기동
<pre>
  <code>
          livenessProbe:
            httpGet:
              path: /manage/health
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 10
            failureThreshold: 3
            periodSeconds: 10
  </code>
</pre>
### 3. 종료 시 ​30​초 이내에 프로세스가 종료되지 않으면 ​SIGKILL​로 강제 종료 시킨다​.
- 해결방안: terminationGracePeriodSeconds 옵션을 통해 강제 종료 옵션 설정
<pre>
  <code>
      terminationGracePeriodSeconds : 30
  </code>
</pre>
### 4. 배포 시와 scale in/out 시 유실되는 트래픽이 없어야 한다.
- 해결방안: petclinic앱의 rollingupdate 정책을 통해 scale in/out시 서비스 순단 및 유실  트래픽 제거
<pre>
  <code>
    replicas: 2
    strategy:
        type: RollingUpdate
        rollingUpdate:
            maxUnavailable: 0
            maxSurge: 1
  </code>
</pre>
### 5. 어플리케이션 프로세스는 ​root ​계정이 아닌 ​uid:1000​으로 실행한다​.
- 해결방안: securityContext를 통해 uid 1000으로 프로세스를 실행
### 6. DB​도 ​kubernetes​에서 실행하며 재 실행 시에도 변경된 데이터는 유실되지 않도록 설정한다​. ​어플리케이션과 ​DB​는 ​cluster domain​을 이용하여 통신한다​.
- 해결방안: mysql resource에 고정된 service명(petclinic-db-svc)을 통해 app과 db가 통신. pv를 사용하여 mysql pod가 재기동 되더라도 데이터 보존.
### 7. nginx-ingress-controller​를 통해 어플리케이션에 접속이 가능하다​.
- 해결방안: ingress를 구성하여 nginx-ingress-controller를 통해 접속 가능
### 8. namespace​는 ​default​를 사용한다​.
- k8s yaml에 namespace defualt로 입력
