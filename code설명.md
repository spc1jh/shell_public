# 패턴 상세 흐름

> 비유나 일반론이 아니라 **`src/team5`에 실제로 있는 클래스·메서드·실행 로그**만으로 설명한다.
> 각 패턴마다 같은 순서로 정리한다.
> ① 무엇이 바뀌는가 / 무엇을 보호하는가 → ② 코드 속 역할 → ③ 호출이 실제로 흐르는 순서 → ④ 변경 시나리오에서 추가된 코드 vs 건드리지 않은 코드 → ⑤ 코드를 읽다 헷갈리기 쉬운 지점 → ⑥ 이 패턴을 쓰지 않았다면 (가정 코드와 비교)
>
> ⑥의 "쓰지 않은 코드"는 실제로 만든 것이 아니라 **비교를 위한 가정 코드**다. 클래스 이름과 시나리오는 우리 것을 그대로 썼다.

실행 로그는 `java -Dstdout.encoding=UTF-8 -cp <out> team5.Main <시나리오번호>` 결과를 그대로 옮겼다. (콘솔이 CP949이면 옵션 없이 실행)

---

## 0. 한 사건이 지나가는 전체 호출 경로

시나리오 4(사기 75점 → 사기조사보류)를 기준으로, 어떤 패턴 코드가 어느 순간에 동작하는지 먼저 본다.

| 순서 | 호출 (실제 코드) | 동작하는 패턴 |
|---|---|---|
| 1 | `new Claim(id, location, vehicleType, workflow)` → `state = workflow.first()` | State (시작 상태 = 접수) |
| 2 | `server.run(new AssignCommand(claim, candidates, strategy))` | Command (Invoker가 실행) |
| 3 | `AssignCommand.execute()` → `strategy.explain()` → `strategy.select()` | Strategy (후보 비교·선택) |
| 4 | `claim.assign(chosen)` → `state.assign(claim, chosen)` → `ReceivedState.assign` | State (상태 객체가 판단) |
| 5 | `c.setAdjuster(a); c.moveNext();` → `workflow.next(state)` | State + Workflow (접수 → 배정) |
| 6 | `notifyAdjuster(...)` → `FieldTerminal` 생성 | Command (Sender 쪽) |
| 7 | `terminal.submit(new RegisterResultCommand(...))` | Command (온라인이면 즉시, 오프라인이면 큐) |
| 8 | `claim.proceed()` ×2 → 조사중 → 산정대기 | State |
| 9 | `claim.inspectFraud(inspector)` → `PendingCalcState.inspectFraud` | State |
| 10 | `inspector.inspect(claim)` → `element.accept(rule)` → `rule.visit(this)` | Visitor |
| 11 | `report.suspicious()` → `c.moveTo(new FraudHoldState())` | State (정상 경로 밖 전이) |
| 12 | `claim.pay()` → `FraudHoldState` 기본 구현 → `거절(보류 중)` | State (기본 거절) |

핵심은 **2~3번(Command·Strategy)이 "누가 담당할까"를 처리하고, 4~5·8·11~12번(State)이 "지금 해도 되나"를 판단하며, 10번(Visitor)이 "의심스러운가"를 계산**한다는 점이다. 서로의 내부를 모른 채 이어진다.

---

## 1. State — "지금 이 일을 해도 되는가"를 상태 객체가 답한다

### ① 무엇이 바뀌고 무엇을 보호하나
- 바뀌는 것: 상태 종류와 순서 (영상조사대기가 새로 끼어든다)
- 보호하는 것: **기존 상태들의 코드와, "접수 상태에서는 지급이 안 된다" 같은 업무 규칙이 한곳에 모여 있는 구조**

### ② 코드 속 역할
| 역할 | 클래스 | 하는 일 |
|---|---|---|
| Context | [Claim.java](src/team5/Claim.java) | 업무 요청 6개(`assign/reassign/registerResult/proceed/inspectFraud/pay`)를 **`state`에 그대로 넘기기만** 한다 |
| State 인터페이스 | [ClaimState.java](src/team5/ClaimState.java) | 6개 메서드의 **기본 구현이 전부 `deny()`** |
| 구체 상태 | `ReceivedState` 등 | 자기 단계에서 허용되는 메서드만 override |
| 순서 관리 | [ClaimWorkflow.java](src/team5/ClaimWorkflow.java) | "다음 상태가 무엇인지"를 한 곳에서 가진다 |

