💻 [CLI] 인스턴스 현황 조회 (운영자용)
```
Bash
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[*].{Name:Tags[?Key==`Name`]|[0].Value, InstanceID:InstanceId, PublicIP:PublicIpAddress, PrivateIP:PrivateIpAddress, State:State.Name}' \
    --output table
```


```
----------------------------------------------------------------------------
|                             DescribeInstances                            |
+----------------------+--------------+------------+-----------+-----------+
|      InstanceID      |    Name      | PrivateIP  | PublicIP  |   State   |
+----------------------+--------------+------------+-----------+-----------+
|  i-016d3edb1fbb68e1f |  MyLabServer |  10.0.1.71 |  None     |  stopped  |
+----------------------+--------------+------------+-----------+-----------+
```


### 1. 서버 전원 켜기 (CLI)
콘솔에서 클릭해도 되지만, 우리는 CLI로 해보겠습니다.

```
aws ec2 start-instances --instance-ids i-016d3edb1fbb68e1f
```


43.203.120.199