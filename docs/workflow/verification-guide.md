# 검증 시스템 실행 방법

코드 품질 점검을 LLM 에게 맡긴다. 점검 항목은 세 저장소에 나뉘어 총 567건이고, 판정 대상은 backend 코드다.

```
커밋하면   로컬에서 Claude 가 본다   (/verify 를 쳐야 돈다)
푸시하면   CI 에서 gemini 가 본다    (자동)
둘 다      병합을 막지 않는다
```

병합을 막는 것은 커버리지(service 패키지 메서드 100%)와 SonarQube 신규 Blocker 0건뿐이다.

무엇이 언제 도는지는 [verification-workflow.md](./verification-workflow.md),
설계 근거는 [qa-llm-verification.md](../software-quality/qa-llm-verification.md) 에 있다.

## 준비

세 저장소를 **같은 부모 디렉터리에 나란히** 둔다. 로컬 검증이 상대 경로로 서로를 찾는다.

```bash
git clone https://github.com/LGU-2/.github.git common
git clone https://github.com/LGU-2/be.git       backend
git clone https://github.com/LGU-2/infra.git    infra
```

```
어딘가/
  common/     품질 속성 217건
  backend/    코드 관용 250건, 판정 대상
  infra/      인프라 제약 100건
```

`common` 이나 `infra` 가 없으면 backend 250건만 판정된다. 그래서 `/verify` 는 없으면 멈춘다.

## 로컬

커밋한 뒤 backend 디렉터리에서 친다.

```
/verify
```

범위를 바꾸려면 인자를 준다. 기본은 `HEAD~1..HEAD` 다.

```
/verify HEAD~3..HEAD
```

돌아가는 것:

```
1  ./gradlew check      커버리지와 정적 분석. 알림만
2  마지막 커밋의 diff
3  판정                 바꾼 파일에 해당하는 항목 전부
4  기록 저장            backend/docs/llm-review/<계정>_<YYYYMMDD-HHMMSS>_llm-review.md
```

작업 중 여러 번 돌려도 된다. 중간 상태에서 위반이 나오는 것은 정상이고, 기록은 초 단위로 구분되어 덮어쓰지 않는다.

## CI

PR 을 열거나 push 하면 자동으로 돈다. 할 일은 없다.

코멘트는 push 마다 새로 달리지 않고 **같은 것이 갱신된다.**
코멘트는 위반을 둘로 나눈다.

```
이 PR 이 만든 위반    펼쳐서 보여 준다. 이것만 고치면 된다
기존 부채            접어 둔다. 이 PR 의 책임이 아니다
```

## 결과 읽기

| 값 | 뜻 | 할 일 |
|---|---|---|
| `VIOLATION` | 위반이다 | 고친다 |
| `OK` | 충족했다 | 없음 |
| `NOT_APPLICABLE` | 이 변경과 무관하다 | 없음 |
| `INSUFFICIENT_EVIDENCE` | 판정할 파일을 못 읽었다 | 반복되면 `anchors.yml` 의 앵커 목록 보강 |
| `CONFLICTING_BASELINE` | 확정값이 문서마다 다르다 | 팀이 어느 쪽인지 정한다 |
| `UNJUDGED` | 물어보지 않았다 | 로컬로 다시 본다 |

`OK` 와 `UNJUDGED` 는 다르다. 통과한 것과 안 본 것을 섞지 않기 위해 나눠 둔다.

LLM 판정이라 오탐이 있고 병합을 막지 않으므로, 틀렸다고 판단되면 그냥 병합해도 된다.

## 무엇이 켜지는지 미리 보기

앵커 규칙을 고쳤을 때 의도한 항목이 켜지는지 확인한다. LLM 을 부르지 않으므로 몇 번이든 돌려도 된다.

```bash
python3 common/.github/llm-verify/run.py --mode judge --dry-run \
  --backend backend --common common --infra infra --base HEAD~1 --head HEAD
```

```
매칭 규칙   service
활성 항목   182건
  1단계    79건
  2단계    103건
앵커 파일   읽음 2, 부재 3, 실패 0
```

## 안 될 때

| 증상 | 조치 |
|---|---|
| `/verify` 가 시작하자마자 멈춘다 | `common`, `infra` 를 같은 부모 디렉터리에 clone |
| 활성 항목이 50건뿐이다 | 정상이다. 문서만 고쳤을 때 그렇다 |
| 같은 항목이 계속 `INSUFFICIENT_EVIDENCE` | `backend/.github/llm-verify/anchors.yml` 의 `anchors` 에 파일 추가 |
| 2단계가 매번 건너뛰어진다 | 코멘트의 사유를 보고 로컬로 대신 본다 |
| CI 워크플로가 빨갛다 | `GEMINI_API_KEY` 조직 시크릿 등록 |
| 같은 지적이 매 PR 마다 나온다 | `common/.github/llm-verify/known-conflicts.yml` 에 등록 |

**지금은 아직 판정할 대상이 없다.** backend 에 Java 코드가 없어 모든 규칙이 기본 집합으로 떨어지고,
`build.gradle` 에 JaCoCo 와 SonarQube 설정이 없어 빌드 게이트가 돌지 않는다.