`Claim`의 위임부는 이게 전부다.

```java
public boolean assign(Adjuster a)      { return state.assign(this, a); }
public boolean pay()                   { return state.pay(this); }
public void moveNext()                 { moveTo(workflow.next(state)); }
```

`Claim` 안에는 `if (state == 접수)` 같은 분기가 **한 줄도 없다.**

### ③ 호출 흐름 A — 접수 상태에서 `pay()`  (시나리오 2)
```
claim.pay()
 └ state.pay(claim)                       ← state는 ReceivedState
    └ ReceivedState는 pay를 override 안 함
       └ ClaimState.pay 기본 구현 → deny("pay")
          └ "  pay() → 거절(조사 전)"      ← denyReason()은 ReceivedState가 "조사 전"으로 구현
```
허용 목록을 따로 두지 않고 **"override 하지 않은 것 = 거절"** 로 만든 것이 핵심이다. 새 작업을 `ClaimState`에 추가해도 기존 상태는 자동으로 거절 처리된다.

### ③ 호출 흐름 B — 접수 → 배정 전이  (시나리오 5, 로그 일부)
```
AssignCommand.execute()
 └ claim.assign(chosen)
    └ ReceivedState.assign(c, a)
       ├ c.setAdjuster(a)                  ← 담당자 기록
       └ c.moveNext()
          └ moveTo( workflow.next(state) ) ← 순서표에서 "접수 다음 = 배정"을 찾는다
             └ "  상태 전이: 접수 → 배정"
```
`ReceivedState`는 **다음 상태가 배정이라는 사실을 모른다.** `moveNext()`만 부른다. 그래서 순서가 바뀌어도 이 클래스는 그대로다.

### ④ 변경 시나리오 — 영상조사대기 (시나리오 5)
추가된 코드: [ext/VideoWaitState.java](src/team5/ext/VideoWaitState.java) 한 파일, 그리고 워크플로를 만드는 곳의 한 줄.
```java
ClaimWorkflow.standard().insertAfter(InvestigatingState.class, new VideoWaitState())
```
건드리지 않은 코드: `InvestigatingState`, `PendingCalcState`, `Claim` 전부.

그런데 어떻게 `InvestigatingState.proceed()`가 영상조사대기로 가는가? 이 메서드는 원래부터 `c.moveNext()`만 부르기 때문이다. 순서표(`order`)가 바뀌면 "조사중 다음"이 달라질 뿐이다.
```
상태 전이: 배정 → 조사중
상태 전이: 조사중 → 영상조사대기      ← InvestigatingState 코드는 그대로
pay() → 거절(영상조사 대기)          ← VideoWaitState는 pay를 override 하지 않았으니 기본 거절
조사 결과 등록: 블랙박스 영상 분석: 후면 충격 확인   ← VideoWaitState가 registerResult만 허용
상태 전이: 영상조사대기 → 산정대기
```

### ⑤ 헷갈리기 쉬운 지점
1. **`moveNext()` vs `moveTo()`**
   - `moveNext()`: 정상 경로. 순서표가 다음을 정한다.
   - `moveTo(x)`: 정상 경로 밖. `PendingCalcState`가 `new FraudHoldState()`로 직접 지정한다. 사기조사보류는 순서표에 없다.
2. **`getClass() ==` 비교** ([ClaimWorkflow.java](src/team5/ClaimWorkflow.java)의 `next`, `insertAfter`): 상태 객체를 `new`로 만들어 쓰므로 객체 동일성이 아니라 "클래스 종류"로 위치를 찾는다.
3. **`undo`에서 `restore()`를 쓰는 이유**: `moveTo`를 쓰면 "상태 전이" 로그가 또 찍힌다. 되돌리기는 출력 없이 필드만 바꾼다.

### ⑥ 이 패턴을 쓰지 않았다면 (가정 코드와 비교)

`Claim`이 상태를 `enum`으로 들고 있고, 업무 요청마다 `switch`로 분기한다고 가정해 보자.

```java
// 가정: State 패턴 없이 작성한 Claim
public boolean pay() {
    switch (status) {
        case RECEIVED:      System.out.println("pay() → 거절(조사 전)");      return false;
        case ASSIGNED:      System.out.println("pay() → 거절(조사 전)");      return false;
        case INVESTIGATING: System.out.println("pay() → 거절(조사 중)");      return false;
        case PENDING_CALC:  System.out.println("pay() → 거절(사기 점검 전)"); return false;
        case FRAUD_HOLD:    System.out.println("pay() → 거절(보류 중)");      return false;
        case APPROVED:      status = CLOSED; return true;
        ...
    }
}
// assign(), reassign(), registerResult(), proceed(), inspectFraud() 도 같은 모양의 switch
```

