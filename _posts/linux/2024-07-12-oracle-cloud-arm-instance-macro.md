---
title: "Oracle Cloud Arm 인스턴스 생성 자동화"

toc: true
toc_sticky: true

categories:
    - Linux
tags:
    - Linux
    - Ubuntu
    - OracleCloud
---

# Oracle Cloud Arm 인스턴스 생성 자동화

AWS와 비교해서 Oracle Cloud의 장점은 프리티어의 평생 Arm 무료일 것이다. 최대 4코어, 24GB의 램을 사용할 수 있다는 것이 엄청난 기대감을 준다. 문제는 프리티어에게 제공되는 자리의 수가 정해져 있다는 것이다. 그 때문에 빈 자리가 생길 때까지 계속해서 생성을 시도해야 하는데, 보통 자리가 열리면 빠르면 2분 컷이라 시간 날 때마다 한다고 얻을 수 있는 것이 아니란 판단이 든다.

이렇기 때문에 우리는 인스턴스 생성을 자동으로 요청하는 작업을 한다.


## Oracle Cloud Cli 설치

보통 자동화를 하는 방법은 2가지다. php로 작성된 스크립트와 Oracle Cloud Cli를 사용하는 방법이다. 전자의 경우에는 php 7.4 버전을 사용하며 Ubuntu 20.04에서 가능했으나, 지금은 어째서인지 오류가 빈번한 것 같다. 때문에 이번에는 Oracle Cloud Cli를 사용해서 작업하고자 한다.


#### oci 설치

