# OpenPaaS 모니터링
OpenPaaS OWL은 PaaS-TA PaaS Platform의 운영자/사용자를 위한 Metric 수집/모니터링/알람 연계 시스템입니다.
세부 기능은 아래와 같습니다.
* OpenPaaS Platform Metric 수집
* OpenPaaS 모니터링 Dashboard 자동 구성
* Agentless Model
* Alert 연계 기능 제공(Grafana/Alertmanager)
* 3rd-party exporter 연계가능 (https://prometheus.io/docs/instrumenting/exporters/)

## 구성요소
* Visualization(Grafana) : PaaS Platform Metric Dashboard 제공
* Metric collector  
  * [bosh exporter](https://github.com/cloudfoundry-community/bosh_exporter) : bosh metric 수집
  * [irehose exporter](https://github.com/cloudfoundry-community/firehose_exporter) : Cloudfoundry metric 수집
  * [cf exporter](https://github.com/cloudfoundry-community/cf_exporter) : Cloudfoundry API metric 수집
  * [node exporter](https://github.com/prometheus/node_exporter) : VM Metric 수집
* Backend Database : Prometheus(Time series database)
