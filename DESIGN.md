# Workflow Task Engine - 설계 문서

## 1. 개요

Workflow Task Engine은 선형 및 병렬 단계를 지원하는 작업 흐름 관리 시스템입니다. 
각 단계마다 여러 사용자에게 Task를 할당할 수 있으며, 누구나 Task를 완료할 수 있습니다.

### 핵심 특징
- **선형 + 병렬 단계 지원**: 기본적으로 선형 진행이지만 병렬 처리 가능
- **동적 Task 할당**: 각 단계에서 여러 사용자에게 Task 할당
- **유연한 완료 정책**: 누구나 Task 완료 가능 (First Come First Served)
- **상태 추적**: 모든 단계와 Task의 상태를 기록
- **감사 로그**: 작업 이력 추적

---

## 2. 도메인 모델

### 2.1 Core Entities

#### Workflow
```
- id: UUID
- name: String
- description: String
- status: DRAFT, ACTIVE, COMPLETED, CANCELLED
- createdAt: LocalDateTime
- updatedAt: LocalDateTime
- createdBy: String (User ID)
```

#### Stage
```
- id: UUID
- workflowId: UUID
- name: String
- description: String
- order: Integer (단계 순서)
- type: SERIAL, PARALLEL (순차/병렬)
- status: PENDING, IN_PROGRESS, COMPLETED, SKIPPED
- createdAt: LocalDateTime
- updatedAt: LocalDateTime
```

#### Task
```
- id: UUID
- stageId: UUID
- title: String
- description: String
- assignees: List<String> (User IDs)
- assignedBy: String (User ID)
- status: PENDING, IN_PROGRESS, COMPLETED, REJECTED, CANCELLED
- assignedAt: LocalDateTime
- completedAt: LocalDateTime
- completedBy: String (Task를 완료한 User ID)
- updatedAt: LocalDateTime
```

#### WorkflowExecution
```
- id: UUID
- workflowId: UUID
- status: RUNNING, COMPLETED, FAILED, CANCELLED
- currentStageId: UUID
- startedAt: LocalDateTime
- completedAt: LocalDateTime
- createdBy: String (Workflow 시작한 User ID)
```

#### StageExecution
```
- id: UUID
- workflowExecutionId: UUID
- stageId: UUID
- status: PENDING, IN_PROGRESS, COMPLETED, SKIPPED
- startedAt: LocalDateTime
- completedAt: LocalDateTime
```

---

## 3. 상태 다이어그램

### 3.1 Workflow 상태 전이
```
DRAFT → ACTIVE → COMPLETED
  ↓
CANCELLED
```

### 3.2 Stage 상태 전이
```
PENDING → IN_PROGRESS → COMPLETED
  ↓           ↓
SKIPPED   CANCELLED
```

### 3.3 Task 상태 전이
```
PENDING → IN_PROGRESS → COMPLETED
  ↓           ↓
REJECTED  CANCELLED
```

---

## 4. 아키텍처

### 4.1 계층 구조

```
┌─────────────────────────────────────────┐
│         Controller/API Layer            │
│    (REST Endpoints, Request/Response)   │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         Service Layer                   │
│  - WorkflowService                      │
│  - StageService                         │
│  - TaskService                          │
│  - WorkflowExecutionService             │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         Domain Layer                    │
│  - WorkflowManager                      │
│  - StageManager                         │
│  - TaskManager                          │
│  - StateTransitionValidator             │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│       Repository/DAO Layer              │
│  - WorkflowRepository                   │
│  - StageRepository                      │
│  - TaskRepository                       │
│  - WorkflowExecutionRepository          │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│       Database Layer (Oracle)           │
└─────────────────────────────────────────┘
```

### 4.2 주요 클래스 구조