| 비교 항목 | 쓰지 않았을 때 | 우리 코드 |
|---|---|---|
| 분기 개수 | 업무 요청 6개 × 상태 7개 = 최대 42개 분기가 `Claim` 한 클래스에 모인다 | 상태 클래스 7개에 나뉘고, 각 클래스는 **자기 단계에서 허용되는 것만** 쓴다 |
| 영상조사대기 추가 | `switch`가 있는 **6개 메서드를 전부** 열어 `case VIDEO_WAIT`를 넣는다. `proceed()`의 `INVESTIGATING → PENDING_CALC`도 `VIDEO_WAIT`로 고친다 | 파일 1개 추가 + `insertAfter` 한 줄 |
| 한 곳을 빠뜨리면 | `pay()`에 `case VIDEO_WAIT`를 안 넣으면 `default`로 빠져 **영상조사 전인데 지급이 될 수 있다.** 컴파일러는 알려 주지 않는다 | 기본 구현이 **거절**이라, 빠뜨려도 막히는 쪽으로 실패한다 (시나리오 5의 `pay() → 거절(영상조사 대기)`) |
| "다음 상태" 지식 | 모든 `case`에 다음 상태가 박혀 있어 순서를 바꾸려면 여러 곳을 찾아 고친다 | `ClaimWorkflow`의 `order` 목록 한 곳 |
| 거절 문구 | 분기마다 문구를 직접 씀 (오타·불일치) | `denyReason()` 한 줄 + `deny()` 공통 처리 |
| 검증 | 한 클래스가 모든 상태를 알아서 테스트 조합이 커진다 | 상태 클래스별로 "이 상태에서 허용되는 작업"만 확인 |

핵심은 **분기가 늘어서 보기 싫다**가 아니라, **변경 때 고쳐야 할 곳이 흩어져 있어 하나라도 놓치면 업무 규칙이 뚫린다**는 점이다.

---

## 2. Strategy — "누구를 배정할까"만 갈아 끼운다

### ① 무엇이 바뀌고 무엇을 보호하나
- 바뀌는 것: 배정 기준 (거리 → 전문분야 → 업무량 → AI 판독)
- 보호하는 것: **배정·재배정을 실행하고 이력을 남기는 Command 코드, 그리고 Claim**

### ② 코드 속 역할
| 역할 | 클래스 |
|---|---|
| Strategy 인터페이스 | [AssignmentStrategy.java](src/team5/AssignmentStrategy.java) — `select()`가 핵심, `rule()/evaluate()/explain()`은 로그용 |
| 구체 전략 | `DistanceFirstStrategy`, `SpecialtyFirstStrategy`, `WorkloadBalancedStrategy`, `ext/AiBasedStrategy` |
| 사용하는 쪽 (Context) | `AssignCommand`, `ReassignCommand` — 필드로 `AssignmentStrategy strategy`만 가진다 |

`AssignCommand.execute()`가 전략과 만나는 부분은 두 줄이다.
```java
strategy.explain(claim, candidates);                           // 비교 값 출력
Adjuster chosen = strategy.select(claim, candidates).orElse(null);  // 선택은 전략에게
```
구체 전략 이름(`DistanceFirstStrategy`)은 `AssignCommand` 안에 **등장하지 않는다.**

### ③ 호출 흐름 — 거리 우선 선택 (시나리오 1~5 공통)
```
AssignCommand.execute()
 ├ strategy.explain(claim, candidates)
 │   └ 후보마다 strategy.evaluate(claim, a)  → "거리 1.4" 등
 ├ strategy.select(claim, candidates)
 │   └ DistanceFirstStrategy: candidates.stream().min( 거리 )   → K
 └ claim.assign(K)                           ← 여기부터는 State의 일
```
실제 로그:
```
[배정 로직] 거리 우선: 사고 현장과 가장 가까운 사람
  후보 K : 거리 1.4
  후보 L : 거리 11.3
  후보 M : 거리 5.7
배정: 거리 우선 → K
```
`select`가 선택하고, `explain`은 보여주기만 한다. (로그와 선택 기준이 어긋나지 않도록 `evaluate`와 `select`를 같은 전략 클래스에 둔 것이다.)

