---
layout: post
title: "EC2 System Manager - EC2와 On-premises 시스템 설정 및 관리"
translate: 2017-01-10
author: Jeff Barr
about_stub: 이상록
profile_picture:
author_site:
categories: [Amazon EC2](https://aws.amazon.com/blogs/aws/category/amazon-ec2/), [EC2 Systems Manager](https://aws.amazon.com/blogs/aws/category/ec2-systems-manager/), [Windows](https://aws.amazon.com/blogs/aws/category/windows/)
---

#EC2 System Manager - EC2와 On-premises 시스템 설정 및 관리
  
지난 해 EC2 Run Command 를 [소개하고](https://aws.amazon.com/blogs/aws/new-ec2-run-command-remote-instance-management-at-scale/) 
EC2 인스턴스와 [하이브리드 및 교차 클라우드 환경](https://aws.amazon.com/blogs/aws/ec2-run-command-update-hybrid-and-cross-cloud-management/)에서 
원격 인스턴스 관리를 대규모로 수행하는 방법을 보여줬습니다. 
이 과정에서 [리눅스 인스턴스에 대한 지원](https://aws.amazon.com/blogs/aws/ec2-run-command-update-now-available-for-linux-instances/)을 추가해 EC2 Run Command를 광범위하게 적용할 수 있고 
매우 유용한 관리 도구로 만들었습니다.  
  
**가족이 되신 것을 환영합니다**  
[Werner](https://twitter.com/Werner)가 [AWS re:Invent](https://reinvent.awsevents.com/)에서 [EC2 System Manager](https://aws.amazon.com/ec2/systems-manager/)를 
발표했고 드디어 이것에 대해 말하게 되었습니다!  
  
EC2 Run Command의 향상된 버전과, 여덟 개의 유용한 기능이 포함된 새로운 관리 서비스입니다. 
EC2 Run Command 처럼 윈도우, 리눅스를 실행하는 서비스와 인스턴스로 이루어진 교차 클라우드 환경 및 하이브리드 환경을 지원합니다. 
[AWS Management Console](https://console.aws.amazon.com/)을 열고 관리할 인스턴스를 선택한 다음 
수행 하고싶은 작업을 정의하기만 하면 됩니다(API와 CLI에서도 접근할 수 있습니다).  
  
다음은 개선된 기능과 새로운 기능에 대한 개요입니다:  
  
**Run Command** - 이제 명령 실행 속도를 제어하고 에러 비율이 높아졌을 때 명령 실행을 중지할 수 있습니다.  
  
**State Manager** - 규칙적인 간격으로 적용되는 정책을 통해 정의된 시스템 설정을 유지합니다.  
  
**Parameter Store** - 라이센스 키, 비밀번호, 사용자 목록 및 다른 값들을 위한 (암호화 가능한)중앙 집중 저장소를 제공합니다.  
  
**Maintenance Window** - 업데이트 설치와 기타 시스템 유지 관리를 위한 기간을 지정하세요.  
  
**Software Inventory** - 각 인스턴스에서 자세한 소프트웨어 사항들과 사용자가 정의한 추가사항까지 포함한 설정 목록을 수집합니다.  
  
**AWS Config Integration** - 새로운 Software Inventory 기능과 함께 [AWS Config](https://aws.amazon.com/config/)는 software inventory의 변화를 인스턴스에 기록할 수 있습니다.  
  
**Patch Management** - 인스턴스의 패치 과정을 단순화하고 자동화합니다.  
  
**Automation** - AMI 구축과 기타 반복적인 AMI 관련 작업을 단순화합니다.  
  
각각을 살펴봅시다..  
  
**Run Command 개선 사항**  
이제 동시에 실행되는 명령의 숫자를 제어할 수 있습니다. 
명령이 내부 업데이트나 서버 패치와 같이 제한적인 공유 리소스들을 참조하고, 너무 많은 리퀘스트로 
인한 오버로딩을 피하고 싶은 상황에서 유용합니다.  
  
이 기능은 현재 CLI와 API로 접근할 수 있습니다. 다음은 CLI에서 동시실행을 2로 제한하는 예제입니다:  
```bash
$ aws ssm send-command \
  --instance-ids "i-023c301591e6651ea" "i-03cf0fc05ec82a30b" "i-09e4ed09e540caca0" "i-0f6d1fe27dc064099" \
  --document-name "AWS-RunShellScript" \
  --comment "Run a shell script or specify the commands to run." \
  --parameters commands="date" \
  --timeout-seconds 600 --output-s3-bucket-name "jbarr-data" \
  --region us-east-1 --max-concurrency 2
```
  
다음은 --instance-ids 대신 --targets를 지정하여 태그와 태그 값에 의해 작동하는 흥미로운 변형 예제입니다.  
```bash
$ aws ssm send-command \
  --targets "Key=tag:Mode,Values=Production" ... 
```
  
최대 에러 수 또는 에러 비율을 지정하는 옵션으로 에러를 반환하면 명령 실행을 멈출 수 있습니다:  
```bash
$ aws ssm send-command --max-errors 5 ... 
$ aws ssm send-command --max-errors 5% ...
```
  
**State Manager**  
State Manager는 문서를 따라 인스턴스를 정의된 상태로 유지하도록 도와줍니다. 
문서를 만들어 타겟 인스턴스 집합과 연결한 다음, 문서를 실행해야 하는 시간과 빈도를 지정하기 위한 연관성을 생성합니다. 
다음은 요일 파일의 메세지를 업데이트하는 문서입니다.  
![1](https://media.amazonwebservices.com/blog/2016/rc_create_doc_3.png)  
  
그리고 연관성은 이와 같습니다(태그를 사용하기 때문에 현재 인스턴스 및 
같은 방식으로 태그된 나중에 만들어질 다른 인스턴스에도 적용됩니다):  
![2](https://media.amazonwebservices.com/blog/2016/rc_create_assoc_1.png)  
  
태그를 사용해서 타겟을 지정하면 미래에도 사용할 수 있으며 동적 오토 스케일 환경에서도 기대했던 대로 작동하게 합니다. 
모든 연관성을 볼 수 있으며 새로운 연관성을 선택하고 Apply Association Now를 클릭해 실행할 수 있습니다.  
![3](https://media.amazonwebservices.com/blog/2016/rc_see_assoc_1.png)  
  
**Parameter Store**  
이 기능은 라이센스 키, 비밀번호 및 인스턴스에 배포하려는 기타 데이터의 저장 및 관리를 단순화합니다. 
각 매개변수는 형(문자열, 문자열 리스트, 보안 문자열)이 지정되어 있고 암호화된 형태로 저장할 수 있습니다. 다음은 매개변수를 생성하는 방법입니다:  
![4](https://media.amazonwebservices.com/blog/2016/rc_create_param_2.png)  
  
다음은 커맨드에서 매개변수를 참조하는 방법입니다:  
![5](https://media.amazonwebservices.com/blog/2016/rc_use_param_3.png)  
  
**Maintenance Window**  
이 기능은 업데이트 설치와 기타 시스템 유지관리를 위한 시간을 지정할 수 있게 합니다. 
다음은 매 주 토요일에 4시간동안 열리는 시간을 생성하는 방법입니다:  
![6](https://media.amazonwebservices.com/blog/2016/rc_create_maint_1.png)  
  
창을 생성한 후에는 인스턴스 Id 또는 태그로 창에 인스턴스 집합을 할당해야 합니다:  
![7](https://media.amazonwebservices.com/blog/2016/rc_reg_maint_target_1.png)  
  
그리고 나서 유지관리 기간 동안 수행할 작업을 등록해야 합니다. 
예를 들어 리눅스 쉘 스크립트를 실행할 수 있습니다:  
![8](https://media.amazonwebservices.com/blog/2016/rc_reg_maint_task_2.png)  
  
**Software Inventory**  
이 기능은 소프트웨어와 인스턴스 집합의 설정에 대한 정보를 수집합니다. 
이 정보에 접근하기 위해 Managed Instance와 Setup Inventory를 클릭합니다:  
![9](https://media.amazonwebservices.com/blog/2016/rc_find_managed_instances_1.png)  
![10](https://media.amazonwebservices.com/blog/2016/rc_setup_inventory_1.png)  
  
인벤토리를 설정하면 AWS 소유 문서와 인스턴스 집합 간에 연관성이 생깁니다. 
타겟을 선택하고 일정을 설정하고 목록화 할 항목의 유형을 확인한 다음 Setup Inventory를 클릭하기만 하세요:  
![11](https://media.amazonwebservices.com/blog/2016/rc_actually_setup_inventory_1.png)  
  
인벤토리가 실행된 후에는 인스턴스를 선택하고 인벤토리를 클릭해서 결과를 고나찰할 수 있습니다:  
![12](https://media.amazonwebservices.com/blog/2016/rc_see_instance_inventory_1.png)  
  
더 분석하기 위해서 결과를 필터링할 수 있습니다. 
예를 들어 개발 도구와 라이브러리만 보기 위해 AWS 요소 목록을 필터링 할 수 있습니다:  
![13](https://media.amazonwebservices.com/blog/2016/rc_inv_just_dev_1.png)  
  
모든 관리 인스턴스에서 인벤토리 기반 쿼리를 실행할 수 있습니다. 
다음은 4.6 이전 버전의 .NET을 실행하고 있는 Windows Server 2012 R2 인스턴스를 찾는 방법입니다:  
![14](https://media.amazonwebservices.com/blog/2016/rc_man_query_1.png)  
  
**AWS Config Integration**  
인벤토리의 결과는 [AWS Config](https://aws.amazon.com/config/)까지 연결되어 어플리케이션, 
AWS 요소, 인스턴스 정보, 네트워크 설정, Windows 업데이트의 변화를 지속적으로 추적합니다. 
인스턴스 Config 타임라인 위에 있는 Managed instance information을 클릭해 이 정보에 접근할 수 있습니다:  
![15](https://media.amazonwebservices.com/blog/2016/rc_mii_button_1.png)  
![16](https://media.amazonwebservices.com/blog/2016/rc_config_info_1.png)  
  
하단의 세줄은 인벤토리 정보로 이어집니다. 다음은 네트워크 설정입니다:  
![17](https://media.amazonwebservices.com/blog/2016/ri_mii_inv_net_1.png)  
  
**Patch Management**  
이 기능은 Windows 인스턴스에 있는 운영체제를 최신으로 유지하게 도와줍니다. 
지정한 유지관리 기간 동안 패치를 적용하고 기준을 준수하여 수행됩니다. 
기준은 분류와 심각도에 따라 패치를 자동으로 승인하는 규칙과 
승인 또는 거부할 명시적인 패치 목록을 지정합니다.  
  
다음은 저의 기준입니다:  
![18](https://media.amazonwebservices.com/blog/2016/rc_create_patch_baseline_1.png)  
  
각 기준은 하나 또는 다수의 패치 그룹에 적용될 수 있습니다. 패치그룹에 들어있는 인스턴스는 
**Patch Group** 태그가 있습니다. 저는 그룹에 **Win2016**라는 이름을 붙였습니다:  
![19](https://media.amazonwebservices.com/blog/2016/rc_win_tagged_patch_group_1.png)  
  
그리고 값을 기준과 연결했습니다:  
![20](https://media.amazonwebservices.com/blog/2016/rc_mod_patch_group_1.png)  
  
다음 단계는 AWS-ApplyPatchBaseline 문서를 사용해 유지 관리 기간동안 패치의 적용을 조정하는 것입니다:  
![21](https://media.amazonwebservices.com/blog/2016/rc_reg_apply_patch_baseline_1.png)  
  
Managed Instances 목록으로 돌아가서 한 쌍의 필터를 사용해 패치가 필요한 인스턴스를 찾을 수 있습니다:  
![22](https://media.amazonwebservices.com/blog/2016/rc_patch_compliance_1.png)  
  
**Automation**  
마지막이지만 앞의 기능 못지않게 중요한 Automation 기능은 일반적인 AMI 구축과 업데이트 작업을 간소화합니다. 
예를 들어 AWS-UpdateLinuxAmi 문서를 사용해 매 달마다 새 Amazon Linux AMI를 만들 수 있습니다:  
![23](https://media.amazonwebservices.com/blog/2016/rc_auto_update_ami_1.png)  
  
다음은 이 automation이 실행되었을 때 일어나는 일을 보여줍니다:  
![24](https://media.amazonwebservices.com/blog/2016/rc_auto_update_1.png)  
  
**지금 사용할 수 있습니다**  
위에서 설명한 EC2 Systems Manager의 모든 기능을 지금 무료로 사용할 수 있습니다. 
당신이 관리하는 리소스에 대해서만 비용을 지불합니다.  
  
- [Jeff](https://twitter.com/jeffbarr);