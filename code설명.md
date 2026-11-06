# 패턴 상세 흐름

> 비유나 일반론이 아니라 **`src/team5`에 실제로 있는 클래스·메서드·실행 로그**만으로 설명한다.
> 각 패턴마다 같은 순서로 정리한다.
> ① 무엇이 바뀌는가 / 무엇을 보호하는가 → ② 코드 속 역할 (②-2 전체 코드는 접어 둠) → ③ 호출이 실제로 흐르는 순서 → ④ 변경 시나리오에서 추가된 코드 vs 건드리지 않은 코드 → ⑤ 코드를 읽다 헷갈리기 쉬운 지점 → ⑥ 이 패턴을 쓰지 않았다면 (가정 코드와 비교)
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

### ②-2 전체 코드

State 패턴의 클래스 전체이다. 아래 순서로 읽으면 "인터페이스 → 위임하는 쪽 → 순서 관리 → 구체 상태" 순으로 이어진다. 파일을 접어 두었으니 펼쳐서 본다. 주석의 [패턴: …] 표시가 패턴 안에서의 역할이다.

<details><summary><b>1. ClaimState.java</b> (49줄) — 기본 구현이 전부 거절(deny)이다</summary>

```java
package team5;

/**
 * [패턴: State] 사고 건의 진행 단계(상태)를 나타내는 인터페이스.
 *
 * 핵심 아이디어:
 *   - Claim 은 업무 요청(assign, pay 등)을 직접 처리하지 않고 현재 상태 객체에 위임한다.
 *   - 기본 구현(default)은 전부 "거절"이다. 각 상태 클래스는 자기 단계에서 허용되는 작업만 override 한다.
 *   - 그래서 "이 상태에서 이 작업이 가능한가?"를 if/switch 없이 다형성으로 판단한다.
 *
 * 반환값 규칙: 작업이 수행되면 true, 거절되면 false.
 */
public interface ClaimState {
    /** 화면에 표시할 상태 이름 (예: "접수", "조사중") */
    String name();

    /** 거절 메시지에 붙는 사유 (예: 조사 전, 보류 중) */
    String denyReason();

    /** 담당자 배정. 기본은 거절. */
    default boolean assign(Claim c, Adjuster a) { return deny("assign"); }

    /** 담당자 재배정. 기본은 거절. */
    default boolean reassign(Claim c, Adjuster a) { return deny("reassign"); }

    /** 조사 결과 등록. 기본은 거절. */
    default boolean registerResult(Claim c, String result) { return deny("registerResult"); }

    /** 다음 단계로 진행. 기본은 거절. */
    default boolean proceed(Claim c) { return deny("proceed"); }

    /** 사기 점검 수행. 기본은 거절. */
    default boolean inspectFraud(Claim c, FraudInspector inspector) { return deny("inspectFraud"); }

    /** 보험금 지급. 기본은 거절. */
    default boolean pay(Claim c) { return deny("pay"); }

    /**
     * 공통 거절 처리. 어떤 작업이 왜 거절됐는지 출력하고 false 를 돌려준다.
     *
     * @param operation 거절된 작업 이름 (예: "pay")
     * @return 항상 false
     */
    default boolean deny(String operation) {
        // 예: "  pay() → 거절(조사 전)" 형태로 출력. denyReason() 은 각 상태가 구현한다.
        System.out.println("  " + operation + "() → 거절(" + denyReason() + ")");
        return false;
    }
}
```

</details>

<details><summary><b>2. Claim.java</b> (123줄) — 업무 요청을 state 에 넘기기만 한다. if/switch 가 없다</summary>

```java
package team5;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * 사고 건. [패턴: State (Context) + Visitor (Element 보유)]
 * 업무 요청은 모두 현재 State 객체에 위임하므로 if/switch 로 상태를 분기하지 않는다.
 * 구성 요소(ClaimElement)는 고정이고, 점검 규칙은 Visitor 로 외부에서 추가된다.
 */
public class Claim {
    /** 사고 번호 (예: AC-3001) */
    private final String id;
    /** 사고 발생 위치 (배정 전략의 거리 계산에 사용) */
    private final Location location;
    /** 차량 유형 (예: 일반, 대형화물). 전문분야 우선 배정에 사용 */
    private final String vehicleType;
    /** 이 사고가 따라갈 상태 진행 순서 */
    private final ClaimWorkflow workflow;
    /** 등록된 조사 결과들 (누적) */
    private final List<String> results = new ArrayList<>();
    /** 사고 구성 요소들 (사고정보, 당사자, 청구항목, 견적) */
    private final List<ClaimElement> elements = new ArrayList<>();
    /** 현재 상태. 모든 업무 요청이 이 객체에게 위임된다. */
    private ClaimState state;
    /** 현재 담당 손해사정사 (배정 전에는 null) */
    private Adjuster adjuster;
    /** AI 1차 판독 손상 등급 (기본 "NONE", 중대 손상은 "SEVERE") */
    private String aiDamageGrade = "NONE";

    /**
     * 사고 건을 만든다. 초기 상태는 워크플로의 첫 상태(접수)이다.
     *
     * @param id          사고 번호
     * @param location    사고 위치
     * @param vehicleType 차량 유형
     * @param workflow    사용할 워크플로 (표준 또는 확장)
     */
    public Claim(String id, Location location, String vehicleType, ClaimWorkflow workflow) {
        this.id = id;
        this.location = location;
        this.vehicleType = vehicleType;
        this.workflow = workflow;
        this.state = workflow.first(); // 시작 상태 = 접수
    }

    // ---- 업무 요청: 현재 상태에 위임 ----
    // 허용 여부와 실제 동작은 현재 state 구현체가 정한다. 여기서는 전달만 한다.

    /** 담당자 배정 요청 */
    public boolean assign(Adjuster a) { return state.assign(this, a); }

    /** 담당자 재배정 요청 */
    public boolean reassign(Adjuster a) { return state.reassign(this, a); }

    /** 조사 결과 등록 요청 */
    public boolean registerResult(String r) { return state.registerResult(this, r); }

    /** 다음 단계 진행 요청 */
    public boolean proceed() { return state.proceed(this); }

    /** 사기 점검 요청 */
    public boolean inspectFraud(FraudInspector i) { return state.inspectFraud(this, i); }

    /** 지급 요청 */
    public boolean pay() { return state.pay(this); }

    // ---- 상태 객체가 사용하는 전이·변경 메서드 ----

    /** 워크플로가 정한 다음 상태로 이동한다. */
    public void moveNext() { moveTo(workflow.next(state)); }

    /**
     * 지정한 상태로 직접 이동한다. 정상 경로 밖의 상태(사기조사보류)로 갈 때도 사용된다.
     *
     * @param next 이동할 상태
     */
    public void moveTo(ClaimState next) {
        System.out.println("  상태 전이: " + state.name() + " → " + next.name());
        this.state = next;
    }

    /** 담당자를 설정한다. (상태 객체가 호출) */
    public void setAdjuster(Adjuster a) { this.adjuster = a; }

    /** 조사 결과를 목록에 추가한다. (상태 객체가 호출) */
    public void addResult(String r) { results.add(r); }

    /**
     * Command.undo 용: 직전 상태와 담당자를 되돌린다.
     * 전이 메시지를 출력하지 않도록 moveTo 를 거치지 않고 필드를 직접 바꾼다.
     */
    public void restore(ClaimState prevState, Adjuster prevAdjuster) {
        this.state = prevState;
        this.adjuster = prevAdjuster;
    }

    /** 가장 최근 조사 결과 한 건을 제거한다. 비어 있으면 아무 일도 하지 않는다. */
    public void removeLastResult() {
        if (!results.isEmpty()) results.remove(results.size() - 1);
    }

    /** 사고 구성 요소를 추가한다. 사기 점검 때 방문 대상이 된다. */
    public void addElement(ClaimElement e) { elements.add(e); }

    /** AI 판독 손상 등급을 설정한다. (AiBasedStrategy 가 참조) */
    public void setAiDamageGrade(String grade) { this.aiDamageGrade = grade; }

    // ---- 조회용 접근자 ----
    public String id() { return id; }
    public Location location() { return location; }
    public String vehicleType() { return vehicleType; }
    public String aiDamageGrade() { return aiDamageGrade; }
    public ClaimState state() { return state; }
    public Adjuster adjuster() { return adjuster; }

    /** 조사 결과 목록 (수정 불가 뷰). 외부에서 직접 변경하지 못하게 막는다. */
    public List<String> results() { return Collections.unmodifiableList(results); }

    /** 구성 요소 목록 (수정 불가 뷰) */
    public List<ClaimElement> elements() { return Collections.unmodifiableList(elements); }
}
```

