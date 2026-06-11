# 최소준비요건
영구 데이터 저장소(Persistence)를 준비히야하고 이에 연결할수 있도록 정확한 StorageClass 등을 값으로 전달해야 함.
default로 어떤 storageclass도 없다면 rencher 의 node-local-path-provisioner를 설치하여 사용한다.

# 제약사항


# 모니터링 데이터 수집룰 관련 설정
pod monitor는 configmap(configmap-lma-provisioner) 내부의 "podmonitor_truss.yaml"에 정의함으로 exporter와 연결될 수 있다.
pod monitor와 exporter의 연동부분은 아래와 같다.
``` yaml
          {{- range $key, $value := $podmon.selector.matchLabels }}
          {{ $key  }}: {{ $value }}
          {{- end }}
```
여기에 사용되는 값들은 value를 재지정함으로서 설정할 수 있는데 그 위치는 lmaProvisoner.podMonitor: 이며 아래는 그 예시이다. (kubeai의 embeded exporter에 대한 수집)
``` yaml
  - name: kubeai-models-pod-monitor
    endpoint:
    - port: http
      path: /metrics
      interval: 15s
    selector:
      matchLabels:
        app: model
        app.kubernetes.io/managed-by: kubeai # FIXME: 이건 pod의 종류별로
```
다음은 구체적인 연결 설명이다. pod의 라벨과 pod monitor의 spec.selector가 매칭된 것을 확인할 수 있다.
``` bash
$ kubectl describe po -n kubeai model-deepseek-r1-57c899f5c6-mr8ht 
Name:             model-deepseek-r1-57c899f5c6-mr8ht
...
Labels:           app=model
                  app.kubernetes.io/instance=ollama-deepseek-r1
                  app.kubernetes.io/managed-by=kubeai
                  app.kubernetes.io/name=ollama
                  model=deepseek-r1
                  pod-hash=57c899f5c6
Annotations:      model-pod-port: 8000
Status:           Running
...

$ kubectl describe podmonitors.monitoring.coreos.com -n truss kubeai-models-pod-monitor 
Name:         kubeai-models-pod-monitor
...
Spec:
  Namespace Selector:
  Pod Metrics Endpoints:
    Interval:  15s
    Path:      /metrics
    Port:      http
  Pod Target Labels:
    camel.apache.org/integration
  Selector:
    Match Labels:
      App:                           model
      app.kubernetes.io/managed-by:  kubeai
Events:                              <none>
```
