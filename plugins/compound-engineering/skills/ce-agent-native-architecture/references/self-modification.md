<overview>
자가 수정(Self-modification)은 에이전트 네이티브 엔지니어링의 고급 단계입니다. 에이전트가 자신의 코드, 프롬프트, 행동을 스스로 진화시킬 수 있는 능력을 의미합니다. 모든 앱에 필요한 것은 아니지만, 미래의 큰 부분을 차지할 영역입니다.

이는 "개발자가 할 수 있는 모든 것은 에이전트도 할 수 있다"는 원칙의 논리적 확장입니다.
</overview>

<why_self_modification>
## 왜 자가 수정인가?

전통적인 소프트웨어는 정적입니다. 작성된 대로만 작동하고 그 이상은 하지 못합니다. 하지만 자가 수정 에이전트는 다음과 같은 일을 할 수 있습니다:

- **자신의 버그 수정** - 오류를 발견하고, 코드를 패치하고, 재시작함
- **새로운 능력 추가** - 사용자가 새로운 기능을 요청하면 에이전트가 직접 구현함
- **행동 진화** - 피드백으로부터 학습하고 프롬프트를 조정함
- **직접 배포** - 코드를 푸시하고, 빌드를 트리거하고, 재시작함

에이전트는 고정된 코드가 아니라 시간이 지남에 따라 스스로 개선되는 살아있는 시스템이 됩니다.
</why_self_modification>

<capabilities>
## 자가 수정이 가능하게 하는 것들

**코드 수정:**
- 소스 파일 읽기 및 이해
- 수정 사항 및 신규 기능 작성
- 버전 관리 시스템에 커밋(commit) 및 푸시(push)
- 빌드 트리거 및 검증

**프롬프트 진화:**
- 피드백에 기반하여 시스템 프롬프트 편집
- 새로운 기능을 프롬프트 섹션으로 추가
- 제대로 작동하지 않는 판단 기준 개선

**인프라 제어:**
- 업스트림(upstream)으로부터 최신 코드 가져오기(pull)
- 다른 브랜치/인스턴스로부터 병합(merge)
- 변경 사항 적용 후 재시작
- 문제가 생겼을 때 롤백(roll back)

**사이트/출력 생성:**
- 웹사이트 생성 및 유지 관리
- 문서 작성
- 데이터를 활용한 대시보드 구축
</capabilities>

<guardrails>
## 필수 가드레일

자가 수정은 강력하지만 안전 장치가 필요합니다.

**코드 변경에 대한 승인 게이트:**
```typescript
tool("write_file", async ({ path, content }) => {
  if (isCodeFile(path)) {
    // 즉시 적용하지 않고 승인 대기 상태로 저장
    pendingChanges.set(path, content);
    const diff = generateDiff(path, content);
    return { text: `승인이 필요합니다:\n\n${diff}\n\n적용하려면 "yes"라고 답해주세요.` };
  }
  // 코드가 아닌 파일은 즉시 적용
  writeFileSync(path, content);
  return { text: `${path} 쓰기 완료` };
});
```

**변경 전 자동 커밋:**
```typescript
tool("self_deploy", async () => {
  // 현재 상태를 먼저 저장
  runGit("stash");  // 또는 커밋되지 않은 변경 사항 커밋

  // 그 후 pull/merge 수행
  runGit("fetch origin");
  runGit("merge origin/main --no-edit");

  // 빌드 및 검증
  runCommand("npm run build");

  // 빌드 성공 시에만 재시작 예약
  scheduleRestart();
});
```

**빌드 검증:**
```typescript
// 빌드가 통과하지 않으면 재시작하지 않음
try {
  runCommand("npm run build", { timeout: 120000 });
} catch (error) {
  // 병합 취소
  runGit("merge --abort");
  return { text: "빌드 실패, 배포를 중단합니다", isError: true };
}
```

**재시작 후 상태 점검 (Health check):**
```typescript
tool("health_check", async () => {
  const uptime = process.uptime();
  const buildValid = existsSync("dist/index.js");
  const gitClean = !runGit("status --porcelain");

  return {
    text: JSON.stringify({
      status: "healthy",
      uptime: `${Math.floor(uptime / 60)}m`,
      build: buildValid ? "valid" : "missing",
      git: gitClean ? "clean" : "uncommitted changes",
    }, null, 2),
  };
});
```
</guardrails>

<git_architecture>
## Git 기반 자가 수정

Git을 자가 수정의 토대로 사용하십시오. Git은 다음을 제공합니다:
- 버전 이력 (롤백 능력)
- 브랜칭 (안전한 실험)
- 병합 (다른 인스턴스와 동기화)
- Push/Pull (배포 및 협업)

**필수 Git 도구:**
```typescript
tool("status", "git 상태 표시", {}, ...);
tool("diff", "파일 변경 사항 표시", { path: z.string().optional() }, ...);
tool("log", "커밋 이력 표시", { count: z.number() }, ...);
tool("commit_code", "코드 변경 사항 커밋", { message: z.string() }, ...);
tool("git_push", "GitHub으로 푸시", { branch: z.string().optional() }, ...);
tool("pull", "GitHub으로부터 가져오기", { source: z.enum(["main", "instance"]) }, ...);
tool("rollback", "최근 커밋 되돌리기", { commits: z.number() }, ...);
```