</details>

<details><summary><b>3. ClaimWorkflow.java</b> (70줄) — "다음 상태"를 한 곳에서 관리하고 insertAfter 로 상태를 끼운다</summary>

```java
package team5;

import java.util.ArrayList;
import java.util.List;

/**
 * 정상 처리 경로(상태 순서)를 한 곳에서 관리한다.
 * 상태 클래스는 "다음이 무엇인지" 모르므로, 새 상태는 insertAfter 한 줄로 끼워 넣는다.
 * (상태를 추가해도 기존 상태 클래스를 수정하지 않아도 되는 이유)
 *
 * 주의: 사기조사보류(FraudHoldState)는 정상 경로 밖이라 이 목록에 없고 moveTo() 로 직접 이동한다.
 */
public class ClaimWorkflow {
    /** 상태 진행 순서. 앞에서부터 차례로 진행한다. */
    private final List<ClaimState> order = new ArrayList<>();

    /**
     * 표준 워크플로를 만든다.
     * 접수 → 배정 → 조사중 → 산정대기 → 지급승인 → 종결
     *
     * @return 표준 순서로 구성된 새 워크플로 (호출할 때마다 새 객체)
     */
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

    /**
     * 기준 상태 바로 뒤에 새 상태를 끼워 넣는다. (변경 시나리오: 영상조사대기 추가)
     *
     * @param anchor 이 클래스의 상태 바로 뒤에 삽입한다
     * @param added  삽입할 새 상태
     * @return 자기 자신 (메서드 체이닝용)
     * @throws IllegalArgumentException 기준 상태가 워크플로에 없을 때
     */
    public ClaimWorkflow insertAfter(Class<? extends ClaimState> anchor, ClaimState added) {
        for (int i = 0; i < order.size(); i++) {
            // 객체 비교가 아니라 "클래스 종류"로 기준 위치를 찾는다.
            if (order.get(i).getClass() == anchor) {
                order.add(i + 1, added); // 기준 바로 뒤 위치에 삽입
                return this;
            }
        }
        throw new IllegalArgumentException("기준 상태 없음: " + anchor.getSimpleName());
    }

    /** 시작 상태(접수)를 돌려준다. Claim 생성자가 초기 상태로 사용한다. */
    public ClaimState first() { return order.get(0); }

    /**
     * 현재 상태 다음에 올 상태를 돌려준다.
     *
     * @param current 현재 상태
     * @return 순서상 다음 상태
     * @throws IllegalStateException 마지막 상태이거나 경로에 없는 상태일 때
     */
    public ClaimState next(ClaimState current) {
        // 마지막 원소는 다음이 없으므로 size() - 1 까지만 탐색한다.
        for (int i = 0; i < order.size() - 1; i++) {
            if (order.get(i).getClass() == current.getClass()) return order.get(i + 1);
        }
        throw new IllegalStateException("다음 상태 없음: " + current.name());
    }
}
```

</details>

<details><summary><b>4. ReceivedState.java</b> (28줄) — 배정(assign)만 허용한다</summary>

```java
package team5;

/**
 * [State] 접수: 사고가 막 접수된 첫 단계.
 * 이 단계에서 허용되는 작업은 "담당자 배정(assign)" 하나뿐이다.
 * 조사 결과 등록, 사기 점검, 지급 등은 ClaimState 의 기본 구현에 따라 모두 거절된다.
 */
public class ReceivedState implements ClaimState {
    /** 상태 이름 */
    @Override public String name() { return "접수"; }

    /** 거절 사유: 아직 조사가 시작되기 전이다. */
    @Override public String denyReason() { return "조사 전"; }

    /**
     * 담당자를 지정하고 다음 단계(배정)로 넘어간다.
     *
     * @param c 대상 사고 건
     * @param a 배정할 손해사정사
     * @return 항상 true (접수 상태에서 배정은 항상 허용)
     */
    @Override
    public boolean assign(Claim c, Adjuster a) {
        c.setAdjuster(a); // 담당자 기록
        c.moveNext();     // 워크플로가 정한 다음 상태(배정)로 전이. 다음이 무엇인지는 이 클래스가 모른다.
        return true;
    }
}
```

</details>

<details><summary><b>5. InvestigatingState.java</b> (52줄) — 재배정·결과 등록·진행을 허용한다</summary>

```java
package team5;

/**
 * [State] 조사중: 담당자가 현장에서 조사하는 단계.
 * 허용 작업: 재배정(reassign), 조사 결과 등록(registerResult), 조사 완료(proceed).
 * 사기 점검과 지급은 조사가 끝난 뒤(산정대기 이후)에만 가능하므로 거절된다.
 */
public class InvestigatingState implements ClaimState {
    /** 상태 이름 */
    @Override public String name() { return "조사중"; }

    /** 거절 사유: 조사가 진행 중이다. */
    @Override public String denyReason() { return "조사 중"; }

    /**
     * 조사 중에도 담당자를 바꿀 수 있다. 상태는 유지된다.
     *
     * @param c 대상 사고 건
     * @param a 새 담당자
     * @return 항상 true
     */
    @Override
    public boolean reassign(Claim c, Adjuster a) {
        c.setAdjuster(a);
        return true;
    }

    /**
     * 현장 조사 결과를 기록한다. 여러 번 호출하면 계속 누적된다.
     *
     * @param c      대상 사고 건
     * @param result 조사 결과 문구 (예: "전면부 파손 확인")
     * @return 항상 true
     */
    @Override
    public boolean registerResult(Claim c, String result) {
        c.addResult(result); // Claim 의 결과 목록에 추가
        return true;
    }

    /**
     * 현장 조사 완료.
     *
     * @param c 대상 사고 건
     * @return 항상 true
     */
    @Override
    public boolean proceed(Claim c) {
        c.moveNext(); // 조사중 → (표준: 산정대기 / 확장: 영상조사대기)
        return true;
    }
}
```

