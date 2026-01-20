ssh bsh@192.168.101.197 node 에 단일 쿠버네클스 클러스터가 구성이 되어 있고 cs지는 longhorn으로 구성되어 있다. monitoring 네임스페이스의 구성을 모두 제거하고 아래작업을 진행하면 된다. 

쿠버네티스 클러스터에 gitops (giblab) + argocd 배포 파이프라인을 구성하여 해당 파이프라인을 통해 
https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack 을  
kustomize + helm 배포로 구성하고 kube-prometheus-stack의 위 helm chart의 value.yaml 의 구조를 그대로 유지한채로 필요부분만 수정하고 주석을 작성한다.

https://artifacthub.io/packages/helm/opensearch-operator/opensearch-operator/ 
https://artifacthub.io/packages/helm/fluent/fluent-operator
https://artifacthub.io/packages/helm/opensearch-project-helm-charts/data-prepper
opensearch + data prepper + fluent-bit 파이프라인을 위 helm 구성을 참고하여 구성하되 마찬가지로 필요한 부분만 수정하고 주석을 작성한다.


기본 모니터링 스택은 위와 같이 구성하여 gitops-argocd 배포를 위한 구성을 하고 구조적으로 구성한다.


log pipe인라인으로 기본적으로 k8s container log  파이프라인을 구성하고 
java application 을 빌드하고 배포하여 별도의 hostpath에 /var/log/{namespace}/{app}-{service}-{jobid-name와 같은 별도의 구성}-YYYY-MM-dd.log 파일을 tail로 pipeline 정책을 통해 배포하도록 한다. 그리고 이 배포파이프라인데 대해 서비스 사용자들이 별도의 hostpath 로그 수집을 위해 무엇을 해야하는지에 대한 가이드 문서도 자세하게 작성하고, 
파이프라인 구성시 모둔 input , parse , filter , output 과 관련된 구성은 모두 fluent-bit CR https://doc.crds.dev/github.com/fluent/fluent-operator@v1.0.0 을 사용하도록 한다. 

