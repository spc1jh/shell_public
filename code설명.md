# 5조 – 코드로 보는 패턴

실제 소스(`src/team5`, `src/team5/ext`)에서 패턴별 핵심 코드만 골랐다. 설명용으로 주석은 줄였고, 로직은 그대로다.

## 0. 한눈에 보기

| 패턴 | 바뀌는 것 | 핵심 파일 | 한 줄 요약 |
|---|---|---|---|
| State | 상태와 상태별 허용 작업 | `ClaimState`, `Claim`, `ReceivedState`, `ClaimWorkflow`, `PendingCalcState` | `Claim`은 상태에 위임만 하고, 각 상태가 허용 작업을 정한다 |
| Strategy | 손해사정사 배정 기준 | `AssignmentStrategy`, `DistanceFirstStrategy`, `AssignCommand` | 선택 기준이 인터페이스 뒤에 있다 |
| Command | 현장 지시, 통신 단절 | `FieldCommand`, `AssignCommand`, `CommandInvoker`, `FieldTerminal` | 지시를 객체로 만들어 실행·취소·이력·큐 보관 |
| Visitor | 사기 점검 규칙 | `ClaimElement`, `ClaimVisitor`, `FraudRule`, `RepeatedClaimRule`, `FraudInspector` | 요소는 고정, 규칙만 클래스로 추가 |

---

## 1. State: 상태가 행동을 결정한다

### 1-1. `ClaimState`: 기본은 전부 거절, 각 상태가 자기 작업만 허용 (`ClaimState.java`)

```java
public interface ClaimState {
    String name();
    String denyReason();

    default boolean assign(Claim c, Adjuster a)             { return deny("assign"); }
    default boolean reassign(Claim c, Adjuster a)           { return deny("reassign"); }
    default boolean registerResult(Claim c, String r)       { return deny("registerResult"); }
    default boolean proceed(Claim c)                        { return deny("proceed"); }
    default boolean inspectFraud(Claim c, FraudInspector i) { return deny("inspectFraud"); }
    default boolean pay(Claim c)                            { return deny("pay"); }

    default boolean deny(String operation) {
        System.out.println("  " + operation + "() → 거절(" + denyReason() + ")");
        return false;
    }
}
```

### 1-2. `Claim`: 판단하지 않고 현재 상태에 위임만 한다 (`Claim.java`)

```java
public boolean assign(Adjuster a) { return state.assign(this, a); }
public boolean pay()              { return state.pay(this); }
public void moveNext()            { moveTo(workflow.next(state)); }
```

if/switch가 없다. 새 상태가 생겨도 이 코드는 바뀌지 않는다.

### 1-3. `ReceivedState`: 접수 상태는 `assign`만 허용 (`ReceivedState.java`)

```java
public class ReceivedState implements ClaimState {
    @Override public String name() { return "접수"; }
    @Override public String denyReason() { return "조사 전"; }

    @Override
    public boolean assign(Claim c, Adjuster a) {
        c.setAdjuster(a);
        c.moveNext();   // 다음이 무엇인지는 이 클래스가 모른다
        return true;
    }
}
```

`pay()`를 부르면 구현이 없으니 기본 거절이 실행되어 `pay() → 거절(조사 전)`이 출력된다.

### 1-4. `ClaimWorkflow`: 순서를 한 곳에서 관리 (`ClaimWorkflow.java`)

```java
public static ClaimWorkflow standard() {
    ClaimWorkflow w = new ClaimWorkflow();
    w.order.add(new ReceivedState());
    w.order.add(new AssignedState());
    w.order.add(new InvestigatingState());
    w.order.add(new PendingCalcState());
    w.order.add(new PaymentApprovedState());
    w.order.add(new ClosedState());
    return w;
}

public ClaimWorkflow insertAfter(Class<? extends ClaimState> anchor, ClaimState added) { ... }
```

상태 클래스는 "다음이 무엇인지"를 모르고, 순서는 이 목록이 안다.

### 1-5. `PendingCalcState`: 점검 결과를 보고 전이를 직접 결정 (`PendingCalcState.java`)

```java
public boolean inspectFraud(Claim c, FraudInspector inspector) {
    FraudInspector.Report report = inspector.inspect(c);
    if (report.suspicious()) {
        c.moveTo(new FraudHoldState());   // 정상 경로 밖이라 직접 지정
    } else {
        c.moveNext();                     // 산정대기 → 지급승인
    }
    return true;
}
```