</details>

<details><summary><b>6. PendingCalcState.java</b> (49줄) — 점검 결과로 moveNext / moveTo 를 고른다</summary>

```java
package team5;

/**
 * [State] 산정대기: 지급승인 전에 사기 점검을 수행하는 단계.
 * 허용 작업: 사기 점검(inspectFraud) 하나뿐이다. 점검을 통과해야만 지급승인으로 넘어간다.
 *
 * FraudInspector 는 점검 결과(Report)만 돌려주고, 상태 전이는 이 클래스가 직접 결정한다.
 * (점검 로직과 상태 전이 책임을 분리한 설계)
 */
public class PendingCalcState implements ClaimState {
    /** 상태 이름 */
    @Override public String name() { return "산정대기"; }

    /** 거절 사유: 사기 점검이 끝나기 전에는 지급할 수 없다. */
    @Override public String denyReason() { return "사기 점검 전"; }

    /**
     * 사기 점검을 수행하고 결과에 따라 다음 상태를 결정한다.
     *   - 합계가 기준 이상  → 사기조사보류(FraudHoldState, 정상 경로 밖)로 이동
     *   - 기준 미만         → 워크플로상 다음 상태(지급승인)로 이동
     *
     * @param c         대상 사고 건
     * @param inspector 규칙이 등록된 사기 점검기
     * @return 항상 true (점검 자체는 수행됨)
     */
    @Override
    public boolean inspectFraud(Claim c, FraudInspector inspector) {
        // 점검기에 사고 건을 넘겨 모든 규칙을 적용한 결과 리포트를 받는다.
        FraudInspector.Report report = inspector.inspect(c);

        System.out.println("  [사기 점검]");
        // 규칙별 결과(걸리지 않은 항목은 0점)를 한 줄씩 출력한다.
        for (FraudInspector.RuleResult r : report.results()) {
            String note = (r.detail() == null) ? "해당 없음" : r.detail();
            System.out.println("    - " + r.title() + ": " + r.points() + "점 / " + r.maxPoints() + "점  (" + note + ")");
        }
        System.out.println("    합계 " + report.total() + "점 (기준 " + report.threshold() + "점 이상이면 보류)");

        if (report.suspicious()) {
            System.out.println("  → 사기조사보류");
            // 정상 경로에 없는 상태이므로 moveNext() 가 아니라 moveTo() 로 직접 지정한다.
            c.moveTo(new FraudHoldState());
        } else {
            System.out.println("  → 이상 없음, 지급승인");
            c.moveNext(); // 산정대기 → 지급승인
        }
        return true;
    }
}
```

</details>

<details><summary><b>7. FraudHoldState.java</b> (15줄) — 아무것도 override 하지 않는다 = 전부 거절</summary>

```java
package team5;

/**
 * [State] 사기조사보류: 정상 경로 밖의 종착 상태.
 * 사기 점검 점수가 기준 이상일 때 PendingCalcState 가 moveTo() 로 직접 보낸다.
 * 어떤 작업도 허용하지 않으므로 지급은 거절되고, 이후 처리는 별도 프로세스(사기 조사)로 넘어간다.
 * ClaimWorkflow 의 순서 목록에는 들어 있지 않다.
 */
public class FraudHoldState implements ClaimState {
    /** 상태 이름 */
    @Override public String name() { return "사기조사보류"; }

    /** 거절 사유: 사기 의심으로 보류 중이다. */
    @Override public String denyReason() { return "보류 중"; }
}
```

</details>

<details><summary><b>8. VideoWaitState.java</b> (42줄) — 변경 시나리오 ①. 이 파일 하나가 추가된 코드의 전부이다</summary>

```java
package team5.ext;

import team5.Claim;
import team5.ClaimState;

/**
 * [변경 시나리오 1] 영상조사대기 상태.
 * 기존 클래스 수정 없이 ClaimWorkflow.insertAfter() 로 워크플로에 끼워 넣는다. (개방-폐쇄 원칙)
 * 허용 작업: 조사 결과 등록, 다음 단계 진행. 지급 등은 기본 구현에 따라 거절된다.
 */
public class VideoWaitState implements ClaimState {
    /** 상태 이름 */
    @Override public String name() { return "영상조사대기"; }

    /** 거절 사유: 영상 조사 결과를 기다리는 중이다. */
    @Override public String denyReason() { return "영상조사 대기"; }

    /**
     * 영상 분석 결과 등을 조사 결과로 기록한다.
     *
     * @param c      대상 사고 건
     * @param result 결과 문구
     * @return 항상 true
     */
    @Override
    public boolean registerResult(Claim c, String result) {
        c.addResult(result);
        return true;
    }

    /**
     * 영상 조사를 마치고 다음 단계로 넘어간다.
     *
     * @param c 대상 사고 건
     * @return 항상 true
     */
    @Override
    public boolean proceed(Claim c) {
        c.moveNext(); // 워크플로상 이 상태 다음(산정대기)으로 전이
        return true;
    }
}
```

</details>

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

### ②-2 전체 코드

Strategy 패턴의 클래스 전체이다. `AssignCommand`/`ReassignCommand`가 이 인터페이스를 어떻게 쓰는지는 3절(Command)의 전체 코드에 있다. 파일을 접어 두었으니 펼쳐서 본다. 주석의 [패턴: …] 표시가 패턴 안에서의 역할이다.

<details><summary><b>1. AssignmentStrategy.java</b> (43줄) — select 가 핵심이고, rule/evaluate/explain 은 로그용 기본 구현이다</summary>

```java
package team5;

import java.util.List;
import java.util.Optional;

/**
 * [패턴: Strategy] 손해사정사 배정 방식을 추상화한 인터페이스.
 * 배정/재배정 Command(AssignCommand, ReassignCommand)가 이 인터페이스에만 의존하므로,
 * 배정 정책(거리·전문분야·업무량·AI 등)을 코드 변경 없이 갈아끼울 수 있다.
 */
public interface AssignmentStrategy {
    /** 전략 이름 (로그·이력 출력용, 예: "거리 우선") */
    String name();

    /**
     * 후보 중에서 이 사고 건을 맡을 손해사정사를 고른다.
     *
     * @param claim      배정 대상 사고 건 (위치, 차량 유형 등을 참고)
     * @param candidates 후보 손해사정사 목록
     * @return 선택된 사람. 후보가 없으면 Optional.empty()
     */
    Optional<Adjuster> select(Claim claim, List<Adjuster> candidates);

    /** 배정 기준을 한 줄로 설명한다. (출력용) */
    default String rule() { return name(); }

    /**
     * 후보 한 명의 비교 값을 문자열로 돌려준다. (출력용, 예: "거리 1.4")
     * 배정 로직이 후보를 어떤 기준으로 비교했는지 눈으로 확인하는 데 쓴다.
     */
    default String evaluate(Claim claim, Adjuster candidate) { return ""; }

    /**
     * 이 전략이 어떤 기준으로 후보를 비교하는지, 후보별 비교 값을 출력한다.
     * 배정 Command 가 실제로 고르기 직전에 호출한다.
     */
    default void explain(Claim claim, List<Adjuster> candidates) {
        System.out.println("  [배정 로직] " + name() + ": " + rule());
        for (Adjuster a : candidates) {
            System.out.println("    후보 " + a.name() + " : " + evaluate(claim, a));
        }
    }
}
```