[github.com/oracle/oci-cli](https://github.com/oracle/oci-cli)를 참고하여 설치한다.

Ubuntu 기준 아래와 같이 터미널에 입력한다.
```bash
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"
```

> 만약 curl이 설치되어있지 않다면 설치한다. `sudo apt  install curl`

설치 과정 중 질문되는 path는 기본 값을 사용했다.


## 필요한 정보 찾기

매크로를 만들기 위해 필요한 정보들을 추출한다.

### 사용자 정보

먼저 오라클 클라우드에 접속한 다음 사진과 같이 `내 프로파일`을 누른다.

![image](https://github.com/user-attachments/assets/c4a07f0e-f5a0-4fd9-8e83-c86eb7d44754)

그런 다음 좌측 메뉴에서 `API 키`를 선택한다.

![image](https://github.com/user-attachments/assets/10a13366-210f-40f1-9408-acd117fac623)

`API키 추가` 버튼을 누른 다음에 프라이빗 키를 다운로드 한다. 이후 하단의 `추가` 버튼을 누른다.

그러면 아래 사진과 같이 정보가 표시된다. 여기서 필요한 정보인 `user`, `tenancy`, `region`는 따로 저장한다.

![image](https://github.com/user-attachments/assets/61d42efb-4e6b-4713-b749-3288d9f6b8ae)

### 인스턴스 생성을 통한 정보 획득

다음으로 인스턴스 생성 페이지로 이동한다. 아래와 같이 원하는 OS와 A1 Shape를 선택한다.

![image](https://github.com/user-attachments/assets/d30671b7-50e2-4eca-8afd-988402b97a02)

이후 네트워크를 생성한 적이 없다면 `기본 VNIC 정보` 항목에서 `새 가상 클라우드 네트워크 생성`을 선택하여 생성한다. 만약 아니라면 기존 것을 선택해도 무관하다.

이후 아래 사진과 같이 `SSH` 항목에서 `SSH 공용키`와 `전용키`를 모두 저장한다.

![image](https://github.com/user-attachments/assets/a472eee8-4f1b-44f4-b3af-d1ee0c586163)

이제 `F12`를 눌러 아래와 같이 `Network`탭과 `instance`를 검색어에 넣어 셋팅한다. 그리고 인스턴스 `생성` 버튼을 눌러 생성을 시도한다.

![image](https://github.com/user-attachments/assets/b0f72af6-ff47-4275-8afb-f0a2ba9b9014)

생성되었다면 그대로 쓰면 되지만, 생성이 되지 않을 것임을 알고 이 작업을 하고 있다. 요청 실패한 `instances/` 항목에 오른쪽 클릭을 하고 `Copy as cURL`을 눌러 복사한 뒤, 텍스트 편집기에 붙여넣는다.

![image](https://github.com/user-attachments/assets/fc443906-1803-403b-b8ed-b4239a8deafd)

그리고 아래 사진과 같이 검색을 통해 원하는 정보들을 따로 저장해야 한다.

<img width="565" alt="image" src="https://github.com/user-attachments/assets/157c5d24-5b60-450a-a150-f9bf143d5617">

저장할 항목은 다음과 같다.
- subnetid
- imageid
- ssh-rsa로 시작하는 ssh 키 값

이제 원하는 키 값은 모두 찾았다. 찾은 값들을 가지고 스크립트를 만들면 끝이 난다.


## 매크로 만들기


우선 시작하기 위해 설치한 `oci`를 설정해야 한다. 그러기 위해서는 앞서 `API Key`를 추가하며 받았던 파일을 `~/oracle` 디렉토리로 이동한다. 그 다음 아래와 같이 명령어를 통해 설정을 시작한다.

```bash
oci setup config
    This command provides a walkthrough of creating a valid CLI config file.

    The following links explain where to find the information required by this
    script:

    User API Signing Key, OCID and Tenancy OCID:

        https://docs.cloud.oracle.com/Content/API/Concepts/apisigningkey.htm#Other

    Region:

        https://docs.cloud.oracle.com/Content/General/Concepts/regions.htm

    General config documentation:

        https://docs.cloud.oracle.com/Content/API/Concepts/sdkconfig.htm


Enter a location for your config [/home/potato/.oci/config]: 
Enter a user OCID: [앞에서 찾은 user 값 입력]
Enter a tenancy OCID: [앞에서 찾은 tenancy 값 입력]
Enter a region by index or name(e.g.
1: af-johannesburg-1, 2: ap-chiyoda-1, 3: ap-chuncheon-1, 4: ap-dcc-canberra-1, 5: ap-dcc-gazipur-1,
6: ap-hyderabad-1, 7: ap-ibaraki-1, 8: ap-melbourne-1, 9: ap-mumbai-1, 10: ap-osaka-1,
11: ap-seoul-1, 12: ap-singapore-1, 13: ap-singapore-2, 14: ap-sydney-1, 15: ap-tokyo-1,
16: ca-montreal-1, 17: ca-toronto-1, 18: eu-amsterdam-1, 19: eu-dcc-dublin-1, 20: eu-dcc-dublin-2,
21: eu-dcc-milan-1, 22: eu-dcc-milan-2, 23: eu-dcc-rating-1, 24: eu-dcc-rating-2, 25: eu-dcc-zurich-1,
26: eu-frankfurt-1, 27: eu-frankfurt-2, 28: eu-jovanovac-1, 29: eu-madrid-1, 30: eu-madrid-2,
31: eu-marseille-1, 32: eu-milan-1, 33: eu-paris-1, 34: eu-stockholm-1, 35: eu-zurich-1,
36: il-jerusalem-1, 37: me-abudhabi-1, 38: me-abudhabi-3, 39: me-dcc-doha-1, 40: me-dcc-muscat-1,
41: me-dubai-1, 42: me-jeddah-1, 43: mx-monterrey-1, 44: mx-queretaro-1, 45: sa-bogota-1,
46: sa-santiago-1, 47: sa-saopaulo-1, 48: sa-valparaiso-1, 49: sa-vinhedo-1, 50: uk-cardiff-1,
51: uk-gov-cardiff-1, 52: uk-gov-london-1, 53: uk-london-1, 54: us-ashburn-1, 55: us-chicago-1,
56: us-gov-ashburn-1, 57: us-gov-chicago-1, 58: us-gov-phoenix-1, 59: us-langley-1, 60: us-luke-1,
61: us-phoenix-1, 62: us-saltlake-2, 63: us-sanjose-1): 3
Do you want to generate a new API Signing RSA key pair? (If you decline you will be asked to supply the path to an existing key.) [Y/n]: n
Enter the location of your API Signing private key file: /home/ubuntu/oracle/mail@mail.com_2024-01-__..pem
Warning: To increase security of your API key located at /home/ubuntu/oracle/mail@mail.com_2024-01-__..pem, append an extra line with 'OCI_API_KEY' at the end. For more information, refer to https://docs.oracle.com/iaas/Content/API/Concepts/apisigningkey.htm
Fingerprint: b6:4e:2e:68:3a:a5:a4:c5:ee:07:23:51:85:bc:c4:9c
Config written to /home/potato/.oci/config


    If you haven't already uploaded your API Signing public key through the
    console, follow the instructions on the page linked below in the section
    'How to upload the public key':

        https://docs.cloud.oracle.com/Content/API/Concepts/apisigningkey.htm#How2
```

이후 설정이 제대로 되었는지 확인하려면 다음의 명령어를 사용해서 정상적으로 값이 나오는지 확인하면 된다.

```bash
oci iam compartment list -c [앞에서 찾은 tenancy 값] --all
```

정상 작동을 확인한 다음에 `~/.oci` 위치에서 아래와 같이 여러 파일을 생성한다.

`availabilityConfig.json`
```bash
{
    "recoveryAction": "RESTORE_INSTANCE"
}
```

`instanceOptions.json`
```bash
{
    "areLegacyImdsEndpointsDisabled": false
}
```

`shapeConfig.json` 필요에 따라 수정한다.
```bash
{
    "ocpus": 4,
    "memoryInGBs": 24
}
```

그리고, 이전에 다운했던 ssh key를 `~/oracle` 디렉토리에 넣어준다.

이제 `~/.oci` 디렉토리에 매크로 스크립트를 작성한다.

`macro.sh`
```sh
#!/bin/bash

/home/ubuntu/bin/oci compute instance launch \
 --availability-domain [앞에서 찾은 리전 예시) vMFl:AP-CHUNCHEON-1-AD-1] \
 --compartment-id [앞에서 찾은 tenancy id] \
 --boot-volume-size-in-gbs [원하는 저장 용량] \
 --shape VM.Standard.A1.Flex \
 --subnet-id [앞에서 찾은 subnet id] \
 --assign-private-dns-record true \
 --assign-public-ip true \
 --availability-config file:///home/ubuntu/.oci/availabilityConfig.json \
 --display-name [instance 이름] \
 --image-id [앞에서 찾은 image id] \
 --instance-options file:///home/ubuntu/.oci/instanceOptions.json \
 --shape-config file:///home/ubuntu/.oci/shapeConfig.json \
 --ssh-authorized-keys-file /home/ubuntu/oracle/[ssh-key.key.pub]
```

> 대괄호를 포함하여 값을 변경하도록 한다.

이제 `~/`로 이동해서 로그를 남길 파일을 하나 생성한다. 

```bash
cd ~/
touch oci.log
```
`
마지막으로 크론탭 설정을 한다. 크론탭 설정은 `crontab -e` 명령어를 통해 접근할 수 있다.

```bash
* * * * * /bin/bash /home/ubuntu/.oci/macro.sh >> /home/ubuntu/oci.log 2>&1
```
해당 구문의 내용은 bash 쉘을 사용하여 `macro.sh`를 실행하고, `oci.log` 파일에 결과를 출력되도록 하는 것이다.

위와 같이 설정을 완료하면 1분마다 `macro.sh` 스크립트가 실행된다. 로그 기록은 `2>&1`를 통해 표준 출력과 오류 메시지가 모두 `oci.log` 파일에 기록된다. 때문에 정상 작동을 했다면 보통은 arm 인스턴스 자리가 없어 오류 메시지가 남게 된다. 만약 다른 방법으로 확인하고 싶다면, 매크로가 시작될 쯤 Oracle Cloud 홈페이지에서 인스턴스 생성을 시도하면 `Too many requests for the user` 라는 오류를 하단에 반환한다.

만약 실패할 경우 다음의 형식의 오류 메시지가 남겨진다.

```js
ServiceError:
{
    "client_version": "Oracle-PythonSDK/2.129.2, Oracle-PythonCLI/3.44.2",
    "code": "InternalError",
    "logging_tips": "Please run the OCI CLI command using --debug flag to find more debug information.",
    "message": "Out of host capacity.",
    "opc-request-id": "-",
    "operation_name": "launch_instance",
    "request_endpoint": "POST -",
    "status": 500,
    "target_service": "compute",
    "timestamp": "-",
    "troubleshooting_tips": "See [https://docs.oracle.com/iaas/Content/API/References/apierrors.htm] for more information about resolving this error. If you are unable to resolve this issue, run this CLI command with --debug option and contact Oracle support and provide them the full error message."
}
```

성공한 적이 없어 성공에 대한 메시지는 필자도 모른다.

만약 매크로가 시작될 때 매크로가 실행된건지 여부가 궁금하다면 다음과 같이 출력 문구를 추가한다.

`macro.sh`
```bash
#!/bin/bash

echo start

/home/ubuntu/bin/oci compute instance launch \
... (이후 생략)
```

이렇게 상단에 `echo start`를 추가하면 매크로가 시작됨과 동시에 로그 파일에 start 라는 문구가 남게 된다.