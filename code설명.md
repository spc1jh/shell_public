# 패턴 상세 흐름

> 비유나 일반론이 아니라 **`src/team5`에 실제로 있는 클래스·메서드·실행 로그**만으로 설명한다.
> 각 패턴마다 같은 순서로 정리한다.
> ① 무엇이 바뀌는가 / 무엇을 보호하는가 → ② 코드 속 역할 → ③ 호출이 실제로 흐르는 순서 → ④ 변경 시나리오에서 추가된 코드 vs 건드리지 않은 코드 → ⑤ 코드를 읽다 헷갈리기 쉬운 지점

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

| 변경 | 추가한 것 | 연결한 지점 | 건드리지 않은 것 |
|---|---|---|---|
| 영상조사대기 | `ext/VideoWaitState` | `ClaimWorkflow.insertAfter(...)` | `InvestigatingState`, `PendingCalcState`, `Claim` |
| AI 1차 판독 배정 | `ext/AiBasedStrategy` | 호출부의 `new AiBasedStrategy()` | `AssignCommand`, `ReassignCommand`, `Claim`의 위임 코드 |
| 제휴 정비공장 반복 거래 | `ext/PartnerRepeatRule` | `FraudInspector.add(...)` | `FraudInspector.inspect`, 4개 Element, `PendingCalcState` |

참고: `Claim`에는 `aiDamageGrade` 필드와 `setAiDamageGrade()`가, `Adjuster`에는 `specialties()`가 변경 시나리오 및 사건 개요 출력 때문에 **나중에 추가**됐다. "변경 시나리오를 붙이려고 기존 클래스를 고쳤다"는 아니지만, 코어에 전혀 손대지 않았다고 말하면 부정확하므로 발표에서는 "패턴 구조(State 위임, Strategy 호출, Visitor 순회)는 수정하지 않았다"로 표현한다.