</details>

<details><summary><b>2. DistanceFirstStrategy.java</b> (35줄) — 거리가 가장 짧은 후보</summary>

```java
package team5;

import java.util.Comparator;
import java.util.List;
import java.util.Optional;

/**
 * [Strategy 구현] 거리 우선: 사고 현장에서 가장 가까운 손해사정사를 고른다.
 * 현장 도착 시간을 줄이는 것이 목적인 정책이다.
 */
public class DistanceFirstStrategy implements AssignmentStrategy {
    /** 전략 이름 */
    @Override public String name() { return "거리 우선"; }

    @Override public String rule() { return "사고 현장과 가장 가까운 사람"; }

    @Override
    public String evaluate(Claim claim, Adjuster a) {
        return String.format("거리 %.1f", a.location().distanceTo(claim.location()));
    }

    /**
     * 사고 위치까지의 거리가 가장 짧은 후보를 반환한다.
     *
     * @param claim      사고 건 (위치 사용)
     * @param candidates 후보 목록
     * @return 가장 가까운 사람, 후보가 비어 있으면 empty
     */
    @Override
    public Optional<Adjuster> select(Claim claim, List<Adjuster> candidates) {
        return candidates.stream()
                // 각 후보의 위치 ~ 사고 위치 거리를 기준으로 최솟값을 찾는다.
                .min(Comparator.comparingDouble(a -> a.location().distanceTo(claim.location())));
    }
}
```

</details>

<details><summary><b>3. SpecialtyFirstStrategy.java</b> (46줄) — 전문가 중 가장 가까운 사람, 없으면 전체 중 가장 가까운 사람</summary>

```java
package team5;

import java.util.Comparator;
import java.util.List;
import java.util.Optional;

/**
 * [Strategy 구현] 전문분야 우선.
 * 차량 유형(대형화물·특수차량 등)에 전문성이 있는 사정사를 우선 배정한다.
 * 전문가가 없으면 가장 가까운 사람을 고른다.
 */
public class SpecialtyFirstStrategy implements AssignmentStrategy {
    /** 전략 이름 */
    @Override public String name() { return "전문분야 우선"; }

    @Override public String rule() { return "차량 종류 전문가 중 가장 가까운 사람 (전문가가 없으면 전체 중 가장 가까운 사람)"; }

    @Override
    public String evaluate(Claim claim, Adjuster a) {
        return String.format("%s 전문 %s, 거리 %.1f", claim.vehicleType(),
                a.hasSpecialty(claim.vehicleType()) ? "예" : "아니오",
                a.location().distanceTo(claim.location()));
    }

    /**
     * 1순위: 사고 차량 유형 전문가 중 가장 가까운 사람. 2순위(전문가 없음): 전체 중 가장 가까운 사람.
     *
     * @param claim      사고 건 (차량 유형과 위치 사용)
     * @param candidates 후보 목록
     * @return 선택된 사람, 후보가 비어 있으면 empty
     */
    @Override
    public Optional<Adjuster> select(Claim claim, List<Adjuster> candidates) {
        // 거리 비교기를 한 번만 만들어 두 번 재사용한다.
        Comparator<Adjuster> byDistance =
                Comparator.comparingDouble(a -> a.location().distanceTo(claim.location()));

        // 차량 유형을 전문 분야로 가진 후보만 걸러서 그중 가장 가까운 사람을 찾는다.
        Optional<Adjuster> specialist = candidates.stream()
                .filter(a -> a.hasSpecialty(claim.vehicleType()))
                .min(byDistance);

        // 전문가가 있으면 그 사람, 없으면 전체 후보 중 가장 가까운 사람(대체 규칙)
        return specialist.isPresent() ? specialist : candidates.stream().min(byDistance);
    }
}
```

</details>

<details><summary><b>4. WorkloadBalancedStrategy.java</b> (33줄) — 진행 건수가 가장 적은 후보</summary>

```java
package team5;

import java.util.Comparator;
import java.util.List;
import java.util.Optional;

/**
 * [Strategy 구현] 업무량 균등: 진행 중인 건수가 가장 적은 손해사정사를 고른다.
 * 특정 사람에게 일이 몰리지 않도록 분산하는 정책이다. 사고 위치·차량 유형은 보지 않는다.
 */
public class WorkloadBalancedStrategy implements AssignmentStrategy {
    /** 전략 이름 */
    @Override public String name() { return "업무량 균등"; }

    @Override public String rule() { return "진행 중인 건수가 가장 적은 사람"; }

    @Override
    public String evaluate(Claim claim, Adjuster a) {
        return "진행 " + a.activeCases() + "건";
    }

    /**
     * activeCases(진행 중 건수)가 가장 적은 후보를 반환한다.
     *
     * @param claim      사고 건 (이 전략에서는 사용하지 않음)
     * @param candidates 후보 목록
     * @return 가장 여유 있는 사람, 후보가 비어 있으면 empty
     */
    @Override
    public Optional<Adjuster> select(Claim claim, List<Adjuster> candidates) {
        return candidates.stream().min(Comparator.comparingInt(Adjuster::activeCases));
    }
}
```

</details>

<details><summary><b>5. AiBasedStrategy.java</b> (51줄) — 변경 시나리오 ②. 이 파일 하나가 추가된 코드의 전부이다</summary>

```java
package team5.ext;

import java.util.Comparator;
import java.util.List;
import java.util.Optional;
import team5.Adjuster;
import team5.AssignmentStrategy;
import team5.Claim;

/**
 * [변경 시나리오 2] AI 1차 판독 결과 기반 배정.
 * 중대 손상이면 중대손상 전문가를, 아니면 업무량 균등으로 배정한다.
 * AssignmentStrategy 를 구현만 하면 되므로 기존 Command 나 Claim 코드는 수정할 필요가 없다.
 */
public class AiBasedStrategy implements AssignmentStrategy {
    /** 전략 이름 */
    @Override public String name() { return "AI 판독 기반"; }

    @Override
    public String rule() {
        return "AI 판독이 중대 손상이면 중대손상 전문가 중 가장 가까운 사람, 아니면 진행 건수가 가장 적은 사람";
    }

    @Override
    public String evaluate(Claim claim, Adjuster a) {
        return String.format("중대손상 전문 %s, 거리 %.1f, 진행 %d건",
                a.hasSpecialty("중대손상") ? "예" : "아니오",
                a.location().distanceTo(claim.location()), a.activeCases());
    }

    /**
     * 1순위: AI 등급이 SEVERE 이면 "중대손상" 전문가 중 가장 가까운 사람.
     * 2순위: 전문가가 없거나 중대 손상이 아니면 진행 건수가 가장 적은 사람.
     *
     * @param claim      사고 건 (AI 손상 등급과 위치 사용)
     * @param candidates 후보 목록
     * @return 선택된 사람, 후보가 비어 있으면 empty
     */
    @Override
    public Optional<Adjuster> select(Claim claim, List<Adjuster> candidates) {
        // 문자열 비교는 상수를 앞에 두어 grade 가 null 이어도 NPE 가 나지 않게 한다.
        if ("SEVERE".equals(claim.aiDamageGrade())) {
            Optional<Adjuster> expert = candidates.stream()
                    .filter(a -> a.hasSpecialty("중대손상"))
                    .min(Comparator.comparingDouble(a -> a.location().distanceTo(claim.location())));
            if (expert.isPresent()) return expert;
        }
        // 중대 손상이 아니거나 전문가가 없을 때의 대체 규칙
        return candidates.stream().min(Comparator.comparingInt(Adjuster::activeCases));
    }
}
```