### ④ 변경 시나리오 — AI 판독 기반 배정 (시나리오 8)
추가된 코드: [ext/AiBasedStrategy.java](src/team5/ext/AiBasedStrategy.java) 한 파일.
```java
if ("SEVERE".equals(claim.aiDamageGrade())) {
    // 중대손상 전문가 중 가장 가까운 사람
}
return candidates.stream().min(Comparator.comparingInt(Adjuster::activeCases)); // 아니면 업무량 최소
```
건드리지 않은 코드: `AssignCommand`, `ReassignCommand`, `CommandInvoker`, `FieldTerminal`. 호출하는 쪽 한 줄만 `new AiBasedStrategy()`로 바뀐다.

같은 사고(AC-3008)로 두 정책을 비교한 결과:
```
기존 정책이었다면 (거리 우선)  → K
AI 판독 기반:
  후보 K : 중대손상 전문 아니오, 거리 1.4, 진행 5건
  후보 L : 중대손상 전문 아니오, 거리 11.3, 진행 3건
  후보 M : 중대손상 전문 예,   거리 5.7, 진행 1건
  배정: AI 판독 기반 → M
```
AI 판독이 `SEVERE`이므로 가까운 K 대신 전문가 M이 선택된다.

### ⑤ 헷갈리기 쉬운 지점
1. 전략은 **배정 가능 여부를 판단하지 않는다.** 접수 상태가 아닌데 배정하려는 요청은 `ReceivedState`가 아닌 다른 상태가 거절한다 (State의 책임).
2. `Optional<Adjuster>`를 돌려주는 이유: 후보가 비면 `AssignCommand`가 "배정 가능한 손해사정사 없음"을 출력하고 `false`를 돌려준다.
3. `ReassignCommand`는 같은 전략을 쓰되 **후보에서 현재 담당자를 뺀** 목록(`others`)을 넘긴다. 전략은 그 사실을 모른다.

### ⑥ 이 패턴을 쓰지 않았다면 (가정 코드와 비교)

배정 정책을 `enum`이나 문자열로 받아 `AssignCommand`와 `ReassignCommand` 안에서 직접 분기한다고 가정해 보자.

```java
// 가정: Strategy 없이 작성한 AssignCommand.execute()
Adjuster chosen;
switch (policy) {
    case DISTANCE:  chosen = min(candidates, 거리);      break;
    case SPECIALTY: chosen = 전문분야 일치 우선 후 거리;    break;
    case WORKLOAD:  chosen = min(candidates, 진행건수);   break;
}
// ReassignCommand.execute() 에도 같은 switch 를 한 번 더 둔다 (현재 담당자만 제외)
```

| 비교 항목 | 쓰지 않았을 때 | 우리 코드 |
|---|---|---|
| 중복 | 같은 `switch`가 배정·재배정 **두 Command에 복사**된다 | 두 Command가 같은 `AssignmentStrategy`를 부른다 |
| AI 판독 기반 추가 | `case AI_BASED`를 **두 곳 모두** 추가해야 한다. 재배정 쪽만 빠뜨리면 최초 배정은 AI, 재배정은 거리로 가서 **정책이 어긋난다** | 파일 1개 (`AiBasedStrategy`), 호출부에서 `new` 한 줄 |
| 로그 `[배정 로직]` | 정책마다 비교 값 출력 코드를 `switch` 안에 또 넣는다 | 각 전략이 `rule()`·`evaluate()`로 자기 기준을 설명한다 |
| Command의 책임 | "이력·취소" 코드와 "누구를 고를지" 계산이 한 메서드에 섞여, 정책을 고치다 이력 코드를 건드릴 위험이 있다 | `AssignCommand`는 `select()` 호출 결과만 쓴다 |
| 정책 비교 시연 | 시나리오 8처럼 같은 사고에 다른 정책을 적용해 보려면 분기값을 바꿔 다시 실행해야 한다 | `new DistanceFirstStrategy()`와 `new AiBasedStrategy()`를 같은 입력에 넘기면 끝 |

---

## 3. Command — "현장 지시"를 객체로 만들어 보관·기록·취소한다

### ① 무엇이 바뀌고 무엇을 보호하나
- 바뀌는 것: 통신 상태(정상/단절), 지시 종류(배정·재배정·결과등록), 지시를 되돌려야 하는 경우
- 보호하는 것: **현장에서 입력한 지시가 사라지거나 순서가 뒤바뀌지 않는 것**, 그리고 "무슨 지시를 누가 실행했는가"의 이력

