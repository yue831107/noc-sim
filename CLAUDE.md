# Development Guidelines — NoC C++ Behavior Model

## Philosophy

### Core Beliefs

- **Incremental progress over big bangs** — Small changes that compile and pass tests
- **Learning from existing code** — Study and plan before implementing
- **Pragmatic over dogmatic** — Adapt to project reality
- **Clear intent over clever code** — Be boring and obvious

### Simplicity Means

- Single responsibility per function/class
- Avoid premature abstractions
- No clever tricks — choose the boring solution
- If you need to explain it, it's too complex

## Process

### 1. Planning & Staging

Break complex work into 3-5 stages. Document in `IMPLEMENTATION_PLAN.md`:

```markdown
## Stage N: [Name]
**Goal**: [Specific deliverable]
**Success Criteria**: [Testable outcomes]
**Tests**: [Specific test cases]
**Status**: [Not Started|In Progress|Complete]
```

- Update status as you progress
- Remove file when all stages are done

### 2. Implementation Flow

1. **Understand** — Study existing patterns in codebase
2. **Test** — Write test first (red)
3. **Implement** — Minimal code to pass (green)
4. **Refactor** — Clean up with tests passing
5. **Commit** — With clear message linking to plan

### 3. When Stuck (After 3 Attempts)

**CRITICAL**: Maximum 3 attempts per issue, then STOP.

1. **Document what failed** — What you tried, specific error messages, why you think it failed
2. **Research alternatives** — Find 2-3 similar implementations, note different approaches
3. **Question fundamentals** — Right abstraction level? Can it split into smaller problems? Simpler approach?
4. **Try different angle** — Different pattern? Remove abstraction instead of adding?

## Technical Standards

### Architecture Principles

- **Composition over inheritance** — Use dependency injection
- **Interfaces over singletons** — Enable testing and flexibility
- **Explicit over implicit** — Clear data flow and dependencies
- **Test-driven when possible** — Never disable tests, fix them

### Code Quality

- **Every commit must**:
  - Compile successfully
  - Pass all existing tests
  - Include tests for new functionality
  - Follow project formatting/linting

- **Before committing**:
  - Run formatters/linters
  - Self-review changes
  - Ensure commit message explains "why"

### Error Handling

- Fail fast with descriptive messages
- Include context for debugging
- Handle errors at appropriate level
- Never silently swallow exceptions

## Decision Framework

When multiple valid approaches exist, choose based on:

1. **Testability** — Can I easily test this?
2. **Readability** — Will someone understand this in 6 months?
3. **Consistency** — Does this match project patterns?
4. **Simplicity** — Is this the simplest solution that works?
5. **Reversibility** — How hard to change later?

## Document Quality Rules

### Parameter Discipline (CRITICAL)

- **修改任何參數值、範圍、預設值前，必須先列出提案等待使用者核可**
- 不確定的數值寧可留空或標註 `[TBD]`，絕不猜測後當事實寫入
- 每次修改參數後，列出所有引用相同參數的其他文件，確認一致性

### Cross-File Consistency

- 編輯 spec 文件後，必須檢查並列出所有引用相同定義的其他 `.md` 檔案
- 多檔編輯時，對所有目標檔案套用相同變更，不可只改第一個就停止
- 若發現跨文件不一致，先回報差異，等使用者決定以哪個版本為準

### Technical Accuracy

- 對外部架構（FlooNoC、AMBA CHI、AMD Versal 等）的分析必須附上依據來源
- 不確定的技術事實標註 `[UNVERIFIED]`，不要假裝確定
- 引用外部設計特徵時，區分「已確認」與「推測」

### Writing Quality

- 預設使用精確、專業的工程文件語氣，避免口語化或模糊用詞
- 使用者要求 high-level summary 時，不要展開 field-level 細節
- 使用者要求精確規格時，不要給模糊概述
- 修改文件前先確認目標粒度（overview / spec / implementation guide）

## Project-Specific Rules

### Language & Communication

- 回應使用繁體中文，技術術語保持英文
- 使用者不熟悉 C++，解釋時用 RTL 類比
- 在做任何設計決策前要先詢問使用者

### Architecture

- **Namespace**: `noc::`
- **Architecture**: User Code → NocSystem Public API → Internal Components → Co-Sim Bridge (DPI-C)
- **Uniform Router**: No Edge/Compute distinction, all routers identical
- **Dual Flow Control**: Valid/Ready (Version A) AND Credit-Based (Version B), compile-time template
- **Factory Pattern**: JSON config -> factory function -> template instantiation (like RTL parameter)
- **Hot-Swap Interface**: `Router_Interface<Mode>` / `NI_Interface<Mode>` abstract base classes

### Build & Test

- C++17 standard
- Build: CMake
- Test framework: GoogleTest
- Use `py -3` instead of `python3` (Windows)
- Path separators: forward slash `/` or double backslash `\\`

### Key Design Docs

- `docs/design/README.md` — File index
- `docs/design/REVIEW_ISSUES.md` — Design decision tracker
- `docs/PROJECT_GOALS.md` — Architecture overview and goals

### Git Conventions

- Commit message format: `type(scope): description` (English)
- Types: feat, fix, docs, style, refactor, test, chore, perf
- Branch: feature work on feature branches, PRs to `main`

## Quality Gates

### Definition of Done

- [ ] Tests written and passing
- [ ] Code follows project conventions
- [ ] No linter/formatter warnings
- [ ] Commit messages are clear
- [ ] Implementation matches plan
- [ ] No TODOs without issue numbers

### Test Guidelines

- Test behavior, not implementation
- One assertion per test when possible
- Clear test names describing scenario
- Use existing test utilities/helpers
- Tests should be deterministic

## Important Reminders

**NEVER**:
- Use `--no-verify` to bypass commit hooks
- Disable tests instead of fixing them
- Commit code that doesn't compile
- Make assumptions — verify with existing code
- Reference external IP names (FlooNoC, booksim2, pcievhost, etc.) in code or docs

**ALWAYS**:
- Commit working code incrementally
- Update plan documentation as you go
- Learn from existing implementations
- Stop after 3 failed attempts and reassess
- Explain C++ concepts using RTL analogies for the user