</details>

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

### ②-2 전체 코드

Command 패턴의 클래스 전체이다. Receiver 는 `Claim`(1절), 배정 방식은 `AssignmentStrategy`(2절)에 있다. 파일을 접어 두었으니 펼쳐서 본다. 주석의 [패턴: …] 표시가 패턴 안에서의 역할이다.

<details><summary><b>1. FieldCommand.java</b> (22줄) — execute / undo / describe 세 개뿐이다</summary>

```java
package team5;

/**
 * [패턴: Command] 현장에서 내리는 지시를 객체로 만든 인터페이스.
 * 요청을 객체로 캡슐화하면 다음이 가능해진다.
 *   - 통신 단절 시 단말에 보관했다가 나중에 실행 (FieldTerminal)
 *   - 서버가 실행 이력을 기록하고 마지막 명령을 취소 (CommandInvoker)
 */
public interface FieldCommand {
    /**
     * 지시를 실행한다.
     *
     * @return 실행 성공 여부 (거절되면 false, 이력에 남지 않는다)
     */
    boolean execute();

    /** 직전 execute() 의 효과를 되돌린다. */
    void undo();

    /** 이력에 남길 한 줄 설명 (예: "배정(AC-3001, 거리 우선)") */
    String describe();
}
```

</details>

<details><summary><b>2. AssignCommand.java</b> (67줄) — Strategy 로 고르고 Claim 에 배정을 요청한다. undo 용으로 이전 값을 저장한다</summary>

```java
package team5;

import java.util.List;

/**
 * [Command 구현] 배정 지시.
 * Strategy 로 담당자를 고른 뒤 Claim 에 전달한다.
 * undo 를 위해 실행 직전의 상태와 담당자를 기억해 둔다.
 */
public class AssignCommand implements FieldCommand {
    /** 배정 대상 사고 건 (Receiver) */
    private final Claim claim;
    /** 담당자 후보 목록 */
    private final List<Adjuster> candidates;
    /** 담당자를 고르는 방식 (Strategy) */
    private final AssignmentStrategy strategy;
    /** undo 용: 실행 직전 상태 */
    private ClaimState prevState;
    /** undo 용: 실행 직전 담당자 */
    private Adjuster prevAdjuster;

    /**
     * @param claim       배정할 사고 건
     * @param candidates  후보 손해사정사 목록
     * @param strategy    배정 전략
     */
    public AssignCommand(Claim claim, List<Adjuster> candidates, AssignmentStrategy strategy) {
        this.claim = claim;
        this.candidates = candidates;
        this.strategy = strategy;
    }

    /**
     * 전략으로 담당자를 고르고 사고 건에 배정을 요청한다.
     *
     * @return 배정 성공 여부. 후보가 없거나 현재 상태가 배정을 허용하지 않으면 false
     */
    @Override
    public boolean execute() {
        // 어떤 기준으로 후보를 비교하는지 먼저 보여준다. (배정 로직)
        strategy.explain(claim, candidates);
        // 전략에게 선택을 위임한다. 후보가 없으면 null.
        Adjuster chosen = strategy.select(claim, candidates).orElse(null);
        if (chosen == null) {
            System.out.println("  배정 가능한 손해사정사 없음");
            return false;
        }
        // 상태가 바뀌기 전에 현재 값을 저장해 둔다 (undo 때 복원).
        prevState = claim.state();
        prevAdjuster = claim.adjuster();
        System.out.println("  배정: " + strategy.name() + " → " + chosen);
        // 실제 가능 여부는 현재 상태 객체가 판단한다. (접수 상태일 때만 true)
        return claim.assign(chosen);
    }

    /** 저장해 둔 이전 상태와 담당자로 되돌린다. */
    @Override
    public void undo() {
        claim.restore(prevState, prevAdjuster);
    }

    /** 이력 문구: "배정(사고ID, 전략명)" */
    @Override
    public String describe() {
        return "배정(" + claim.id() + ", " + strategy.name() + ")";
    }
}
```

</details>

<details><summary><b>3. ReassignCommand.java</b> (74줄) — 현재 담당자를 후보에서 빼고 요청자를 기록한다</summary>

```java
package team5;

import java.util.List;

/**
 * [Command 구현] 재배정 지시.
 * 현재 담당자를 제외한 후보 중에서 Strategy 로 다시 고른다.
 * undo 를 위해 실행 직전의 상태와 담당자를 기억해 둔다.
 */
public class ReassignCommand implements FieldCommand {
    /** 재배정 대상 사고 건 (Receiver) */
    private final Claim claim;
    /** 담당자 후보 목록 (현재 담당자 포함, 실행 시 제외함) */
    private final List<Adjuster> candidates;
    /** 담당자를 고르는 방식 (Strategy) */
    private final AssignmentStrategy strategy;
    /** undo 용: 실행 직전 상태 */
    private ClaimState prevState;
    /** undo 용: 실행 직전 담당자 */
    private Adjuster prevAdjuster;
    /** 재배정을 요청한 손해사정사 (실행 시점의 담당자). 이력에만 남긴다. */
    private Adjuster requester;

    /**
     * @param claim       재배정할 사고 건
     * @param candidates  후보 손해사정사 목록
     * @param strategy    재배정 전략
     */
    public ReassignCommand(Claim claim, List<Adjuster> candidates, AssignmentStrategy strategy) {
        this.claim = claim;
        this.candidates = candidates;
        this.strategy = strategy;
    }

    /**
     * 현재 담당자를 뺀 후보 중에서 새 담당자를 골라 재배정한다.
     *
     * @return 성공 여부. 후보가 없거나 현재 상태가 재배정을 허용하지 않으면 false
     */
    @Override
    public boolean execute() {
        // 같은 사람이 다시 뽑히지 않도록 현재 담당자를 후보에서 제외한다. (참조 비교)
        List<Adjuster> others = candidates.stream().filter(a -> a != claim.adjuster()).toList();
        // 재배정을 요청한 사람은 요청 시점의 담당자다. 이력에 남기려고 기억해 둔다. (거절 사유는 받지 않는다)
        requester = claim.adjuster();
        System.out.println("  재배정 요청자: " + requester);
        // 현재 담당자를 뺀 후보를 어떤 기준으로 비교하는지 보여준다. (배정 로직)
        strategy.explain(claim, others);
        Adjuster chosen = strategy.select(claim, others).orElse(null);
        if (chosen == null) {
            System.out.println("  재배정 가능한 손해사정사 없음");
            return false;
        }
        // undo 를 위해 변경 전 값을 저장한다.
        prevState = claim.state();
        prevAdjuster = claim.adjuster();
        System.out.println("  재배정: " + strategy.name() + " → " + chosen);
        // 배정/조사중 상태에서만 허용된다. (상태 객체가 판단)
        return claim.reassign(chosen);
    }

    /** 저장해 둔 이전 상태와 담당자로 되돌린다. */
    @Override
    public void undo() {
        claim.restore(prevState, prevAdjuster);
    }

    /** 이력 문구: "재배정(사고ID, 전략명, 요청자 K)". 요청자는 실행 전이면 현재 담당자를 보여준다. */
    @Override
    public String describe() {
        Adjuster who = (requester != null) ? requester : claim.adjuster();
        return "재배정(" + claim.id() + ", " + strategy.name() + ", 요청자 " + who + ")";
    }
}
```