---

## 2. Strategy: 배정 기준을 갈아 끼운다

### 2-1. 인터페이스와 구현체 (`AssignmentStrategy.java`, `DistanceFirstStrategy.java`)

```java
public interface AssignmentStrategy {
    String name();
    Optional<Adjuster> select(Claim claim, List<Adjuster> candidates);
}

public class DistanceFirstStrategy implements AssignmentStrategy {
    @Override public String name() { return "거리 우선"; }

    @Override
    public Optional<Adjuster> select(Claim claim, List<Adjuster> candidates) {
        return candidates.stream()
                .min(Comparator.comparingDouble(a -> a.location().distanceTo(claim.location())));
    }
}
```

구현체는 `DistanceFirstStrategy`(거리 우선), `SpecialtyFirstStrategy`(전문분야 우선), `WorkloadBalancedStrategy`(업무량 균등)와 변경 시나리오의 `AiBasedStrategy`(AI 판독 기반)가 있다.

### 2-2. 쓰는 쪽은 `AssignCommand`: 기준이 무엇인지 모른다 (`AssignCommand.java`)

```java
Adjuster chosen = strategy.select(claim, candidates).orElse(null);
...
return claim.assign(chosen);   // 가능 여부는 State가 판단
```

선택 기준(Strategy)과 허용 여부(State)가 서로 섞이지 않는다.

---

## 3. Command: 동작을 객체로 만든다

### 3-1. `FieldCommand` (`FieldCommand.java`)

```java
public interface FieldCommand {
    boolean execute();    // 거절되면 false → 이력에 안 남음
    void undo();          // 되돌리기
    String describe();    // 이력 한 줄
}
```

구현체는 `AssignCommand`(배정), `ReassignCommand`(재배정), `RegisterResultCommand`(조사 결과 등록)다.

### 3-2. `AssignCommand`: undo를 위해 실행 직전 값을 저장 (`AssignCommand.java`)

```java
prevState = claim.state();
prevAdjuster = claim.adjuster();
return claim.assign(chosen);

public void undo() { claim.restore(prevState, prevAdjuster); }
```

### 3-3. `CommandInvoker`: 성공한 명령만 이력과 스택에 쌓는다 (`CommandInvoker.java`)

```java
public boolean run(FieldCommand command) {
    boolean ok = command.execute();
    if (ok) {
        executed.push(command);                    // 취소용 스택
        audit.add("실행: " + command.describe());  // 이력
    }
    return ok;
}

public boolean undoLast() {
    FieldCommand command = executed.pop();
    command.undo();
    audit.add("취소: " + command.describe());
    return true;
}
```

### 3-4. `FieldTerminal`: 통신이 끊기면 큐에 보관, 복구되면 순서대로 전송 (`FieldTerminal.java`)

```java
public void submit(FieldCommand command) {
    if (online) server.run(command);
    else pending.add(command);               // FIFO 큐에 보관
}

public void setOnline(boolean online) {
    this.online = online;
    if (!online) return;
    while (!pending.isEmpty()) {
        server.run(pending.poll());          // 먼저 보관한 것부터
    }
}
```

---

## 4. Visitor: 요소는 고정, 규칙만 추가

### 4-1. 요소(Element)와 방문자(Visitor) (`ClaimElement.java`, `ClaimVisitor.java`, `PartnerEstimate.java`)

```java
public interface ClaimElement {
    void accept(ClaimVisitor visitor);
}

public interface ClaimVisitor {
    void visit(AccidentInfo element);
    void visit(Party element);
    void visit(ClaimItem element);
    void visit(PartnerEstimate element);
}

public record PartnerEstimate(String partner, long amount, long referenceAmount, int repeatDeals)
        implements ClaimElement {
    @Override
    public void accept(ClaimVisitor visitor) { visitor.visit(this); }   // 더블 디스패치
}
```

### 4-2. `FraudRule`: 관심 있는 요소만 override (`FraudRule.java`)