```
com.workflow.taskengine
├── domain
│   ├── model
│   │   ├── Workflow
│   │   ├── Stage
│   │   ├── Task
│   │   ├── WorkflowExecution
│   │   └── StageExecution
│   ├── enums
│   │   ├── WorkflowStatus
│   │   ├── StageStatus
│   │   ├── TaskStatus
│   │   └── StageType
│   └── manager
│       ├── WorkflowManager
│       ├── StageManager
│       ├── TaskManager
│       └── WorkflowExecutionManager
├── service
│   ├── WorkflowService
│   ├── StageService
│   ├── TaskService
│   ├── WorkflowExecutionService
│   └── StateTransitionValidator
├── repository
│   ├── WorkflowRepository
│   ├── StageRepository
│   ├── TaskRepository
│   ├── WorkflowExecutionRepository
│   └── StageExecutionRepository
├── dto
│   ├── request
│   │   ├── CreateWorkflowRequest
│   │   ├── CreateStageRequest
│   │   ├── CreateTaskRequest
│   │   └── CompleteTaskRequest
│   └── response
│       ├── WorkflowResponse
│       ├── StageResponse
│       ├── TaskResponse
│       └── WorkflowExecutionResponse
├── controller
│   ├── WorkflowController
│   ├── StageController
│   ├── TaskController
│   └── WorkflowExecutionController
└── exception
    ├── WorkflowException
    ├── StageException
    ├── TaskException
    └── InvalidStateTransitionException
```

---

## 5. 핵심 업무 흐름

### 5.1 Workflow 생성 및 시작

```
1. Workflow 정의 (DRAFT 상태)
   - Workflow 이름, 설명 등록
   
2. Stage 정의
   - 각 Stage 이름, 순서, 타입(SERIAL/PARALLEL) 정의
   
3. Workflow 활성화 (DRAFT → ACTIVE)
   - Validation: 모든 Stage가 올바르게 정의되었는지 확인
   
4. Workflow 실행 시작
   - WorkflowExecution 생성 (RUNNING 상태)
   - 첫 번째 Stage에 대한 StageExecution 생성
```

### 5.2 Stage 실행 및 Task 할당

```
1. Stage 시작 (PENDING → IN_PROGRESS)
   - 해당 Stage의 StageExecution 생성
   
2. Stage의 모든 Task 할당
   - 여러 사용자에게 Task 생성 및 할당
   - Task 상태: PENDING
   
3. Task 완료 대기
   - 할당된 사용자 중 누구나 Task 완료 가능
   - 첫 번째 완료 시 Task 상태: COMPLETED
```

### 5.3 Stage 완료 및 다음 Stage 진행

```
1. Stage 진행 조건 확인
   - SERIAL Stage: 모든 Task 완료 시 다음 Stage로 진행
   - PARALLEL Stage: 설정된 조건에 따라 진행
     (모든 Task 완료 OR 첫 완료 완료 등)
   
2. Stage 상태 업데이트 (IN_PROGRESS → COMPLETED)
   - StageExecution 완료 시간 기록
   
3. 다음 Stage 시작
   - 다음 Stage가 있으면 새로운 StageExecution 생성
   - 마지막 Stage이면 Workflow 완료
```

### 5.4 Task 완료 프로세스

```
1. Task 완료 요청
   - completedBy: 완료한 사용자 ID
   - completedAt: 완료 시간
   
2. 상태 검증
   - 유효한 상태 전이인지 확인 (PENDING → COMPLETED)
   
3. Task 상태 업데이트
   - Task 상태: COMPLETED
   - 다른 미완료 Task는 유지 또는 자동 취소 (설정에 따라)
   
4. Stage 완료 여부 판단
   - 모든 Task 완료 조건 확인
   - 조건 만족 시 Stage 진행
```

---

## 6. 데이터베이스 설계

### 6.1 테이블 구조 (Oracle SQL)

#### WORKFLOWS 테이블
```sql
CREATE TABLE workflows (
    id VARCHAR2(36) PRIMARY KEY,
    name VARCHAR2(255) NOT NULL,
    description CLOB,
    status VARCHAR2(50) NOT NULL, -- DRAFT, ACTIVE, COMPLETED, CANCELLED
    created_by VARCHAR2(100) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

#### STAGES 테이블
```sql
CREATE TABLE stages (
    id VARCHAR2(36) PRIMARY KEY,
    workflow_id VARCHAR2(36) NOT NULL,
    name VARCHAR2(255) NOT NULL,
    description CLOB,
    stage_order NUMBER NOT NULL,
    stage_type VARCHAR2(50) NOT NULL, -- SERIAL, PARALLEL
    status VARCHAR2(50) NOT NULL, -- PENDING, IN_PROGRESS, COMPLETED, SKIPPED
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    FOREIGN KEY (workflow_id) REFERENCES workflows(id)
);