### ② 코드 속 역할
| 역할 | 클래스 | 하는 일 |
|---|---|---|
| Command | [FieldCommand.java](src/team5/FieldCommand.java) | `execute()` / `undo()` / `describe()` 3개 |
| ConcreteCommand | `AssignCommand`, `ReassignCommand`, `RegisterResultCommand` | Receiver는 `Claim` |
| Invoker (서버) | [CommandInvoker.java](src/team5/CommandInvoker.java) | `run()`으로 실행, 성공분만 스택(`executed`)과 이력(`audit`)에 기록 |
| Sender (단말) | [FieldTerminal.java](src/team5/FieldTerminal.java) | 온라인이면 서버로 즉시, 오프라인이면 FIFO 큐(`pending`)에 보관 |

### ③ 호출 흐름 A — 정상 통신
```
terminal.submit(cmd)
 └ online == true → server.run(cmd)
    ├ cmd.execute()                  ← Claim 상태 객체가 허용하면 true
    ├ true  → executed.push(cmd), audit.add("실행: " + cmd.describe())
    └ false → 스택·이력에 남기지 않음
```

### ③ 호출 흐름 B — 터널에서 통신 단절 (시나리오 3)
```
terminal.setOnline(false)            → "통신 단절"
terminal.submit(결과1)               → pending.add, "단말 보관: 결과등록(...) (대기 1건)"
terminal.submit(결과2)               → pending.add, "단말 보관: ... (대기 2건)"
(이 시점 claim.results() = 0건)       ← 서버가 아직 실행하지 않았다
terminal.setOnline(true)
 └ while (!pending.isEmpty()) server.run(pending.poll())    ← 먼저 넣은 것부터
```
실제 로그:
```
단말 보관: 결과등록(AC-3003, 운전석 파손 확인) (대기 1건)
단말 보관: 결과등록(AC-3003, 보조석 파손 확인) (대기 2건)
서버에 기록된 조사 결과: 0건
[손해사정사 K 단말] 통신 복구 → 보관 2건 순서대로 전송
조사 결과 등록: 운전석 파손 확인
조사 결과 등록: 보조석 파손 확인
서버에 기록된 조사 결과: 2건
```
`pending`을 `ArrayDeque`로 쓰는 이유는 **제출 순서(FIFO)** 를 지키기 위해서다. 서버 이력도 같은 순서로 남는다.

### ③ 호출 흐름 C — 재배정 요청 (시나리오 9)
```
terminal.submit(new ReassignCommand(claim, 후보, 전략))
 └ server.run → ReassignCommand.execute()
    ├ others = candidates 중 claim.adjuster()가 아닌 사람
    ├ requester = claim.adjuster()          ← 요청자(K)를 기억. 사유는 받지 않는다
    ├ strategy.select(claim, others)        → M
    └ claim.reassign(M)                     ← 조사중/배정 상태 객체가 허용 판단
이력: 실행: 재배정(AC-3009, 거리 우선, 요청자 K)
```
`describe()`에 요청자를 넣었기 때문에 `audit`에 **누가 요청했는지만** 남는다.

### ④ 변경 시나리오와의 관계
Command는 변경 시나리오 3개와 직접 맞물리지는 않는다. 다만 **새 지시를 추가하는 방법**이 같다. `FieldCommand`를 구현한 클래스를 하나 만들면 `FieldTerminal`과 `CommandInvoker`는 그대로 쓴다. 지금 3개 Command가 이 규칙을 따른다.

### ⑤ 헷갈리기 쉬운 지점
1. **거절된 명령은 이력에 안 남는다.** `run()`이 `execute()`의 반환값을 보고 push/audit를 한다. 상태가 허용하지 않아 `false`가 나온 지시는 취소 대상도 아니다.
2. **`undo()`가 각 Command에 있는 이유**: 되돌리는 방법은 지시마다 다르다. 배정은 이전 상태·담당자 복원(`claim.restore`), 결과등록은 마지막 결과 제거(`removeLastResult`).
3. 단말 큐와 서버 스택은 **다른 것**이다. 큐(`pending`)는 아직 실행 전인 지시, 스택(`executed`)은 이미 실행된 지시다.

### ⑥ 이 패턴을 쓰지 않았다면 (가정 코드와 비교)

단말이 `Claim`의 메서드를 직접 부른다고 가정해 보자.

