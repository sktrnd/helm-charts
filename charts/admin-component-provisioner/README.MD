본 챠트는 멀티클러스터 환경에서 모니터링 시스템을 지원하기 위한 것으로 배포되는 클러스터에 중앙 컴포넌트를 배포하고 지정한 워커클러스터에 필요한 컴포넌트들을 배포하여 연동되도록 한다.

본 챠트를 통해 구성되는 모니터링 컴포넌트는 다음과 같다. 하기의 리스트는 최초 개발시 내역이므로 추가될수 있다.
- thanos
- grafana
- worker-provisioner
- minio
- (option) ganesha

worker-provisioner는 워커클러스터에 필요한 컴포넌트를 배포하는 것으로 설정에따라 job을 통해 원격에 배포된다. 

## 장기데이터 저장지원
기 준비된 s3 서비스가 있다면 lmaProvisioner.objstore에 필요한 정보를 입력하여 연동하도록 할 수 있다. 
값 지정 예시: (helm의 키바식은 낙타형이 아니라 _를 사용한 것은 이 내역이 s3 설정이기 때문)
``` yaml
  objstore:
    type: s3
    config:
      access_key: taco
      bucket: tks-thanos
      endpoint: 192.168.123.13:30003
      insecure: true
      secret_key: mypassword
```

위 값을 지정하지 않는다면 admin cluster에 minio를 설치하고 이를 s3로 활용하여 모니터링 파이프라인이 구성된다. 이 경우 아래의 값을 정확히 지정해줘야 한다. 
명시적으로 enabled에 true를 지정하여 내부에 minio를 쓸것을 확인하고, 
admin cluster의 외부에서 클러스터에 설치된 minio에 연동하도록 하는 endpoint를 지정하는 것으로 nodeport를 통해 구성된다. 따라서 selfServiceMinIo.host를 정확하게 지정하고 네트워크 경로를 확보한다. 
``` yaml
selfServiceMinIo:
  enabled: true
  host: 192.168.5.161
  port: 30003
```

## 메트릭수집
thanos를 통해 관리하는 워커클러스터의 데이터를 연동한다. 연동구조는 value값을 통해 지정할 수도 있으며 지정하지 않는다면 kubeconfig 및 worker-provisioner 구성을 기반으로 자동으로 연결한다. 
| admin에서 worker의 수집기로 자동 연결은 kubeconfig의 서버접속정보를 기반으로 worker-provisioner와의 협력을 통해 구성하는 30901 포트로 연결한다.

## 대시보드
본 챠트에서는 grafana를 배포하고 thanos와 연동을 통해 각 클러스터의 메트릭에 접근할 수 있도록 하며 필요한 대시보드들도 제공한다.

## 워커노드의 지정방법
본 챠트의 value 값을 통해 워커노드를 지정할 수 있으며 자동으로 worker-provisioner가 실행되면서 워커노드에 필요한 자원을 배포한다. 수정시에도 value 값 변경을 통해 helm upgrade 를 수행해야 메트릭 수집 pipeline이 지정될 수 있다.

워커노드 배포 관련 workerProvisioner.cluster에 지정한다. 입력해야하는 값은 다음과 같다.
- name : 클러스터를 구분하는 구분자
- tenant : 소속된 그룹을 지정한다. (pvc를 공유)
- kubeconfig.secretName : kubeconfig를 저장하고 있는 secret명 지정
- (옵션) kubeconfig.config : kubeconfig 내용을 그대로 입력한다. 이곳에 값이 지정되면 이 값을 기반으로 kubeconfig.secretName 이름으로 secret을 생성한다.
- (옵션) monitoringAccessIp : 환경에 따라 kubeconfig를 통한 접근과 서비스계층에서의 접근이 다를 수 있는데(ex. petasus kaas) 이경우 모니터링 데이터 수집을 위한 접근IP를 지정할 수 있다. 

```yaml
  - name: cluster6
    tenant: tenant_a
    kubeconfig:
      secretName: cluster6-kubeconfig
      config:
        clusters:
        - cluster:
            certificate-authority-data: LS0....
            server: https://192.168.5.161:6443
          name: c6
        contexts:
        - context:
            cluster: c6
            user: kubernetes-admin
          name: kubernetes-admin@c6
        current-context: kubernetes-admin@c6
        kind: Config
        preferences: {}
        users:
        - name: kubernetes-admin
          user:
            client-certificate-data: LS0tLS1CRU...
            client-key-data: LS0tLS1C...
```