CREATE INDEX idx_stages_workflow_id ON stages(workflow_id);
```

#### TASKS 테이블
```sql
CREATE TABLE tasks (
    id VARCHAR2(36) PRIMARY KEY,
    stage_id VARCHAR2(36) NOT NULL,
    title VARCHAR2(255) NOT NULL,
    description CLOB,
    assigned_by VARCHAR2(100) NOT NULL,
    status VARCHAR2(50) NOT NULL, -- PENDING, IN_PROGRESS, COMPLETED, REJECTED, CANCELLED
    assigned_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    completed_by VARCHAR2(100),
    updated_at TIMESTAMP NOT NULL,
    FOREIGN KEY (stage_id) REFERENCES stages(id)
);

CREATE INDEX idx_tasks_stage_id ON tasks(stage_id);
CREATE INDEX idx_tasks_status ON tasks(status);
```

#### TASK_ASSIGNEES 테이블 (다대다 관계)
```sql
CREATE TABLE task_assignees (
    task_id VARCHAR2(36) NOT NULL,
    user_id VARCHAR2(100) NOT NULL,
    assigned_at TIMESTAMP NOT NULL,
    PRIMARY KEY (task_id, user_id),
    FOREIGN KEY (task_id) REFERENCES tasks(id)
);

CREATE INDEX idx_task_assignees_user_id ON task_assignees(user_id);
```

#### WORKFLOW_EXECUTIONS 테이블
```sql
CREATE TABLE workflow_executions (
    id VARCHAR2(36) PRIMARY KEY,
    workflow_id VARCHAR2(36) NOT NULL,
    status VARCHAR2(50) NOT NULL, -- RUNNING, COMPLETED, FAILED, CANCELLED
    current_stage_id VARCHAR2(36),
    created_by VARCHAR2(100) NOT NULL,
    started_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    FOREIGN KEY (workflow_id) REFERENCES workflows(id),
    FOREIGN KEY (current_stage_id) REFERENCES stages(id)
);

CREATE INDEX idx_workflow_executions_workflow_id ON workflow_executions(workflow_id);
```

#### STAGE_EXECUTIONS 테이블
```sql
CREATE TABLE stage_executions (
    id VARCHAR2(36) PRIMARY KEY,
    workflow_execution_id VARCHAR2(36) NOT NULL,
    stage_id VARCHAR2(36) NOT NULL,
    status VARCHAR2(50) NOT NULL, -- PENDING, IN_PROGRESS, COMPLETED, SKIPPED
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    FOREIGN KEY (workflow_execution_id) REFERENCES workflow_executions(id),
    FOREIGN KEY (stage_id) REFERENCES stages(id)
);