```java
// 가정: Command 없이 작성한 단말
void onSave(String text) {
    claim.registerResult(text);   // 통신이 끊겼다면? 예외이거나 호출이 사라진다
}
```

통신 단절을 처리하려면 단말이 "문구"와 "지시 종류"를 따로 저장하고, 복구 후 종류별로 다시 해석해야 한다.

```java
// 가정: 보관은 하되 객체가 아니라 데이터로 보관
record Saved(String type, Object... args) {}
void onReconnect() {
    for (Saved s : saved) {
        if (s.type().equals("RESULT"))        claim.registerResult((String) s.args()[0]);
        else if (s.type().equals("REASSIGN")) ...   // 지시 종류가 늘 때마다 분기 추가
        else if (s.type().equals("ASSIGN"))   ...
    }
}
```

| 비교 항목 | 쓰지 않았을 때 | 우리 코드 |
|---|---|---|
| 시나리오 3 (터널) | 호출이 유실되거나, 보관 로직을 지시 종류마다 따로 짜야 한다 | 어떤 지시든 `FieldCommand`이므로 `pending.add(command)` 한 줄로 보관, 복구 시 `server.run(pending.poll())` |
| 순서 보장 | 종류별로 따로 보관하면 결과등록과 재배정의 **상대 순서**가 깨질 수 있다 | 큐 하나(FIFO)에 지시를 그대로 넣어 제출 순서가 유지된다 |
| 서버 이력 | 호출하는 곳마다 `log.add(...)`를 직접 써야 해서 빠뜨리면 이력이 비고, 거절된 호출까지 남을 수 있다 | `CommandInvoker.run()` 한 곳에서 **성공한 것만** 기록 |
| 취소(undo) | 무엇을 되돌릴지 호출자가 따로 기억해야 한다 (이전 상태, 이전 담당자…) | 각 Command가 자기 `undo()`를 갖고, 서버는 `executed.pop().undo()`만 부른다 |
| 새 지시 추가 | 보관·전송·이력·취소 코드를 모두 찾아 `else if`를 추가한다 | `FieldCommand` 구현체 1개. 단말과 서버는 그대로 |

---

## 4. Visitor — "점검 항목"을 늘려도 사고 구성 요소를 건드리지 않는다

### ① 무엇이 바뀌고 무엇을 보호하나
- 바뀌는 것: 사기 점검 규칙 (단기 반복 청구 → 견적 과다 → 제휴 정비공장 반복 거래 …)
- 보호하는 것: **사고 구성 요소 클래스(AccidentInfo, Party, ClaimItem, PartnerEstimate)와 `FraudInspector`의 합산·판정 코드**

### ② 코드 속 역할
| 역할 | 클래스 | 하는 일 |
|---|---|---|
| Element | [ClaimElement.java](src/team5/ClaimElement.java) + 4개 record | `accept(ClaimVisitor v) { v.visit(this); }` |
| Visitor 인터페이스 | [ClaimVisitor.java](src/team5/ClaimVisitor.java) | 요소 종류별 `visit` 4개 |
| 기반 Visitor | [FraudRule.java](src/team5/FraudRule.java) | `visit` 4개를 **빈 구현**으로 두고, `flag()`·`drain()` 제공 |
| ConcreteVisitor | `RepeatedClaimRule`, `ExcessiveEstimateRule`, `ext/PartnerRepeatRule` | 관심 있는 `visit` 하나만 override |
| 순회·합산 | [FraudInspector.java](src/team5/FraudInspector.java) | 규칙 × 요소로 방문시키고 `Report` 생성 |

### ③ 호출 흐름 — 사기 점검 (시나리오 4)
```
claim.inspectFraud(inspector)
 └ PendingCalcState.inspectFraud(c, inspector)
    └ inspector.inspect(c)
       for rule in rules:                       ← 규칙 하나씩
         for element in claim.elements():       ← 요소 하나씩
           element.accept(rule)                 ← ① 요소가 호출
             └ rule.visit(this)                 ← ② 요소의 타입에 맞는 visit 선택 (더블 디스패치)
                └ 관심 있는 요소라면 flag("근거")
         found = rule.drain()                   ← 걸린 기록 수거 + 내부 비우기
         points = found 점수 합 → RuleResult(title, points, maxPoints, detail)
       Report(results, total, threshold=75, total >= threshold)
```
`RepeatedClaimRule`은 `visit(Party)`만 override하므로, `AccidentInfo`·`ClaimItem`·`PartnerEstimate`를 방문해도 `FraudRule`의 **빈 `visit`**이 실행되어 아무 일도 없다.
```java
@Override
public void visit(Party element) {
    if (element.recentClaims() >= 3) flag("최근 6개월 청구 " + element.recentClaims() + "회");
}
```
실제 로그:
```
[사기 점검]
  - 단기 반복 청구: 40점 / 40점  (최근 6개월 청구 4회)
  - 견적 과다: 35점 / 35점  (견적 450만원, 표준 250만원 (1.8배))
  합계 75점 (기준 75점 이상이면 보류)
→ 사기조사보류
상태 전이: 산정대기 → 사기조사보류
pay() → 거절(보류 중)
```
"75점 이상"은 `total >= threshold`다. 정확히 75점도 보류다.