</details>

<details><summary><b>4. RegisterResultCommand.java</b> (47줄) — 조사 결과 등록. undo 는 마지막 결과 제거</summary>

```java
package team5;

/**
 * [Command 구현] 조사 결과 등록 지시.
 * 현장 단말에서 입력한 조사 결과 문구를 사고 건에 기록한다.
 * 통신이 끊긴 구간에서는 단말에 보관되었다가 복구 후 서버로 전송된다.
 */
public class RegisterResultCommand implements FieldCommand {
    /** 결과를 등록할 사고 건 (Receiver) */
    private final Claim claim;
    /** 등록할 조사 결과 문구 */
    private final String result;

    /**
     * @param claim  대상 사고 건
     * @param result 조사 결과 문구
     */
    public RegisterResultCommand(Claim claim, String result) {
        this.claim = claim;
        this.result = result;
    }

    /**
     * 조사 결과를 등록한다.
     *
     * @return 등록 성공 여부. 조사중(또는 영상조사대기) 상태가 아니면 거절되어 false
     */
    @Override
    public boolean execute() {
        // 가능 여부는 현재 상태 객체가 결정한다.
        boolean ok = claim.registerResult(result);
        if (ok) System.out.println("  조사 결과 등록: " + result);
        return ok;
    }

    /** 가장 최근에 등록한 결과 한 건을 제거한다. */
    @Override
    public void undo() {
        claim.removeLastResult();
    }

    /** 이력 문구: "결과등록(사고ID, 결과문구)" */
    @Override
    public String describe() {
        return "결과등록(" + claim.id() + ", " + result + ")";
    }
}
```

</details>

<details><summary><b>5. CommandInvoker.java</b> (50줄) — 성공한 명령만 스택과 이력에 남긴다</summary>

```java
package team5;

import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.List;

/**
 * [Command 패턴: Invoker] 서버 측 Command 실행기.
 * 실행에 성공한 명령만 스택에 쌓아 마지막 명령부터 취소(undo)할 수 있게 하고,
 * 실행·취소 내역은 별도 문자열 이력(audit)으로 남긴다.
 */
public class CommandInvoker {
    /** 실행 성공한 명령 스택. push/pop 으로 가장 최근 명령이 먼저 나온다(LIFO). */
    private final Deque<FieldCommand> executed = new ArrayDeque<>();
    /** 서버 이력 로그 ("실행: ...", "취소: ...") */
    private final List<String> audit = new ArrayList<>();

    /**
     * 명령을 실행하고, 성공하면 이력에 남긴다.
     *
     * @param command 실행할 명령
     * @return 실행 성공 여부. 거절(false)된 명령은 스택·이력에 남기지 않는다.
     */
    public boolean run(FieldCommand command) {
        boolean ok = command.execute();
        if (ok) {
            executed.push(command);                       // 취소할 수 있도록 스택에 저장
            audit.add("실행: " + command.describe());     // 이력 기록
        }
        return ok;
    }

    /**
     * 가장 최근에 실행한 명령 하나를 취소한다.
     *
     * @return 취소했으면 true, 취소할 명령이 없으면 false
     */
    public boolean undoLast() {
        if (executed.isEmpty()) return false;
        FieldCommand command = executed.pop();            // 가장 최근 명령 꺼내기
        command.undo();                                   // 명령 자신이 되돌리는 방법을 안다.
        audit.add("취소: " + command.describe());
        System.out.println("  [취소] " + command.describe());
        return true;
    }

    /** 이력 조회. 외부에서 수정하지 못하도록 복사본(불변)을 돌려준다. */
    public List<String> audit() { return List.copyOf(audit); }
}
```

</details>

<details><summary><b>6. FieldTerminal.java</b> (64줄) — 오프라인이면 큐에 보관하고 복구 시 FIFO 로 전송한다</summary>

```java
package team5;

import java.util.ArrayDeque;
import java.util.Queue;

/**
 * 현장 단말. [패턴: Command (Client/Sender)]
 * 현장 사정사가 쓰는 휴대 단말로, 명령을 서버(CommandInvoker)에 전달한다.
 * 통신이 끊기면 명령을 단말 큐에 보관하고, 복구되면 보관한 순서대로 서버에 전송한다.
 * (터널 등 통신 음영 구간 대응)
 */
public class FieldTerminal {
    /** 단말 이름 (로그 출력용) */
    private final String name;
    /** 명령을 실제로 실행하는 서버 */
    private final CommandInvoker server;
    /** 통신 단절 중 보관 중인 명령 큐. FIFO 라서 제출한 순서가 유지된다. */
    private final Queue<FieldCommand> pending = new ArrayDeque<>();
    /** 현재 통신 상태 (true: 연결됨) */
    private boolean online = true;

    /**
     * @param name   단말 이름
     * @param server 명령을 전달할 서버
     */
    public FieldTerminal(String name, CommandInvoker server) {
        this.name = name;
        this.server = server;
    }

    /**
     * 명령을 제출한다.
     *   - 온라인  : 즉시 서버에서 실행
     *   - 오프라인: 단말 큐에 보관만 한다 (아직 실행되지 않음)
     *
     * @param command 제출할 명령
     */
    public void submit(FieldCommand command) {
        if (online) {
            server.run(command);
        } else {
            pending.add(command); // 큐 맨 뒤에 추가
            System.out.println("  단말 보관: " + command.describe() + " (대기 " + pending.size() + "건)");
        }
    }

    /**
     * 통신 상태를 바꾼다. 복구(true) 시에는 보관 중이던 명령을 모두 순서대로 서버로 보낸다.
     *
     * @param online true 면 통신 복구, false 면 통신 단절
     */
    public void setOnline(boolean online) {
        this.online = online;
        if (!online) {
            System.out.println("  [" + name + "] 통신 단절");
            return; // 단절 시에는 보관만 시작하면 되므로 여기서 끝낸다.
        }
        System.out.println("  [" + name + "] 통신 복구 → 보관 " + pending.size() + "건 순서대로 전송");
        // 큐에서 하나씩 꺼내(poll) 서버로 실행. 먼저 보관한 명령이 먼저 실행된다.
        while (!pending.isEmpty()) {
            server.run(pending.poll());
        }
    }
}
```

</details>

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

### ②-2 전체 코드

Visitor 패턴의 클래스 전체이다. 요소 쪽(`accept`)과 방문자 쪽(`visit`)이 어떻게 맞물리는지 보면 된다. 파일을 접어 두었으니 펼쳐서 본다. 주석의 [패턴: …] 표시가 패턴 안에서의 역할이다.