CREATE INDEX idx_stage_executions_workflow_execution_id ON stage_executions(workflow_execution_id);
```

#### AUDIT_LOGS 테이블
```sql
CREATE TABLE audit_logs (
    id VARCHAR2(36) PRIMARY KEY,
    entity_type VARCHAR2(100) NOT NULL, -- WORKFLOW, STAGE, TASK
    entity_id VARCHAR2(36) NOT NULL,
    action VARCHAR2(100) NOT NULL, -- CREATED, UPDATED, COMPLETED, REJECTED
    performed_by VARCHAR2(100) NOT NULL,
    details CLOB,
    created_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_audit_logs_entity_id ON audit_logs(entity_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);
```

---

## 7. 주요 알고리즘

### 7.1 Stage 진행 결정 알고리즘

```java
boolean canAdvanceToNextStage(Stage stage, List<Task> tasks) {
    if (stage.getType() == StageType.SERIAL) {
        // SERIAL: 모든 Task 완료 필요
        return tasks.stream()
            .allMatch(task -> task.getStatus() == TaskStatus.COMPLETED);
    } else if (stage.getType() == StageType.PARALLEL) {
        // PARALLEL: 조건에 따라 결정
        // - 모든 Task 완료
        // - 첫 번째 완료
        // - N개 이상 완료 등
        return checkParallelAdvanceCondition(stage, tasks);
    }
    return false;
}
```

### 7.2 다음 Stage 결정 알고리즘

```java
Stage getNextStage(Stage currentStage) {
    // workflow에서 order > currentStage.order인 다음 Stage 조회
    List<Stage> stages = workflow.getStages();
    return stages.stream()
        .filter(s -> s.getOrder() > currentStage.getOrder())
        .min(Comparator.comparingInt(Stage::getOrder))
        .orElse(null); // null이면 Workflow 완료
}
```

### 7.3 Task 완료 시 처리

```java
void completeTask(Task task, String completedBy) {
    // 1. Task 상태 업데이트
    task.setStatus(TaskStatus.COMPLETED);
    task.setCompletedBy(completedBy);
    task.setCompletedAt(LocalDateTime.now());
    
    // 2. 같은 Stage의 다른 Task 처리
    Stage stage = task.getStage();
    List<Task> stageTasks = stage.getTasks();
    
    if (stage.getType() == StageType.SERIAL) {
        // SERIAL: 다른 미완료 Task 취소
        stageTasks.stream()
            .filter(t -> !t.getId().equals(task.getId()))
            .filter(t -> t.getStatus() != TaskStatus.COMPLETED)
            .forEach(t -> t.setStatus(TaskStatus.CANCELLED));
    } else if (stage.getType() == StageType.PARALLEL) {
        // PARALLEL: 다른 Task 유지 또는 설정에 따라 처리
    }
    
    // 3. Stage 진행 가능 여부 확인
    if (canAdvanceToNextStage(stage, stageTasks)) {
        advanceWorkflow(stage);
    }
}
```

---

## 8. API 엔드포인트 (예시)

### Workflow 관리
- `POST /api/workflows` - Workflow 생성
- `GET /api/workflows/{id}` - Workflow 조회
- `PUT /api/workflows/{id}` - Workflow 수정
- `POST /api/workflows/{id}/activate` - Workflow 활성화
- `POST /api/workflows/{id}/execute` - Workflow 실행 시작

### Stage 관리
- `POST /api/workflows/{workflowId}/stages` - Stage 생성
- `GET /api/workflows/{workflowId}/stages` - Stage 목록 조회
- `PUT /api/stages/{id}` - Stage 수정

### Task 관리
- `POST /api/stages/{stageId}/tasks` - Task 생성
- `GET /api/stages/{stageId}/tasks` - Task 목록 조회
- `PUT /api/tasks/{id}/complete` - Task 완료
- `PUT /api/tasks/{id}/reject` - Task 거절 (옵션)

### Workflow 실행
- `GET /api/executions/{id}` - 실행 상태 조회
- `GET /api/executions/{id}/stages` - 실행 중인 Stage 조회
- `GET /api/executions/{id}/audit-logs` - 감사 로그 조회

---

## 9. 주요 고려사항

### 9.1 동시성 처리
- 같은 Task를 여러 사용자가 동시에 완료할 경우 대응
- Database Lock 또는 Optimistic Lock 사용
- 트랜잭션 관리

### 9.2 상태 일관성
- 상태 전이 검증: 유효하지 않은 전이 방지
- 데이터베이스 제약조건 활용

### 9.3 확장성
- 복잡한 병렬 조건 지원 (향후)
- 조건부 분기 지원 (향후)
- Task 우선순위 지원 (향후)

### 9.4 모니터링
- 감사 로그 기록
- 각 Stage별 소요 시간 추적
- Task 완료 시간 분석

---

## 10. 향후 확장 계획

1. **복잡한 병렬 조건**: AND, OR 조건 조합
2. **조건부 분기**: 특정 조건에 따라 다른 Stage로 이동
3. **Task 우선순위**: 높은 우선순위 Task부터 처리
4. **Task 재할당**: Task 거절 시 다시 할당
5. **알림 시스템**: Task 할당/완료 시 알림
6. **웹훅**: 외부 시스템 연동
7. **동적 Stage**: 런타임 중 Stage 추가/제거

---

## 11. 기술 스택

- **Language**: Java 8+
- **Framework**: Spring Framework / Spring Boot
- **Database**: Oracle Database
- **ORM**: JPA / Hibernate
- **Build**: Maven
- **Testing**: JUnit 5, Mockito
- **Logging**: SLF4J, Logback
- **API**: REST (Spring MVC)

---

이 설계 문서를 기반으로 구현을 진행할 준비가 되었습니다.
다음 단계:
1. 핵심 Entity 클래스 개발
2. Repository 인터페이스 정의
3. Service 로직 구현
4. Controller API 개발
5. 단위 테스트 작성
