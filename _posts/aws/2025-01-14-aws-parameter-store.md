---
title: "AWS Parameter Store 사용하기"

toc: true
toc_sticky: true

categories:
    - AWS
tags:
    - AWS
    - Parameter Store
---

# AWS Parameter Store 사용하기

프로젝트를 진행하다 보면 외부에 노출되지 않아야 하는 민감 정보들이 있다. DB User & PW, JWT Secret key, OAuth id 등 다양한 정보들이 포함된다. 기존에는 Git Submodule을 통해 민감 정보를 분리했지만, 그닥 좋은 방법이라고 생각되지 않는다. GitHub Actions를 통해 Private 저장소에 접근하도록 하려면 전용 키가 필요한데, 이를 관리하는 것이 큰 부담 요소로 다가온다.

이번에는 AWS의 Parameter Store를 통해 환경변수를 관리해 보고자 한다.

Parameter store의 주요 장점은 다음과 같다.
- 10,000 파라미터까지 무료 제공
- 키-값 쌍으로 값을 저장
- KMS를 이용해 암호화된 값 저장
- IAM을 통해 접근 제어

## Parameter Store 추가하기

![image](https://github.com/user-attachments/assets/60389461-c586-47af-b9c7-d776cc33e1f1)

`AWS Systems Manager > Parameter store`로 이동한다. 검색 창에 `Parameter store`만 입력하면 '특징' 칸에 표시된다.

그리고 파라미터 생성 버튼을 눌러 페이지를 이동한다.

### 파라미터 이름

파라미터 이름은 `/`를 통해 계증 구조를 이룰 수 있다. 수많은 파라미터를 하나의 집합 목록으로 저장할 경우 관리가 복잡해질 수 있다. Spring Boot를 예로 들면 `application.yml` 파일을 만드는 것과 같다. 유사하게 `/`를 활용하여 관련 집합들을 묶어 추가하도록 한다.

### 파라미터 유형

데이터를 일반 문자열로 저장할 것인지, 암호화하여 저장할 것인지 선택할 수 있다. AWS Key Management Service(KMS) Customer Master Key(CMK)를 사용하여 파라미터값을 암호화한다. 때문에 KMS의 요금이 부과된다. [AWS Key Management Service 요금](https://aws.amazon.com/ko/kms/pricing/)

하지만 민감 정보를 저장하는 경우에는 꼭 `보안 문자열`을 사용하는 것이 좋다.

### 저장

필요한 정보를 모두 기재했다면, 이제 저장 버튼을 눌러 작업을 완료한다. 만약 정상적으로 저장되었는지 CLI를 통해 확인하고 싶다면, 다음의 과정을 거친다.

## CLI에서 Parameter store 요청해보기

기존 AWS CLI에 연결된 계정에 IAM 권한을 추가한다. 

>학습 목적이기 때문에 기존 계정에 권한을 추가한다. 원래는 Parameter store용 IAM 계정을 생성하여 권한을 주도록 하자.

### 특정 파라미터만 접근 가능하도록 하기

테스트용 계정이기 때문에 IAM에 `AmazonSSMFFullAccess` 권한을 추가한다. 만약 특정 키 집합만 접근할 수 있도록 권한을 설정하려면 다음과 같은 형식으로 설정한다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:GetParameter",
                "ssm:GetParameters",
                "ssm:GetParametersByPath"
            ],
            "Resource": "arn:aws:ssm:REGION:ACCOUNT-ID:parameter/test/spring*"
        },
        {
            "Effect": "Allow", 
            "Action": "ssm:DescribeParameters",
            "Resource": "*"
        }
    ]
}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:GetParameter",
                "ssm:GetParameters",
                "ssm:GetParametersByPath"
            ],
            "Resource": "arn:aws:ssm:REGION:ACCOUNT-ID:parameter/test/spring/*"
        },
        {
            "Effect": "Allow", 
            "Action": "ssm:DescribeParameters",
            "Resource": "*"
        }
    ]
}
```
`REGION`과 `ACCOUNT-ID`에는 실제 값으로 작성해야 한다. `Resource ARN`에 와일드카드를 사용하여 `parameter/test/spring` 경로로 시작하는 모든 파라미터를 조회할 수 있게 될 것이다.

[참고 자료](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-paramstore-access.html)

### CLI에서 요청하기

```bash
$ aws ssm get-parameter --name /test/spring/id
{
    "Parameter": {
        "Name": "/test/spring/id",
        "Type": "String",
        "Value": "test",
        "Version": 1,
        "LastModifiedDate": "2025-01-13T22:48:18.164000+09:00",
        "ARN": "arn:aws:ssm:ap-northeast-2:339712697347:parameter/test/spring/id",
        "DataType": "text"
    }
}
```


추가로 특정 키에만 접근할 수 있도록 권한을 설정했을 때를 확인해 보았다. `/test/spring2/id` 값을 추가해 두었고, `/test/spring/*`만 접근할 수 있는 권한을 가지고 있다.

```bash
$ aws ssm get-parameter --name /test/spring/id
{
    "Parameter": {
        "Name": "/test/spring/id",
        "Type": "String",
        "Value": "test",
        "Version": 1,
        "LastModifiedDate": "2025-01-13T22:48:18.164000+09:00",
        "ARN": "arn:aws:ssm:ap-northeast-2:339712697347:parameter/test/spring/id",
        "DataType": "text"
    }
}

$ aws ssm get-parameter --name /test/spring2/id

An error occurred (AccessDeniedException) when calling the GetParameter operation: User: arn:aws:iam::339712697347:user/github-action is not authorized to perform: ssm:GetParameter on resource: arn:aws:ssm:ap-northeast-2:339712697347:parameter/test/spring2/id because no identity-based policy allows the ssm:GetParameter action
```

특정 키값만 잘 가지고 오는 것을 확인해 볼 수 있다.

---

## 참고

[AWS Parameter store 사용하기](https://dublin-java.tistory.com/66)
