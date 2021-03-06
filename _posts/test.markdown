---
layout: post
title: AWS Redshift를 이용해 S3 데이터 활용 및 SQL Client에서 쿼리 작업하기
date: 2021-04-30 17:09:00
description: AWS Redshift 기본
---

기본적으로 따라가기 좋은 예제는 공식 가이드였다.

https://docs.aws.amazon.com/redshift/latest/gsg/getting-started.html

여길 정독해서 실습을 따라해보면 되는데 Redshift 상의 쿼리 에디터 뿐 아니라 AWS 환경 밖에서 SQL client로도 연결을 진행해봤다.

# Step 1 : Set up prerequisites

AWS 계정을 만드는 단계라 회사에서 받았으면 패스, 아니면 만들면 됨
<br>
<br>

# Step 2 : Create an IAM role

Identity and Access Management = IAM

데이터에 접근하려면 우리가 만든 클러스터는 권한(permission)이 있어야 하고 그걸 증명하는데 쓰이는 게 IAM이다. 대표적인 예로 S3에 저장해둔 데이터를 일부분 Redshift로 빠르게 복사할 때 **COPY**라는 방법을 쓰는데 거기서 IAM 권한을 증명하게 된다. AWS에서는 IAM 권한을 만들어서 클러스터에 붙여서 사용하라고 추천하고 있다.

### **To create an IAM role for Amazon Redshift**

1. AWS IAM 콘솔로 들어간다. https://console.aws.amazon.com/iam/

2. 네비게이션 바에서 **역할** 선택
3. **역할 만들기** 선택
4. AWS 서비스 중 Redshift 선택
5. Redshift-Customizable 선택 후 다음
6. Attach permission policies 에서 AmazonS3ReadOnlyAccess 선택
7. Add Tags는 넘어감
8. Role name에서 **myRedShiftRole**을 입력한다.
9. Create Role
10. 방금 만든 권한을 클릭
11. Role ARN 값을 클립보드에 복사-> 나중에 COPY 기능쓸 때 필요함
<br>
<br>

# Step 3: Create a sample Amazon Redshift cluster

이제 비용이 발생하는 공간임

1. Redshift 콘솔로 들어감 
2. 오른쪽 상단에서 내가 클러스터를 만들고 싶은 AWS Region 선택. 딱히 없으면 걍 디폴트로 두면 됨
3. 네비게이션에서 **클러스터** 선택 후 **클러스터 생성** 
4. **클러스터 식별자**에 **examplecluster**라고 적고 노드 유형은 **dc2.large**, 노드수는 **2**개로 한다.
5. 데이터베이스 구성에는 마스터 이름과 암호를 설정해주고 스크롤을 조금 내려서 추가 구성 단계에서 기본값 사용을 체크해제하면 데이터베이스 구성 메뉴가 하나 더 나옴. 거기서 Database name은 디폴트로 dev일거고 Database port 가 5439일거다. 이걸 바꾸고 싶으면 수정작업을 진행하면 된다.
6. 아까 Step 2에서 만들어둔 IAM 을 여기서 쓸 차례다. 클러스터 권한에서 IAM 역할을 추가해준다(myRedshiftRole). 
7. 다른 디폴트는 그대로 두고 클러스터를 생성한다.
<br>
<br>

# Step 4: Authorize access to the cluster
Redshift 내에서 모든 쿼리 처리를 다 한다면 굳이 상관없겠지만(클러스터에 VPC-virtual private cloud 기반 Amazon VPC 서비스를 사용해 접근한다면 상관없음), SQL 클라이언트를 사용해 클러스터로 접근하려면 inbound 접근을 줘야 한다. 

1. 이 경우 vpc 콘솔로 들어간다.
 https://console.aws.amazon.com/vpc/ 
2. 보안 그룹 으로 들어가서 ID를 눌러준다.
3. 인바운드 규칙 편집 클릭
4. 규칙 추가
<br>
4-1. 일단 나는 사용자 지정 TCP, 포트범위는 위에서 클러스터 만들 때 바꿔준 포트(디폴트는 5439임)번호, 그리고 소스는 내 IP로 설정했음. 
5. 이렇게 하면 SQL 클라이언트(나의 경우 DBeaver)에서 잘 연결되는 것을 확인함.

# Step 5: Grant access to the query editor and run queries
aws 계정을 받을 때 만약 내 권한이 AdministratorAccess 면 사실상 모든 권한이 있기 때문에 굳이 설정해주지 않아도 됨. 하지만 그렇지 않은 경우 Redshift 내에서 쿼리 에디터로 테스트할 때 권한이 추가로 필요함.

* AmazonRedshiftQueryEditor
* AmazonRedshiftReadOnlyAccess

위 권한을 추가로 주면 됨

이제 쿼리 에디터로 간단한 쿼리를 쏴보겠음

1. 쿼리 편집기로 들어감
2. 데이터베이스에 연결 클릭- 새 연결 생성- 임시 자격 증명 - 클러스터 선택 - 데이터베이스 이름(디폴트는 dev) - 데이터베이스 사용자( 마스터 이름 적으면 됨)
3. 스키마에서 public 선택하면 dev db에 테이블을 만들 수 있음
4. 
{% highlight sql %}

create table shoes(
                shoetype varchar (10),
                color varchar(10));

insert into shoes values 
('loafers', 'brown'),
('sandals', 'black');

select * from shoes; 

{% endhighlight %}

5. sql 결과를 보고 데이터가 잘 들어갔나 확인
<br>
<br>


# Step 6: Load sample data from Amazon S3
일단 S3에 데이터를 먼저 옮겨 놓는 걸 해본다.

tickitdb.zip 파일을 로컬로 받아서 압축을 푼다. 개별 파일을 Amazon S3 **버킷 만들기**를 통해서 테스트 버킷을 하나 생성한다. 객체로 **tickit**을 만들고 그 안에다 tickit 개별 파일을 모두 업로드한다. 간혹 업로드 실패하는 파일도 있는데 이 경우 다시 업로드 하면 올라감. 사실 이 부분도 다 ETL 작업을 하면 자동으로 올리게 됨.

Redshift에서 ..
1. 테이블을 생성함.
* users
* venue
* category
* date
* event
* listing
* sales
2. COPY 명령으로 S3에서 데이터를 로드함
{% highlight sql %}

copy users from 's3://<myBucket>/tickit/allusers_pipe.txt' 
credentials 'aws_iam_role=<iam-role-arn>' 
delimiter '|' region '<aws-region>';

{% endhighlight %}

이런 식으로 Redshift의 빈 테이블 7개 테이블에 S3 데이터를 로드하면 됨