```java
public abstract class FraudRule implements ClaimVisitor {
    public abstract String title();
    public abstract int maxPoints();

    protected void flag(String detail) {
        findings.add(new Finding(title(), maxPoints(), detail));
    }
    public List<Finding> drain() { /* 기록을 꺼내고 비운다 */ }

    @Override public void visit(AccidentInfo element) {}   // 관심 없음
    @Override public void visit(Party element) {}
    @Override public void visit(ClaimItem element) {}
    @Override public void visit(PartnerEstimate element) {}
}
```

### 4-3. 규칙 하나 = 클래스 하나 (`RepeatedClaimRule.java`)

```java
public class RepeatedClaimRule extends FraudRule {
    @Override public String title() { return "단기 반복 청구"; }
    @Override public int maxPoints() { return 40; }

    @Override
    public void visit(Party element) {
        if (element.recentClaims() >= 3) flag("최근 6개월 청구 " + element.recentClaims() + "회");
    }
}
```

### 4-4. `FraudInspector`: 점수만 합산해서 돌려준다 (`FraudInspector.java`)

```java
for (FraudRule rule : rules) {
    for (ClaimElement element : claim.elements()) {
        element.accept(rule);                 // accept → visit
    }
    List<FraudRule.Finding> found = rule.drain();
    int points = found.stream().mapToInt(FraudRule.Finding::points).sum();
    results.add(new RuleResult(rule.title(), points, rule.maxPoints(), detail));
    total += points;
}
return new Report(results, total, threshold, total >= threshold);   // 75점 이상이면 suspicious
```

`FraudInspector`는 상태 클래스를 모르고 결과만 돌려준다. 전이는 1-5의 `PendingCalcState`가 결정한다.

---

## 5. 변경 시나리오: 기존 코드 수정 없이 꽂기 (`team5.ext`)

### 5-1. 영상조사대기 (State): 상태 클래스 하나와 조립 한 줄 (`VideoWaitState.java`)

```java
public class VideoWaitState implements ClaimState {
    @Override public String name() { return "영상조사대기"; }
    @Override public String denyReason() { return "영상조사 대기"; }
    @Override public boolean registerResult(Claim c, String result) { c.addResult(result); return true; }
    @Override public boolean proceed(Claim c) { c.moveNext(); return true; }
}

ClaimWorkflow extended = ClaimWorkflow.standard()
        .insertAfter(InvestigatingState.class, new VideoWaitState());   // 한 줄
```

### 5-2. AI 판독 기반 배정 (Strategy): 전략 구현체 하나 (`AiBasedStrategy.java`)

```java
new AssignCommand(c, POOL, new AiBasedStrategy())
```

AI 등급이 `SEVERE`이면 중대손상 전문가 중 가장 가까운 사람, 아니면 진행 건수가 가장 적은 사람을 고른다.

### 5-3. 정비공장 반복 거래 규칙 (Visitor): 규칙 클래스 하나와 등록 한 줄 (`PartnerRepeatRule.java`)

```java
public class PartnerRepeatRule extends FraudRule {
    @Override public String title() { return "특정 제휴 정비공장 반복 거래"; }
    @Override public int maxPoints() { return 30; }

    @Override
    public void visit(PartnerEstimate element) {
        if (element.repeatDeals() >= 5) flag(element.partner() + " 반복 거래 " + element.repeatDeals() + "회");
    }
}

FraudInspector inspector = baseInspector().add(new PartnerRepeatRule());   // 한 줄
```

---

## 6. 발표에서 짚을 포인트

| 패턴 | 짚을 말 |
|---|---|
| State | `Claim.pay()`는 `state.pay(this)` 한 줄이라서 상태가 늘어도 `Claim`은 안 바뀐다 |
| Strategy | 선택 기준이 인터페이스 뒤에 있어서 `AssignCommand`는 기준을 모른다 |
| Command | `run`은 성공한 명령만 이력과 스택에 쌓고, 단말은 단절 중에 큐에 쌓는다 |
| Visitor | `FraudInspector`는 `accept`만 호출하고 규칙 내용은 모른다. 규칙이 늘어도 요소 4종은 그대로다 |

## 7. 관련 문서

| 문서 | 내용 |
|---|---|
| `변경시나리오_상세분석.md` | 변경 시나리오 3종의 패턴과 구현 분석 |
| `클래스다이어그램_발표대본.md` | 클래스 다이어그램 장표 발표 대본 |
| `시퀀스_상태다이어그램.md` | State 다이어그램, 배정·현장 조사 시퀀스, 사기 점검 시퀀스 |