<details><summary><b>1. ClaimElement.java</b> (15줄) — accept(visitor) 하나뿐인 요소 인터페이스</summary>

```java
package team5;

/**
 * [패턴: Visitor (Element)] 사고 건의 구성 요소 인터페이스.
 * 구성 요소의 종류(사고정보·당사자·청구항목·제휴사 견적)는 고정이고, 점검 규칙(Visitor)만 계속 늘어난다.
 * 규칙이 늘어나도 요소 클래스는 수정할 필요가 없다는 점이 이 패턴의 이점이다.
 */
public interface ClaimElement {
    /**
     * 방문자를 받아들인다. 각 구현체는 자기 자신의 타입에 맞는 visit 을 호출한다. (더블 디스패치)
     *
     * @param visitor 이 요소를 방문할 방문자 (사기 점검 규칙 등)
     */
    void accept(ClaimVisitor visitor);
}
```

</details>

<details><summary><b>2. ClaimVisitor.java</b> (21줄) — 요소 종류별 visit 4개</summary>

```java
package team5;

/**
 * [패턴: Visitor] 구성 요소 종류별 visit 메서드를 선언한 방문자 인터페이스.
 * 요소 종류가 고정이라는 과제 가정에 의존한다.
 * (새 요소 종류가 생기면 이 인터페이스와 모든 구현체를 고쳐야 하는 것이 Visitor 의 한계)
 * 오버로딩된 visit 중 어떤 것이 호출될지는 요소 쪽 accept() 가 결정한다.
 */
public interface ClaimVisitor {
    /** 사고 정보(유형, 장소)를 방문 */
    void visit(AccidentInfo element);

    /** 당사자(계약자 등)를 방문 */
    void visit(Party element);

    /** 청구 항목을 방문 */
    void visit(ClaimItem element);

    /** 제휴사 견적을 방문 */
    void visit(PartnerEstimate element);
}
```

</details>

<details><summary><b>3. AccidentInfo.java</b> (13줄) — 요소 ① 사고 정보</summary>

```java
package team5;

/**
 * [Visitor Element] 사고 정보.
 *
 * @param type  사고 유형 (예: 추돌, 접촉, 전복)
 * @param place 사고 장소 (예: 서울 강변북로)
 */
public record AccidentInfo(String type, String place) implements ClaimElement {
    /** 방문자에게 자기 자신을 넘긴다. this 의 타입이 AccidentInfo 이므로 visit(AccidentInfo)가 호출된다. */
    @Override
    public void accept(ClaimVisitor visitor) { visitor.visit(this); }
}
```

</details>

<details><summary><b>4. Party.java</b> (13줄) — 요소 ② 당사자 (최근 청구 횟수)</summary>

```java
package team5;

/**
 * [Visitor Element] 당사자(계약자 등).
 *
 * @param name         당사자 구분/이름 (예: "계약자")
 * @param recentClaims 최근 6개월 청구 횟수 (단기 반복 청구 규칙이 사용)
 */
public record Party(String name, int recentClaims) implements ClaimElement {
    /** 방문자에게 자기 자신을 넘긴다. visit(Party)가 호출된다. */
    @Override
    public void accept(ClaimVisitor visitor) { visitor.visit(this); }
}
```

</details>

<details><summary><b>5. ClaimItem.java</b> (13줄) — 요소 ③ 청구 항목</summary>

```java
package team5;

/**
 * [Visitor Element] 청구 항목.
 *
 * @param name   청구 항목 이름 (예: 대물, 자차)
 * @param amount 청구 금액(원)
 */
public record ClaimItem(String name, long amount) implements ClaimElement {
    /** 방문자에게 자기 자신을 넘긴다. visit(ClaimItem)이 호출된다. */
    @Override
    public void accept(ClaimVisitor visitor) { visitor.visit(this); }
}
```

</details>

<details><summary><b>6. PartnerEstimate.java</b> (16줄) — 요소 ④ 제휴사 견적</summary>

```java
package team5;

/**
 * [Visitor Element] 제휴사(정비공장) 견적.
 *
 * @param partner         제휴 정비공장 이름
 * @param amount          공장이 제출한 견적 금액(원)
 * @param referenceAmount 표준 수리비(원). 견적이 적정한지 비교하는 기준
 * @param repeatDeals     해당 정비공장과의 반복 거래 횟수
 */
public record PartnerEstimate(String partner, long amount, long referenceAmount, int repeatDeals)
        implements ClaimElement {
    /** 방문자에게 자기 자신을 넘긴다. visit(PartnerEstimate)가 호출된다. */
    @Override
    public void accept(ClaimVisitor visitor) { visitor.visit(this); }
}
```

</details>

<details><summary><b>7. FraudRule.java</b> (58줄) — visit 4개를 빈 구현으로 두고 flag/drain 을 제공하는 기반 클래스</summary>

```java
package team5;

import java.util.ArrayList;
import java.util.List;

/**
 * [패턴: Visitor (ConcreteVisitor 기반)] 사기 점검 규칙의 기반 클래스.
 * 관심 있는 요소의 visit 만 override 하면 되고, 나머지는 아무 일도 하지 않는다.
 * 규칙은 자기 이름(title)과 만점 점수(maxPoints)를 가지며, 조건이 맞으면 flag 로 기록한다.
 *
 * 사용 흐름: FraudInspector 가 모든 요소에 이 규칙을 방문시킨 뒤 drain() 으로 걸린 항목을 수거한다.
 */
public abstract class FraudRule implements ClaimVisitor {
    /**
     * 점검에 걸린 기록 한 건.
     *
     * @param reason 걸린 항목 이름 (규칙의 title)
     * @param points 가산 점수
     * @param detail 걸린 근거 설명
     */
    public record Finding(String reason, int points, String detail) {}

    /** 이번 점검에서 걸린 기록들 (drain 으로 비워질 때까지 누적) */
    private final List<Finding> findings = new ArrayList<>();

    /** 점검 항목 이름 (예: 단기 반복 청구) */
    public abstract String title();

    /** 이 규칙이 걸렸을 때 가산되는 점수 */
    public abstract int maxPoints();

    /**
     * 규칙 조건에 해당할 때 하위 클래스가 호출하여 기록을 남긴다.
     *
     * @param detail 걸린 근거 (예: "최근 6개월 청구 4회")
     */
    protected void flag(String detail) {
        findings.add(new Finding(title(), maxPoints(), detail));
    }

    /**
     * 지금까지 기록된 점검 결과를 꺼내고 내부 목록을 비운다.
     * 비우지 않으면 같은 규칙 객체를 재사용할 때 이전 사고 건의 결과가 섞이기 때문이다.
     *
     * @return 걸린 기록의 복사본(불변)
     */
    public List<Finding> drain() {
        List<Finding> copy = List.copyOf(findings);
        findings.clear();
        return copy;
    }

    // 아래 visit 들은 "관심 없음"을 뜻하는 빈 기본 구현이다. 하위 규칙이 필요한 것만 override 한다.
    @Override public void visit(AccidentInfo element) {}
    @Override public void visit(Party element) {}
    @Override public void visit(ClaimItem element) {}
    @Override public void visit(PartnerEstimate element) {}
}
```

