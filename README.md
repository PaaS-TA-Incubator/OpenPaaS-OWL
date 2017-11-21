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
  * [firehose exporter](https://github.com/cloudfoundry-community/firehose_exporter) : Cloudfoundry metric 수집
  * [cf exporter](https://github.com/cloudfoundry-community/cf_exporter) : Cloudfoundry API metric 수집
  * [node exporter](https://github.com/prometheus/node_exporter) : VM Metric 수집
* Backend Database : Prometheus(Time series database)


## 배포 방법
본 가이드는 bosh manifest V1 기준으로 작성되었습니다.

1. 배포 전, Metric 수집을 위한 사전 준비가 필요합니다
    * bosh exporter를 이용한 Metric 수집을 위해 Director SSL 인증서 구성하여 Director에 적용합니다.
      * gen_cert_bosh.sh 수행하여 ssl 인증서를 생성합니다.
      * gen_cert_bosh.sh 내용 중 bosh target IP를 환경에 맞게 수정합니다.
        ```
        generateCert director x.x.x.x # <--- Replace with public Director IP
        generateCert uaa-web x.x.x.x  # <--- Replace with public Director IP
        generateCert uaa-sp x.x.x.x   # <--- Replace with public Director IP
        ```
        ```
        $ ./gen_cert_bosh.sh
          director.crt  director.key  rootCA.key  rootCA.pem  rootCA.srl  uaa-sp.crt  uaa-sp.key  uaa-web.crt  uaa-web.key
        ```
        * Director 배포 manifest 를 업데이트합니다.
        
        ```
        ...
        jobs:
        - name: bosh
          properties:
                director:
                  ssl:
                    key: |
                      -----BEGIN RSA PRIVATE KEY-----
                      certs/director.key
                      -----END RSA PRIVATE KEY-----
                    cert: |
                      -----BEGIN CERTIFICATE-----
                      certs/director.crt
                      -----END CERTIFICATE-----
                ...
                hm:
                  director_account:
                    user: hm
                    password: hm-password
                    ca_cert: |
                    -----BEGIN RSA PRIVATE KEY-----
                    certs/rootCA.pem
                    -----END RSA PRIVATE KEY-----
                ...
          ```
      * microbosh ssl 인증서 적용
        ```
        $ bosh-init deploy bosh-deployment.yml
        ```

    * cf exporter/firehose exporter를 이용한 Metric 수집을 위해 UAA Client 설정합니다.
    * OpenPaaS Controller 배포정의서에 uaa client 정보를 추가하기 위해 배포 정의서를 수정합니다.
      ```
      properties:
        ...
        uaa:
          ...
          clients:
            ...
              cf-exporter:
                override: true
                authorized-grant-types: client_credentials,refresh_token
                authorities: cloud_controller.admin
                scope: openid,cloud_controller.admin
                secret: cf-exporter-secret
              firehose-exporter:
                override: true
                authorized-grant-types: client_credentials,refresh_token
                authorities: doppler.firehose
                scope: openid,doppler.firehose
                secret: firehose-exporter-secret
      ```
    * UAA <-> grafana OAuth 인증 연동을 위한 UAA Client 설정합니다.
      ```
      properties:
        ...
        uaa:
          ...
          clients:
            ...
              grafana:
                override: true
                authorized-grant-types: authorization_code
                authorities: uaa.none
                scope: openid
                secret: grafana-secret
      ```
    * OpenPaaS Controller 변경 내용 적용
      ```
      $ bosh deployment paasta-controller-2.0.yml
      $ bosh deploy
      ```
    * UAA CLI(UAAC)를 통한 Client 확인
      * UAAC 설치 docs를 참고 
        (https://docs.cloudfoundry.org/uaa/uaa-user-management.html)
      * 추가 된 Client 정보 확인
      ```
      $ uaac token client get admin -s 'xxxxxxx'     //paasta-controller 배포 정의내 uaa.admin.client_secret 정보 확인
      $ uaac client get cf-exporter
      scope: uaa.none
      client_id: cf-exporter
      resource_ids: none
      authorized_grant_types: refresh_token client_credentials
      autoapprove:
      authorities: cloud_controller.admin
      name: cf-exporter
      lastmodified: 1492082120000
      ```
      * UAA 내 Client가 정상적으로 등록 되지 않았을 경우 메시지 참고
      ```
      $ uaac client get cf-exporters-test
      CF::UAA::NotFound: CF::UAA::NotFound
      ```

2. OpenPaaS OWL 배포
    * BOSH Stemcell Upload
      * 기존 OpenPaaS Controller/Container 배포 시 사용하였던 Stemcell 사용하는 경우 별도의 Stemcell Upload 작업이 필요하지 않습니다. 
      * 배포에 필요한 별도의 Stemcell Upload가 필요한 경우 아래 docs를 참고하여 업로드합니다.
        https://bosh.io/docs/sysadmin-commands.html#dir-stemcells
    * BOSH Release upload
      * OpenPaaS OWL 배포를 위한 BOSH release를 업로드 합니다.
      ```
      $ bosh upload release openpaas-owl-1.0.0.tgz
      $ bosh upload release exporters/node-exporter-1.1.0.tgz
      $ bosh releases
        RSA 1024 bit CA certificates are loaded due to old openssl compatibility
        Acting as user 'admin' on 'mybosh'

        +-------------------+----------+-------------+
        | Name              | Versions | Commit Hash |
        +-------------------+----------+-------------+
        | node-exporter     | 1.1.0    | d2706592+   |
        | openpaas-owl      | 1.0.0    | 0641ca50+   |
        +-------------------+----------+-------------+
        (*) Currently deployed
        (+) Uncommitted changes

        Releases total: 2
      ```    
    * OpenPaaS OWL 배포 정의서 수정
      * OpenPaaS OWL 1.0 에서는 OpenStack IaaS 환경 기준의 배포정의서가 준비되어있습니다.
        (별도 AWS, VMware, GCP, Softlayer등 IaaS 환경의 배포 정의서는 배포정의서의 일부 수정이 필요합니다.) 
      * openpaas-owl-openstack-1.0.yml 배포 정의서를 각 환경에 맞게 수정합니다.
        (openpaas-owl-openstack-1.0.yml 참조)

    * OpenPaaS OWL 배포(Openstack IaaS 기준)
      ```
      $ bosh deployment openpaas-owl-openstack-1.0.yml
      $ bosh deploy
      ```
    * 배포 확인
      ```
      $ bosh vms openpaas-owl-deployment
      +-------------------------------------------------------+---------+-----+------------------+---------------+
      | VM                                                    | State   | AZ  | VM Type          | IPs           |
      +-------------------------------------------------------+---------+-----+------------------+---------------+
      | alertmanager/0 (2c888859-8a24-4fe2-80cf-258a9683edba) | running | n/a | small_z1         | x.x.x.x       |
      | grafana/0 (41956e07-0a4f-43e0-9bbd-f9082e7f3617)      | running | n/a | small_z1         | x.x.x.x       |
      | prometheus/0 (fb5a7bc0-f0ba-4bbc-be68-1d7dc819a61e)   | running | n/a | medium_z1        | x.x.x.x       |
      +-------------------------------------------------------+---------+-----+------------------+---------------+
      VMs total: 3
      ```
3. 배포 확인 
    * grafana UI : http://grafana_static_IP:3000
    * alertmanager UI : http://alertmanager_static_IP:9093
    * prometheus UI : http://prometheus_static_IP:9090
    
## 사용 방법
아래 가이드 참조 
  * https://sk-paas.atlassian.net/wiki/spaces/DOCS/pages/295022