### ④ 변경 시나리오 — 특정 제휴 정비공장 반복 거래 (시나리오 6)
추가된 코드: [ext/PartnerRepeatRule.java](src/team5/ext/PartnerRepeatRule.java) (`visit(PartnerEstimate)`만 override, 5회 이상 +30점) 그리고 점검기에 `.add(new PartnerRepeatRule())` 한 줄.

건드리지 않은 코드: `FraudInspector.inspect()`, 4개 Element, 기존 2개 규칙, `PendingCalcState`.

점검 항목 목록은 `Report.results()`가 **규칙 개수만큼** 자동으로 늘어나므로, `PendingCalcState`의 출력 반복문도 수정할 필요가 없다.

### ⑤ 헷갈리기 쉬운 지점
1. **`accept`와 `visit`이 왜 둘로 나뉘는가**: 자바는 오버로딩을 컴파일 시점의 타입으로 고른다. `element.accept(rule)` 안에서 `visitor.visit(this)`를 부르면 `this`가 `PartnerEstimate`로 확정되어 올바른 `visit`이 선택된다.
2. **`drain()`이 필요한 이유**: 같은 규칙 객체를 여러 사고 건에 재사용하므로, 수거 후 비우지 않으면 이전 사건의 결과가 섞인다.
3. **한계**: 새 **요소 종류**가 생기면 `ClaimVisitor`와 모든 규칙을 고쳐야 한다. 우리는 요소 4종이 고정이라는 가정으로 Visitor를 골랐다.
4. **전이는 Visitor의 일이 아니다.** `FraudInspector`는 점수만 계산하고, 보류로 보낼지는 `PendingCalcState`가 정한다. 그래서 점검 규칙을 바꿔도 상태 전이 규칙이 안 흔들린다.

### ⑥ 이 패턴을 쓰지 않았다면 (가정 코드와 비교)

두 가지 방식을 생각할 수 있다.

**방식 A. 점검기가 요소 타입을 직접 검사한다**
```java
// 가정: Visitor 없이 작성한 FraudInspector.inspect()
for (ClaimElement e : claim.elements()) {
    if (e instanceof Party p && p.recentClaims() >= 3)                                  total += 40;
    if (e instanceof PartnerEstimate pe && pe.amount() > pe.referenceAmount() * 1.5)    total += 35;
    if (e instanceof PartnerEstimate pe && pe.repeatDeals() >= 5)                       total += 30; // 새 규칙마다 한 줄
}
// 같은 요소(PartnerEstimate)에 규칙이 둘 이상 걸리므로 else-if 로 묶을 수도 없다
```
**방식 B. 요소가 스스로 점수를 계산한다**
```java
// 가정: 각 요소에 score() 를 둠
record Party(...) { int score() { return recentClaims >= 3 ? 40 : 0; } }
```

| 비교 항목 | 쓰지 않았을 때 | 우리 코드 |
|---|---|---|
| 제휴 정비공장 규칙 추가 | A: `inspect()`를 연다. 규칙이 늘수록 이 메서드가 계속 커진다 / B: `PartnerEstimate` 클래스를 연다 | 파일 1개 + `add()` 한 줄 |
| 책임 | A: 점수 합산·판정과 규칙 조건이 한 메서드에 섞인다 / B: **데이터 클래스가 사기 규칙을 알게 된다** | 요소는 `accept`만, 규칙은 `FraudRule`, 합산은 `FraudInspector` |
| 항목별 점수 출력 | 규칙별 이름·만점·근거를 따로 모아야 해서 시나리오 4의 `단기 반복 청구: 40점 / 40점 (최근 6개월 청구 4회)` 같은 출력을 만들기 번거롭다 | `title()`, `maxPoints()`, `flag(detail)`로 규칙이 스스로 제공 |
| 걸리지 않은 항목(0점) 표시 | 규칙 목록이 코드 곳곳에 흩어져 있어 "점검한 항목 전체"를 뽑기 어렵다 | `rules` 목록을 돌며 0점 항목도 `RuleResult`로 포함 |
| 한 규칙이 여러 요소를 보는 경우 | `if` 체인이 규칙 단위가 아니라 요소 단위로 쪼개져 규칙 하나가 여러 곳에 흩어진다 | 규칙 클래스 하나가 필요한 `visit`만 override |

