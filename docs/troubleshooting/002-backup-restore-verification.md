# [Troubleshooting] 백업 복구(Restore) 검증

## 상황

R2에 자동으로 쌓이고 있는 백업이 "실제로 복원 가능한 상태인가"는 별도로 검증된 적이 없었다.
백업이 성공적으로 생성되고 업로드되는 것과, 그 백업으로 실제 서비스를 복구할 수 있는 것은
다른 문제이기 때문에, 운영 중인 DB에 영향을 주지 않는 격리된 환경에서 실제 복구를 시도해
검증했다.

## 검증 과정

### 1. R2에서 최신 백업 파일 다운로드

로컬(WSL2)에 aws-cli가 설치되어 있지 않아, 처음에는 `amazon/aws-cli` 이미지로 임시 Pod를
띄워 다운로드를 시도했다. 다운로드 자체는 성공했지만 해당 이미지에 `gunzip`이 포함되어 있지
않아 압축 해제 단계에서 막혔다.

```
sh: gunzip: command not found
command terminated with exit code 127
```

최종적으로는 로컬에 aws-cli를 직접 설치(`awscliv2.zip` → `sudo ./aws/install`)하는 방식으로
전환했다. 매번 임시 Pod를 만드는 것보다 로컬에 CLI를 설치해두는 편이 반복 작업에 더 실용적
이었다.

### 2. 격리된 임시 PostgreSQL에 복원

운영 중인 `postgres-postgresql-0`을 직접 건드리지 않기 위해, 별도의 임시 Pod
(`postgres:18-alpine`)를 띄우고 그 안에 `restore_test`라는 별도 DB를 만들어 복원을 시도했다.

```bash
kubectl run restore-test-pg --image=postgres:18-alpine --restart=Never \
  --env="POSTGRES_PASSWORD=testpass" --env="POSTGRES_DB=restore_test" \
  -n default -- sleep 3600
```

### 3. 복원 실행 및 결과

```bash
psql -U postgres -d restore_test -f /tmp/backup.sql
```

아래 항목이 에러 없이 모두 정상적으로 수행되는 것을 확인했다.

- `CREATE TABLE` × 7 (전체 테이블 스키마 복구)
- `COPY` × 7 (테이블별 실제 데이터 복구)
- `setval` × 6 (시퀀스 값 복구 — auto increment 컬럼이 다음 insert 시 기존 데이터와 충돌하지
  않도록 정확한 현재 값으로 복원됨)
- `ALTER TABLE` × 9 (제약조건, 외래키 등 복구)
- `psql` 프로세스가 에러 없이 정상 종료

### 4. 정리

검증에 사용한 임시 리소스(Pod, 로컬 덤프 파일)는 모두 삭제해 운영 환경에 흔적을 남기지
않았다.

## 결론

현재 R2에 쌓이고 있는 백업은 스키마·데이터·시퀀스·제약조건을 포함해 완전한 형태로 복원
가능함을 확인했다. "백업이 만들어지고 있다"는 사실과 "그 백업으로 실제 복구가 된다"는 사실은
별개로 검증해야 한다는 원칙에 따라, 이번 검증을 통해 백업 파이프라인의 신뢰도를 실제로
확인했다.

## 교훈

- 백업 자동화를 구축했다고 해서 그 자체로 "복구 가능"이 보장되지는 않는다. 복구 절차를 최소
  한 번은 실제로 실행해봐야 그 백업이 쓸모 있다는 것을 알 수 있다.
- 운영 DB와 완전히 격리된 임시 환경(별도 Pod, 별도 DB명)에서 검증하면, 복구 테스트 자체가
  실제 서비스에 영향을 주는 사고로 이어질 위험 없이 반복적으로 수행할 수 있다.
- 도구 준비(이번 경우 aws-cli)는 매번 임시 환경을 만드는 것보다 로컬에 한 번 설치해두는 편이
  반복되는 운영/검증 작업에서 더 효율적이었다.