</details>

<details><summary><b>8. RepeatedClaimRule.java</b> (24줄) — visit(Party)만 override</summary>

```java
package team5;

/**
 * [Visitor 구현: 사기 점검 규칙] 단기 반복 청구.
 * 최근 6개월 청구가 3회 이상이면 +40점.
 * 당사자(Party) 정보만 보면 되므로 visit(Party)만 override 한다.
 */
public class RepeatedClaimRule extends FraudRule {
    /** 점검 항목 이름 */
    @Override public String title() { return "단기 반복 청구"; }

    /** 걸렸을 때 가산 점수 */
    @Override public int maxPoints() { return 40; }

    /**
     * 당사자의 최근 청구 횟수가 기준(3회) 이상이면 기록을 남긴다.
     *
     * @param element 방문한 당사자
     */
    @Override
    public void visit(Party element) {
        if (element.recentClaims() >= 3) flag("최근 6개월 청구 " + element.recentClaims() + "회");
    }
}
```

</details>

<details><summary><b>9. ExcessiveEstimateRule.java</b> (30줄) — visit(PartnerEstimate)만 override</summary>

```java
package team5;

/**
 * [Visitor 구현: 사기 점검 규칙] 견적 과다.
 * 정비공장 견적이 표준 수리비의 1.5배를 초과하면 +35점.
 * 제휴사 견적(PartnerEstimate)만 보면 되므로 visit(PartnerEstimate)만 override 한다.
 */
public class ExcessiveEstimateRule extends FraudRule {
    /** 점검 항목 이름 */
    @Override public String title() { return "견적 과다"; }

    /** 걸렸을 때 가산 점수 */
    @Override public int maxPoints() { return 35; }

    /**
     * 견적이 표준 수리비의 1.5배를 넘으면 배수와 금액을 근거로 기록한다.
     *
     * @param element 방문한 제휴사 견적
     */
    @Override
    public void visit(PartnerEstimate element) {
        if (element.amount() > element.referenceAmount() * 1.5) {
            // 정수 나눗셈을 피하려고 double 로 변환해서 배수를 계산한다.
            double ratio = (double) element.amount() / element.referenceAmount();
            // 금액은 만원 단위로 환산해서 표시한다. (예: "견적 450만원, 표준 250만원 (1.8배)")
            flag(String.format("견적 %d만원, 표준 %d만원 (%.1f배)",
                    element.amount() / 10_000, element.referenceAmount() / 10_000, ratio));
        }
    }
}
```

</details>

<details><summary><b>10. PartnerRepeatRule.java</b> (28줄) — 변경 시나리오 ③. 이 파일 하나가 추가된 코드의 전부이다</summary>

```java
package team5.ext;

import team5.FraudRule;
import team5.PartnerEstimate;

/**
 * [변경 시나리오 3] 특정 제휴 정비공장 반복 거래 규칙: 5회 이상이면 +30점.
 * FraudRule 을 상속하고 FraudInspector.add() 로 등록만 하면 되므로 기존 코드를 수정하지 않는다.
 */
public class PartnerRepeatRule extends FraudRule {
    /** 점검 항목 이름 */
    @Override public String title() { return "특정 제휴 정비공장 반복 거래"; }

    /** 걸렸을 때 가산 점수 */
    @Override public int maxPoints() { return 30; }

    /**
     * 정비공장과의 반복 거래 횟수가 기준(5회) 이상이면 기록을 남긴다.
     *
     * @param element 방문한 제휴사 견적
     */
    @Override
    public void visit(PartnerEstimate element) {
        if (element.repeatDeals() >= 5) {
            flag(element.partner() + " 반복 거래 " + element.repeatDeals() + "회");
        }
    }
}
```

</details>

<details><summary><b>11. FraudInspector.java</b> (82줄) — 규칙 × 요소로 방문시키고 Report 를 만든다</summary>

```java
package team5;

import java.util.ArrayList;
import java.util.List;

/**
 * 사기 점검기. 규칙들을 사고 건의 모든 요소에 방문시켜 점수를 합산한다.
 * 상태는 바꾸지 않고 결과(Report)만 돌려준다. 전이는 호출한 상태(PendingCalcState)가 결정한다.
 * 결과에는 걸리지 않은 항목도 0점으로 포함해서, 점검 항목별 점수를 모두 보여줄 수 있다.
 *
 * 규칙 추가는 add() 한 줄이면 되므로 기존 코드를 수정하지 않고 점검 항목을 늘릴 수 있다.
 */
public class FraudInspector {
    /**
     * 점검 항목 하나의 결과. 걸리지 않았으면 points = 0, detail = null
     *
     * @param title     항목 이름
     * @param points    실제 받은 점수
     * @param maxPoints 이 항목의 만점(걸렸을 때 점수)
     * @param detail    걸린 근거 (없으면 null)
     */
    public record RuleResult(String title, int points, int maxPoints, String detail) {}

    /**
     * 점검 전체 결과.
     *
     * @param results    항목별 결과
     * @param total      점수 합계
     * @param threshold  보류 기준 점수
     * @param suspicious 합계가 기준 이상이면 true
     */
    public record Report(List<RuleResult> results, int total, int threshold, boolean suspicious) {}

    /** 이 점수 이상이면 사기 의심으로 판단 */
    private final int threshold;
    /** 등록된 점검 규칙들 */
    private final List<FraudRule> rules = new ArrayList<>();

    /**
     * @param threshold 사기 의심 기준 점수 (합계가 이 값 이상이면 보류)
     */
    public FraudInspector(int threshold) {
        this.threshold = threshold;
    }

    /**
     * 점검 규칙을 등록한다.
     *
     * @param rule 추가할 규칙
     * @return 자기 자신 (.add(a).add(b) 처럼 이어 붙이기 위함 — 메서드 체이닝)
     */
    public FraudInspector add(FraudRule rule) {
        rules.add(rule);
        return this;
    }

    /**
     * 사고 건에 모든 규칙을 적용하고 결과 리포트를 만든다.
     *
     * @param claim 점검 대상 사고 건
     * @return 항목별 점수, 합계, 의심 여부를 담은 리포트
     */
    public Report inspect(Claim claim) {
        List<RuleResult> results = new ArrayList<>();
        int total = 0;
        for (FraudRule rule : rules) {
            // 규칙(Visitor)이 사고의 각 구성 요소를 방문한다. accept → visit 더블 디스패치.
            for (ClaimElement element : claim.elements()) {
                element.accept(rule);
            }
            // 방문 중 규칙이 남긴 기록을 수거한다. (수거하면 규칙 내부는 비워진다)
            List<FraudRule.Finding> found = rule.drain();
            int points = found.stream().mapToInt(FraudRule.Finding::points).sum();
            // 여러 건이 걸려도 대표로 첫 번째 근거만 표시한다. 안 걸렸으면 null.
            String detail = found.isEmpty() ? null : found.get(0).detail();
            results.add(new RuleResult(rule.title(), points, rule.maxPoints(), detail));
            total += points;
        }
        // 합계가 기준 이상이면 suspicious = true
        return new Report(results, total, threshold, total >= threshold);
    }
}
```

</details>

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