**Visitor가 불리한 경우도 있다.** 새 **요소 종류**가 생기면 `ClaimVisitor`와 모든 규칙을 고쳐야 한다. 우리는 요소 4종이 고정이고 **규칙이 계속 늘어난다**는 가정(사기 규칙 증가)이므로 Visitor를 골랐다. 반대로 요소가 자주 늘고 규칙이 고정이라면 방식 B가 더 낫다.

---

## 5. 네 패턴의 접점 — 한 호출에서 책임이 넘어가는 지점

시나리오 9(재배정)에서 책임이 넘어가는 순서만 뽑으면 이렇다.

```
FieldTerminal.submit        (Command: 지시를 서버로 전달하거나 보관)
 → CommandInvoker.run       (Command: 실행·이력)
  → ReassignCommand.execute (Command: 요청자 기록, 후보 구성)
   → strategy.select        (Strategy: 누구를 고를까)
   → claim.reassign         (State 진입)
    → InvestigatingState.reassign   (State: 지금 허용? → setAdjuster)
```

| 질문 | 답하는 곳 |
|---|---|
| 이 지시를 지금 보낼 수 있나, 나중에 보내야 하나 | `FieldTerminal` (Command) |
| 누가 이 사고를 맡을까 | `AssignmentStrategy` (Strategy) |
| 지금 단계에서 이 작업이 허용되나 | `ClaimState` 구현체 (State) |
| 의심 점수는 몇 점인가 | `FraudRule` (Visitor) |
| 의심이면 어디로 보낼까 | `PendingCalcState` (State) |

**어떤 클래스도 다른 질문의 답을 계산하지 않는다.** 이 분리가 변경 시나리오 3개를 "클래스 한 개 추가 + 등록 한 줄"로 받아낸 이유다.

---

## 6. 변경 시나리오 3개 — 추가한 코드와 건드리지 않은 코드

| 변경 | 추가한 것 | 연결한 지점 | 건드리지 않은 것 | 패턴 없이 짰다면 고쳐야 할 곳 |
|---|---|---|---|---|
| 영상조사대기 | `ext/VideoWaitState` | `ClaimWorkflow.insertAfter(...)` | `InvestigatingState`, `PendingCalcState`, `Claim` | `Claim`의 `switch` 6개 메서드 전부, `proceed()`의 다음 상태 |
| AI 1차 판독 배정 | `ext/AiBasedStrategy` | 호출부의 `new AiBasedStrategy()` | `AssignCommand`, `ReassignCommand`, `Claim`의 위임 코드 | 배정 Command와 재배정 Command의 `switch` 두 곳 |
| 제휴 정비공장 반복 거래 | `ext/PartnerRepeatRule` | `FraudInspector.add(...)` | `FraudInspector.inspect`, 4개 Element, `PendingCalcState` | `FraudInspector.inspect()`의 `instanceof` 체인 (또는 `PartnerEstimate` 클래스) |

패턴 없이도 **동작은 한다.** 차이는 변경이 들어올 때 드러난다. 고칠 곳이 여러 군데로 흩어지고, 하나를 빠뜨려도 컴파일은 통과하므로 업무 규칙(영상조사 전 지급 금지, 재배정 정책 일치 등)이 조용히 깨질 수 있다.

참고: `Claim`에는 `aiDamageGrade` 필드와 `setAiDamageGrade()`가, `Adjuster`에는 `specialties()`가 변경 시나리오 및 사건 개요 출력 때문에 **나중에 추가**됐다. "변경 시나리오를 붙이려고 기존 클래스를 고쳤다"는 아니지만, 코어에 전혀 손대지 않았다고 말하면 부정확하므로 발표에서는 "패턴 구조(State 위임, Strategy 호출, Visitor 순회)는 수정하지 않았다"로 표현한다.