**멀티 인스턴스 아키텍처:**
```
main                      # 공유 코드
├── instance/bot-a       # 인스턴스 A의 브랜치
├── instance/bot-b       # 인스턴스 B의 브랜치
└── instance/bot-c       # 인스턴스 C의 브랜치
```

각 인스턴스는 다음을 수행할 수 있습니다:
- main으로부터 업데이트 가져오기(pull)
- 개선 사항을 main으로 푸시(push) (PR을 통해)
- 다른 인스턴스로부터 기능 동기화
- 인스턴스 전용 설정 유지
</git_architecture>

<prompt_evolution>
## 스스로 진화하는 프롬프트

시스템 프롬프트는 에이전트가 읽고 쓸 수 있는 파일입니다.

```typescript
// 에이전트는 자신의 프롬프트를 읽을 수 있음
tool("read_file", ...);  // src/prompts/system.md를 읽을 수 있음

// 에이전트는 변경 사항을 제안할 수 있음
tool("write_file", ...);  // src/prompts/system.md에 쓰기 가능 (승인 절차 포함)
```

**살아있는 문서로서의 시스템 프롬프트:**
```markdown
## 피드백 처리

누군가 피드백을 공유하면:
1. 따뜻하게 응대하십시오.
2. 중요도를 1-5점으로 평가하십시오.
3. 피드백 도구를 사용하여 저장하십시오.

<!-- 자기 앞의 메모: 비디오 워크스루는 항상 4-5점으로 평가해야 함.
     2024-12-07 Dan의 피드백으로부터 학습함 -->
```

에이전트는 다음과 같은 일을 할 수 있습니다:
- 자신을 위한 메모 추가
- 판단 기준 정교화
- 새로운 기능 섹션 추가
- 학습한 엣지 케이스 기록
</prompt_evolution>

<when_to_use>
## 언제 자가 수정을 도입해야 하는가

**좋은 대상:**
- 장기 실행되는 자율 에이전트
- 피드백에 적응해야 하는 에이전트
- 행동 진화가 가치 있는 시스템
- 빠른 반복이 중요한 내부용 도구

**필요하지 않은 경우:**
- 단순한 단일 작업 에이전트
- 규제가 엄격한 환경
- 행동을 엄격히 감사(audit)해야 하는 시스템
- 일회성 또는 수명이 짧은 에이전트

처음에는 자가 수정 기능이 없는 프롬프트 네이티브 에이전트로 시작하십시오. 필요성이 느껴질 때 자가 수정 기능을 추가하십시오.
</when_to_use>

<example_tools>
## 자가 수정 도구 모음 예시

```typescript
const selfMcpServer = createSdkMcpServer({
  name: "self",
  version: "1.0.0",
  tools: [
    // 파일 작업
    tool("read_file", "프로젝트 파일 읽기", { path: z.string() }, ...),
    tool("write_file", "파일 쓰기 (코드는 승인 필요)", { path, content }, ...),
    tool("list_files", "디렉토리 내용 나열", { path: z.string() }, ...),
    tool("search_code", "패턴 검색", { pattern: z.string() }, ...),

    // 승인 워크플로우
    tool("apply_pending", "승인된 변경 사항 적용", {}, ...),
    tool("get_pending", "대기 중인 변경 사항 표시", {}, ...),
    tool("clear_pending", "대기 중인 변경 사항 삭제", {}, ...),

    // 재시작
    tool("restart", "재빌드 및 재시작", {}, ...),
    tool("health_check", "봇 상태 점검", {}, ...),
  ],
});

const gitMcpServer = createSdkMcpServer({
  name: "git",
  version: "1.0.0",
  tools: [
    // 상태
    tool("status", "git 상태 표시", {}, ...),
    tool("diff", "변경 사항 표시", { path: z.string().optional() }, ...),
    tool("log", "이력 표시", { count: z.number() }, ...),

    // 커밋 및 푸시
    tool("commit_code", "코드 변경 사항 커밋", { message: z.string() }, ...),
    tool("git_push", "GitHub으로 푸시", { branch: z.string().optional() }, ...),

    // 동기화
    tool("pull", "업스트림으로부터 가져오기", { source: z.enum(["main", "instance"]) }, ...),
    tool("self_deploy", "Pull, 빌드, 재시작", { source: z.enum(["main", "instance"]) }, ...),

    // 안전장치
    tool("rollback", "커밋 되돌리기", { commits: z.number() }, ...),
    tool("health_check", "상세 상태 보고서", {}, ...),
  ],
});
```
</example_tools>

<checklist>
## 자가 수정 체크리스트

자가 수정을 활성화하기 전:
- [ ] Git 기반 버전 관리 설정 완료
- [ ] 코드 변경에 대한 승인 게이트 마련
- [ ] 재시작 전 빌드 검증 프로세스
- [ ] 롤백 메커니즘 가용성
- [ ] Health check 엔드포인트 구축
- [ ] 인스턴스 정체성(Identity) 설정

구현 시:
- [ ] 에이전트가 모든 프로젝트 파일을 읽을 수 있음
- [ ] 에이전트가 파일을 쓸 수 있음 (적절한 승인 포함)
- [ ] 에이전트가 커밋 및 푸시를 할 수 있음
- [ ] 에이전트가 업데이트를 pull 할 수 있음
- [ ] 에이전트가 스스로를 재시작할 수 있음
- [ ] 필요시 에이전트가 롤백할 수 있음
</checklist>
