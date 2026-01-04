---
layout: post
title: "CloudWatch Logs에서 특정 log group의 로그를 S3으로 export하는 방법 + 최소 권한 원칙으로 S3 권한설정"
date: 2026-01-05 04:12:41 +0900
categories: [AWS]
---
### 1. 일반 export 과정
내가 날짜(시간)을 정하면 해당 로그가 그대로 s3으로 export 되는 기능이다.
![](/assets/img/cloudwatch-logs에서-특정-log-group의-로그를-s3으로-export하는-방법-최소-권한-원칙으로-s3-권한설정/image.png)
웹 콘솔에서 해당 log group → Export data to Amazon S3 → 날짜 설정 하고 내보낼 버킷을 설정한다.
![](/assets/img/cloudwatch-logs에서-특정-log-group의-로그를-s3으로-export하는-방법-최소-권한-원칙으로-s3-권한설정/image 2.png)<!-- {"width":506} -->
해당 s3 버킷에 성공적으로 export 된것을 확인할 수 있다.
![](/assets/img/cloudwatch-logs에서-특정-log-group의-로그를-s3으로-export하는-방법-최소-권한-원칙으로-s3-권한설정/image 4.png)<!-- {"width":356} -->
- `aws-logs-write-test`가 export 할 때마다 새로 갱신된다.이는 한번 export 할 때 export하는 범위가 넓으면 로그도 많고, 로그 파일 크기도 크기 때문에, 쓰기 작업 시작 전의 확인 절차로 보인다.\

- 아까 bucket prefix를 log1/로 정했었다. 해당 prefix에 정상적으로 저장되었으며, 현재 Task Id가 내가 최초로 정한 prefix 하위 bucket prefix에 포함되는 것을 볼 수 있다.
![](/assets/img/cloudwatch-logs에서-특정-log-group의-로그를-s3으로-export하는-방법-최소-권한-원칙으로-s3-권한설정/image 3.png)<!-- {"width":405} -->
![](/assets/img/cloudwatch-logs에서-특정-log-group의-로그를-s3으로-export하는-방법-최소-권한-원칙으로-s3-권한설정/image 5.png)
파일명이 000000.gz인데, 이는 로그파일이 너무 크면 분리하기 위함으로 보인다.
![](/assets/img/cloudwatch-logs에서-특정-log-group의-로그를-s3으로-export하는-방법-최소-권한-원칙으로-s3-권한설정/image 8.png)<!-- {"width":301} -->
조금 많은 로그를 export 했더니 파일이 분리되는 것을 알 수 있다.

또한 export는 리전당 동시에 1개까지 export 작업을 수행할 수 있어서, 동시에 두번 export 작업을 하면 에러가 발생한다.
![](/assets/img/cloudwatch-logs에서-특정-log-group의-로그를-s3으로-export하는-방법-최소-권한-원칙으로-s3-권한설정/image 6.png)<!-- {"width":264} -->
![](/assets/img/cloudwatch-logs에서-특정-log-group의-로그를-s3으로-export하는-방법-최소-권한-원칙으로-s3-권한설정/image 7.png)
3달정도의 로그를 export 했는데, 7분정도가 지났는데도 시작단계인 것을 볼 수 있다.
꽤 시간이 지나야 하는 듯 하다.

### 2. S3에서 최소 권한 원칙 적용
- 현재 source인 log group: `/aws/ec2/docker-logs`
- 현재 destination인 S3 버킷: `20260104`
1. public access 차단
   ![](/assets/img/cloudwatch-logs에서-특정-log-group의-로그를-s3으로-export하는-방법-최소-권한-원칙으로-s3-권한설정/image 9.png)
   우선 버킷의 public access 차단 탭에서 `Block all public access` 한다.
   이 버킷에 접근하는 리소스는 cloudwatch logs이기 때문에, “AWS 서비스 호출” 에 해당하며, public access에 해당하지 않기에 차단해도 상관없다.

2. bucket resource policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "allowCWLogsGetBucketACL",
            "Effect": "Allow",
            "Principal": {
                "Service": "logs.amazonaws.com"
            },
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::20260104",
            "Condition": {
                "StringLike": {
                    "aws:SourceArn": "arn:aws:logs:ap-northeast-2:6...7:log-group:/aws/ec2/docker-logs:*"
                }
            }
        },
        {
            "Sid": "allowCWLogsPutObject",
            "Effect": "Allow",
            "Principal": {
                "Service": "logs.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::20260104/*",
            "Condition": {
                "StringLike": {
                    "aws:SourceArn": "arn:aws:logs:ap-northeast-2:6...7:log-group:/aws/ec2/docker-logs:*"
                }
            }
        }
    ]
}
```
- 목표: cloudwatch logs가 s3의 `s3:GetBucketAcl`과 `s3:PutObject` 를 허용하게 하는 것.
  - `s3:GetBucketAcl`은 s3 버킷에게 현재 권한이 있는지 확인하는 용도로, 대상이 해당 버킷이다. 따라서 `arn:aws:s3:::20260104` 가 `Resource`가 된다.
  - `s3:PutObject`는 s3 버킷 내부에 데이터를 쓰는 용도로, 대상을 “해당 버킷 내부 object”로 지정해 주어야 한다. 따라서 `arn:aws:s3:::20260104/*` 가 `Resource`가 된다.
- 그럼 왜 굳이 `s3:GetBucketAcl`로 권한이 있는지 확인할까?
  - 배치 작업이기에, 작업을 시작하기 전에 우선 버킷에 실제로 쓸 수 있는지 확인해야 하기 때문이다.
  - GetBucketAcl로 확인 후에 실제로 `aws-logs-write-test`로 써지는지도 한번 더 체크한다.
- 실제로 CLI로 해 보면..
  - `aws s3api get-bucket-acl --bucket 20260104`
  - `--debug` 를 붙이면 실제 http 요청이 오고가는 것을 볼 수 있다. 
  - 만약 성공한다면 200 OK가 나온다.
    ![](/assets/img/cloudwatch-logs에서-특정-log-group의-로그를-s3으로-export하는-방법-최소-권한-원칙으로-s3-권한설정/image 10.png)
  - 만약 권한이 없다면.. 403 forbidden이 나온다.
    ![](/assets/img/cloudwatch-logs에서-특정-log-group의-로그를-s3으로-export하는-방법-최소-권한-원칙으로-s3-권한설정/image 11.png)
- 일반적인 s3 작업에서는 데이터를 준비하고, 데이터 삽입 시도 후 오류 발생 시 리턴하는 게 베스트 프랙티스이며, 이를 EAFP (It's easier to ask forgiveness than permission) 라 한다.
  - 실제로 SDK(boto3 등) 이 이러한 방식을 사용한다고 한다.
- 단 데이터 처리에 꽤 많은 CPU 용량이 소모되는 대용량 작업 같은 경우에는, 먼저 가능한지 여부를 판단하고 데이터를 가공하고 준비하는 게 효율적이고, 실제로 웹 콘솔에서 배치 작업 시작을 누르자마자 권한이 없다면 권한 관련 에러가 나오며 작업이 큐에 들어가지도 않는다.
  - 따라서 먼저 getbucketacl로 확인 후 실제로 aws-logs-write-test로 파일을 써본 후 처리를 시작하는 구조를 택한 것이다.