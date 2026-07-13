---
layout: post
title: "제한된 네트워크 환경에서 Agent와 Skill로 확장하는 AI 개발 자동화"
author: haewonshin
tag: [ GitHub Copilot, Copilot CLI, Agentic Workflow, Harness Engineering, GitHub Actions, Foundry Model]
category: [ Solution ]
image: assets/images/thumnails/GitHub_Copilot_CLI_Icon.png
---

제한된 네트워크 환경에서 GitHub Copilot CLI를 Microsoft Foundry 모델과 연결하면, 개발자는 터미널에서 AI 기반 코드 생성과 분석을 수행할 수 있습니다. 하지만 실제 엔터프라이즈 환경에서는 단순히 `copilot` 명령어를 실행하는 것만으로는 충분하지 않습니다. 반복 가능한 작업 단위, 입력 컨텍스트, 출력 형식, 검증 기준, 승인 절차가 함께 설계되어야 합니다.

이 포스팅에서는 이전 글인 [제한된 네트워크 환경에서 구성하는 AI 기반 개발환경](/ai-development-environment-in-priave-network/)의 Copilot CLI 환경 구성을 바탕으로, agent와 skill을 활용한 agentic workflow 설계 방법을 설명합니다. 여기서 agent와 skill은 managed cloud agent 제품을 의미하기보다, 조직 내부에서 반복 가능한 AI 작업을 정의하고 실행하기 위한 역할 정의와 작업 레시피를 의미합니다.

> **Note** 이 글의 예시는 제한된 네트워크 환경에서도 적용할 수 있는 설계 패턴을 설명하지만, 모든 기능이 네트워크 차단 상태에서 자동으로 동작하는 것은 아닙니다. GitHub Actions marketplace 액션, artifact 업로드, PR/issue comment 작성, Microsoft Foundry 또는 Azure OpenAI 엔드포인트 호출, Azure 인증 토큰 발급은 각 엔드포인트에 대한 네트워크 허용과 사전 구성이 필요합니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">Agentic Workflow를 구성하는 기본 개념

최근 AI 기반 개발 도구는 단순히 질문에 답하거나 코드를 한 번 생성하는 수준을 넘어, 특정 역할을 가진 agent가 작업을 계획하고 필요한 skill을 호출하며 결과를 검증하는 agentic workflow 형태로 발전하고 있습니다. 개발자 입장에서는 “AI에게 무엇을 시킬 것인가”보다 “어떤 역할, 어떤 절차, 어떤 검증 경계 안에서 AI가 일하게 할 것인가”가 더 중요해지고 있습니다.

이런 흐름을 엔터프라이즈 환경에 적용하려면 Agent, Skill, Harness를 분리해서 설계하는 것이 좋습니다. 세 개념을 분리하면 AI 작업을 재사용 가능한 단위로 만들 수 있고, 보안팀이나 운영팀도 어떤 입력이 사용되고 어떤 결과가 생성되는지 검토하기 쉬워집니다.

| 구성 요소 | 정의 | 사용하는 목적 | 예시 |
| --- | --- | --- |
| Agent | 특정 목적과 책임을 가진 AI 작업자 역할 정의 | 보안 리뷰, 테스트 분석, 문서 생성처럼 반복되는 업무를 역할 단위로 분리하고, 출력 형식과 금지사항을 명확히 하기 위해 사용 | 보안 리뷰 에이전트, 테스트 실패 분석 에이전트, 배포 스크립트 리뷰 에이전트 등 |
| Skill | Agent가 수행할 수 있는 재사용 가능한 작업 절차와 도메인 지식 | diff 요약, 로그 분석, IaC 검토처럼 여러 agent나 workflow에서 반복되는 절차를 재사용하기 위해 사용 | diff 분석, 실패 로그 요약, Terraform 변경 검토, 릴리즈 노트 생성 |
| Harness | Agent와 Skill을 안전하게 실행하기 위한 실행 설계와 통제 계층 | 입력 수집, 민감정보 제거, 모델 호출, 출력 검증, 승인 절차를 표준화해 AI 작업을 운영 가능한 형태로 만들기 위해 사용 | 컨텍스트 수집, 프롬프트 조립, Copilot CLI 실행, 결과 저장, 테스트/정책 검증 |

<br>이 구조의 핵심은 AI에게 모든 것을 맡기는 것이 아니라, AI가 수행할 작업 범위와 입력/출력 계약을 명확히 제한하는 것입니다. 특히 금융권과 엔터프라이즈 환경에서는 agentic workflow를 구성하더라도 자동 반영보다 “AI 제안 → 검증 → 사람 승인” 흐름을 유지하는 것이 안전합니다.

Agent와 Skill을 문서형 설정으로 관리하면 팀이 같은 기준으로 AI 작업을 반복 실행할 수 있고, 변경 이력도 코드처럼 리뷰할 수 있습니다. Harness engineering은 이 작업이 실제 개발자 로컬 환경, GitHub Actions runner 또는 제한된 네트워크 환경에서 일관되게 실행되도록 만드는 운영 설계에 가깝습니다. 따라서 다음 단계에서는 이 세 개념을 저장소 안에서 어떻게 배치할지 살펴보겠습니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">GHCP를 이용해 구성할 수 있는 권장 디렉터리 구조

Agent와 Skill은 코드와 같이 버전 관리되는 문서형 설정으로 관리하는 것이 좋습니다. GHCP/VS Code agent mode에서 사용할 수 있는 표준 커스터마이징 구조를 기준으로 다음과 같이 구성할 수 있습니다.

```text
.github/
  copilot-instructions.md
  instructions/
    security-review.instructions.md
    test-writing.instructions.md
  prompts/
    review-pr.prompt.md
    analyze-test-failure.prompt.md
  agents/
    security-review.agent.md
    test-failure-analysis.agent.md
    deployment-review.agent.md
  skills/
    summarize-diff/
      SKILL.md
    analyze-test-log/
      SKILL.md
    review-infra-change/
      SKILL.md
    draft-release-note/
      SKILL.md
  harness/
    context/
      README.md
    policies/
      output-policy.md
      sensitive-data-policy.md
    schemas/
      review-output.md
    scripts/
      collect-context.sh
      validate-output.sh
      mask-sensitive-data.sh
    outputs/
```

이 구조는 크게 두 영역으로 나눌 수 있습니다. `.github/agents`와 `.github/skills`는 GHCP와 Copilot CLI가 활용할 수 있는 agent/skill 정의 영역이고, `.github/harness`는 agent와 skill을 실제 개발자 환경이나 runner에서 안전하게 실행하기 위한 운영 설계 영역입니다. 중요한 것은 “AI가 어떤 역할로 어떤 작업을 수행하는가”와 “그 작업을 어떤 입력, 검증, 승인 절차로 실행하는가”를 분리하는 것입니다.

| 경로 | 용도 | GHCP/VS Code에서의 사용 | Copilot CLI에서의 사용 |
| --- | --- | --- | --- |
| `.github/copilot-instructions.md` | 저장소 전체에 적용할 공통 개발 규칙, 보안 원칙, 코딩 스타일 정의 | 저장소 공통 instructions로 활용 | CLI도 프로젝트 instructions로 활용할 수 있습니다. `copilot init` 또는 `/init`으로 생성/개선할 수 있습니다. |
| `.github/instructions/*.instructions.md` | 특정 파일, 폴더, 언어, 작업 유형에 적용할 세부 지침 정의 | Copilot Chat 또는 agent mode에서 상황별 instructions로 활용 | CLI 환경과 버전에 따라 custom instructions로 활용할 수 있습니다. 자동화에서는 필요한 지침을 명시적으로 프롬프트에 포함하는 방식도 검토합니다. |
| `.github/prompts/*.prompt.md` | 반복 실행할 단일 작업 프롬프트 정의 | Copilot Chat에서 재사용 가능한 prompt로 활용 | CLI의 agent/skill 자동 라우팅 대상이라기보다, 자동화에서는 파일 내용을 `copilot -p` 입력으로 전달하는 방식이 명확합니다. |
| `.github/agents/*.agent.md` | 특정 역할을 가진 custom agent 정의 | 역할 기반 custom agent로 활용 | CLI도 project custom agent로 로드할 수 있습니다. `/agent`에서 선택하거나 `--agent <agent-id>`로 명시 호출할 수 있으며, 설명에 따라 agent 사용을 추론할 수도 있습니다. |
| `.github/skills/<skill-name>/SKILL.md` | 특정 작업 절차와 도메인 지식을 skill 단위로 패키징 | skill로 활용 | CLI도 project skill로 로드할 수 있습니다. `/skills` 또는 `copilot skill`로 관리하고, `/SKILL-NAME` 또는 agent의 자동 호출로 사용할 수 있습니다. |
| `.github/harness/context/` | agent 실행에 전달할 입력 컨텍스트 설명과 수집 기준 | GHCP 표준 인식 경로는 아니며, 운영 문서로 참고 | diff, 로그, 설정 파일 등 어떤 입력을 전달할지 정리하는 내부 기준 |
| `.github/harness/policies/` | 출력 정책, 민감정보 처리, 승인 기준 정의 | GHCP 표준 인식 경로는 아니며, 운영 문서로 참고 | 자동화 전후 검증 기준, 민감정보 마스킹 기준, 승인 조건 정의 |
| `.github/harness/schemas/` | agent 출력 형식과 필수 섹션 정의 | GHCP 표준 인식 경로는 아니며, 운영 문서로 참고 | `Summary`, `Findings`, `Tests`, `Next Actions` 같은 출력 형식 검증 기준 |
| `.github/harness/scripts/` | 컨텍스트 수집, 민감정보 마스킹, 출력 검증 스크립트 | GHCP 표준 인식 경로는 아니며, 필요 시 수동 또는 task로 실행 | runner나 로컬 개발환경에서 `copilot` 실행 전후에 호출할 보조 스크립트 |
| `.github/harness/outputs/` | 로컬 또는 runner 실행 결과 저장 | GHCP 표준 구조는 아니며, 산출물 보관용 임시 폴더 | context, prompt, output, 검증 결과 파일 저장용 임의 폴더 |

<br/>
> **중요한 구분** GHCP/VS Code agent mode와 Copilot CLI 모두 `.github/agents`와 `.github/skills` 구조를 활용할 수 있습니다. 다만 runner나 headless 자동화에서는 어떤 agent와 skill이 사용되는지 재현 가능해야 하므로, `--agent <agent-id>`처럼 명시적으로 agent를 지정하거나 필요한 skill 이름을 프롬프트에 명확히 포함하는 방식을 권장합니다.

> **Harness 구조에 대한 구분** `.github/harness`는 GHCP나 Copilot CLI가 자동으로 agent 또는 skill로 인식하는 표준 폴더가 아닙니다. 이 폴더는 조직이 agentic workflow를 안전하게 운영하기 위해 추가하는 내부 실행 설계 영역입니다. 실제 구현에서는 shell script, GitHub Actions workflow, 사내 task runner, 또는 보안 검증 도구가 이 폴더의 정책과 스크립트를 사용하게 됩니다.

좋은 harness 구조는 다음 질문에 답할 수 있어야 합니다.

- 어떤 파일과 로그를 AI에게 전달하는가?
- 민감정보는 전달 전에 제거되는가?
- Agent와 Skill 정의는 어떤 버전인가?
- 출력은 어떤 형식이어야 하는가?
- 결과가 정책을 위반하면 어떻게 실패 처리하는가?
- 사람이 검토하기 전까지 코드나 배포에 반영되지 않는가?

이 글에서는 별도의 harness 스크립트 구현을 제공하지 않습니다. 대신 개발자 로컬 환경과 GitHub Actions runner 예시에서 `.github/agents`, `.github/skills`, `.github/harness/outputs`를 조합해 실행 흐름을 설명합니다. 실제 운영 환경에서 harness를 구현한다면 secret masking, context size 제한, 출력 schema 검증, 금칙어 검사, artifact 저장 같은 기능을 별도 스크립트나 workflow step으로 추가해야 합니다.

> **제한된 네트워크 환경 확인 사항** 위 파일 구조 자체는 로컬 저장소 안에서 동작하므로 외부 네트워크가 필요하지 않습니다. 다만 agent 또는 skill이 외부 패키지 설치, 원격 문서 다운로드, 외부 API 호출을 전제로 작성되면 해당 경로를 별도로 허용하거나 내부 mirror를 사용해야 합니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">위 구조를 제한된 네트워크 환경에서 Copilot CLI로 활용하기

앞에서 정리한 `.github/agents`, `.github/skills`, `.github/harness` 구조는 GHCP/VS Code agent mode뿐 아니라 Copilot CLI 기반 자동화에서도 활용할 수 있습니다. 제한된 네트워크 환경에서는 GHCP/VS Code agent mode보다 Copilot CLI 기반 구성이 더 통제하기 쉬운 경우가 있습니다. Copilot CLI는 `COPILOT_OFFLINE=true`와 `COPILOT_PROVIDER_*` 환경변수를 사용해 GitHub 서버를 경유하지 않고 조직이 허용한 Microsoft Foundry 또는 Azure OpenAI 엔드포인트를 직접 호출하도록 구성할 수 있기 때문입니다.

이때 구성 가능한 범위는 다음과 같이 정리할 수 있습니다.

| 구성 영역 | Copilot CLI로 가능한 구성 | 구성을 위한 Copilot CLI 명령 예시 |
| --- | --- | --- |
| Agent 선택 | `.github/agents/*.agent.md`를 project custom agent로 두고 `--agent <agent-id>`로 명시 실행 | `copilot --agent security-review --add-dir ./.github/agents --add-dir ./src -p "Use .github/agents/security-review.agent.md and review all code under ./src"` |
| Skill 활용 | `.github/skills/<skill-name>/SKILL.md`를 project skill로 두고 `/skills`, `/SKILL-NAME`, `copilot skill` 또는 agent의 자동 호출로 활용 | `copilot skill list`<br>`copilot --add-dir ./.github/skills --add-dir ./src -p "Use .github/skills/summarize-diff/SKILL.md to review all changes under ./src"` |
| 모델 호출 | `COPILOT_PROVIDER_BASE_URL`, `COPILOT_PROVIDER_TYPE`, `COPILOT_MODEL`, `COPILOT_OFFLINE` 등으로 Foundry/Azure OpenAI 직접 호출 | `export COPILOT_PROVIDER_TYPE="azure"`<br>`export COPILOT_PROVIDER_BASE_URL="https://<resource-name>.cognitiveservices.azure.com"`<br>`export COPILOT_MODEL="<deployment-name>"`<br>`export COPILOT_OFFLINE="true"`<br>`copilot -p "hello"` |
| 자동 실행 | `copilot -p`, `--agent`, `--model`, `--allow-tool`, `--secret-env-vars` 등을 조합해 로컬 script 또는 GitHub Actions runner에서 실행 | `copilot --agent security-review --add-dir ./.github/agents --add-dir ./.github/skills --add-dir ./.github/harness --add-dir ./src -p "Use .github/skills/summarize-diff/SKILL.md. Review ./src and follow .github/harness/policies/output-policy.md" --secret-env-vars="COPILOT_PROVIDER_API_KEY" > .github/harness/outputs/agent-output.md` |
| 실행 안전장치 | `.github/harness`에 컨텍스트 수집 기준, 민감정보 마스킹, 출력 검증 정책을 정의 | `copilot --agent security-review --add-dir ./.github/harness --add-dir ./src -p "Follow .github/harness/policies/output-policy.md and review all files under ./src"`<br>`grep -q "Summary" .github/harness/outputs/agent-output.md` |

<br>따라서 제한된 네트워크 환경에서는 다음과 같은 패턴이 현실적입니다.

1. `.github/agents`와 `.github/skills`에 역할과 작업 절차를 정의합니다.
2. `.github/harness`에는 어떤 입력을 수집하고, 어떤 정보를 제거하고, 어떤 출력 형식을 검증할지 정의합니다.
3. 로컬 개발환경에서는 `copilot --agent <agent-id> -p "..."` 형태로 먼저 검증합니다.
4. runner에서는 동일한 agent와 skill을 사용하되, secret 주입과 artifact 저장을 workflow에서 통제합니다.
5. 모델 출력은 자동 반영하지 않고 리뷰 가능한 문서, job summary, artifact로 남긴 뒤 사람이 승인합니다.

이 방식은 GHCP에서 사용할 수 있는 기능을 그대로 대체하지는 않지만, 제한된 네트워크 환경에서 agentic workflow를 점진적으로 구성할 수 있는 실용적인 접근 방법을 제시합니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">GHCP와 Copilot CLI에서 활용할 수 있는 Agent 구성

앞의 `.github/agents`, `.github/skills` 구조는 GHCP/VS Code agent mode뿐 아니라 Copilot CLI에서도 활용할 수 있는 커스터마이징 구조입니다. 다만 두 실행 방식의 사용 경험은 다릅니다.

즉, 로컬 개발환경에서 GHCP를 사용하는 경우에는 다음과 같이 두 가지 방식을 구분하는 것이 좋습니다.

| 구분 | 설명 | 적합한 경우 |
| --- | --- | --- |
| Copilot CLI 직접 호출 | `.github/agents`, `.github/skills`를 CLI가 로드하도록 구성하고, 필요 시 `--agent`나 skill 이름을 명시해 실행 | 제한망 runner, shell 기반 자동화, 명시적 프롬프트 실행 |
| GHCP/VS Code agent mode | instructions, prompts, custom agent 파일을 통해 Copilot Chat 또는 agent mode가 참고하도록 구성 | 개발자 IDE 안에서 반복 작업, 역할별 리뷰, 코드 수정 보조 |

<br/>GHCP/VS Code 방식에서는 저장소 공통 지침, 파일별 지침, 반복 prompt를 함께 활용할 수 있습니다. 예를 들어 저장소 전체에 적용할 공통 지침은 다음처럼 작성할 수 있습니다.

```markdown
# Repository Instructions

- 보안 관련 변경은 인증, 권한, secret 노출 여부를 우선 검토합니다.
- 테스트가 필요한 변경에는 최소 하나 이상의 검증 방법을 제안합니다.
- 자동 수정 전에는 변경 의도와 영향 범위를 먼저 설명합니다.
- 제한된 네트워크 환경에서는 외부 패키지 설치나 원격 API 호출을 전제로 제안하지 않습니다.
```

보안 리뷰용 instruction은 다음처럼 분리할 수 있습니다.

```markdown
---
description: "Use when: reviewing security-sensitive code changes, authentication, authorization, secrets, network access, or infrastructure changes."
applyTo: "**/*"
---

# Security Review Instructions

When reviewing changes, check:

- Authentication and authorization changes
- Secret, token, key, or connection string exposure
- Input validation and output encoding
- Logging of sensitive data
- Network exposure and firewall or private endpoint changes
- Missing security tests
```

반복 실행할 prompt는 다음처럼 분리할 수 있습니다.

```markdown
---
description: "Review the current pull request diff and produce a Korean reviewer checklist."
---

# PR Review Prompt

현재 변경사항을 기준으로 다음 항목을 한국어로 정리하세요.

1. 변경 요약
2. 위험 가능성이 있는 부분
3. 추가 테스트가 필요한 부분
4. 리뷰어가 확인해야 할 체크리스트
```

Copilot CLI에서는 custom agent를 `/agent`에서 선택하거나 `--agent <agent-id>`로 지정할 수 있고, skill은 `/skills`, `/SKILL-NAME`, `copilot skill` 명령을 통해 확인하거나 사용할 수 있습니다. headless 자동화에서는 어떤 agent 또는 skill이 선택되었는지 명확해야 하므로, agent는 `--agent security-review`처럼 명시적으로 지정하고, skill은 `/SKILL-NAME` 또는 프롬프트에서 skill 이름과 작업 범위를 명확히 지정하는 것이 재현성과 감사 측면에서 안전합니다.

> **제한된 네트워크 환경 확인 사항** GHCP/VS Code agent mode는 GitHub Copilot 서비스와 IDE 확장 기능에 의존합니다. 제한된 네트워크 환경에서 사용하려면 GitHub Copilot 서비스, 인증 경로, VS Code 확장 업데이트 경로, 프록시 및 인증서 구성이 허용되어야 합니다. 반면 Copilot CLI를 `COPILOT_OFFLINE=true`와 provider 환경변수로 실행하는 방식은 조직이 허용한 Microsoft Foundry 또는 Azure OpenAI 엔드포인트를 직접 호출하는 구조로 설계할 수 있습니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">Agent 정의 예시

Agent 파일은 “이 AI가 어떤 역할을 수행할 것인가”를 명확히 정의합니다. 예를 들어 `.github/agents/security-review.agent.md`는 다음과 같이 작성할 수 있습니다.

```markdown
---
name: security-review
description: "Use when: performing security review for code, infrastructure, authentication, authorization, or secret handling changes."
---

# Security Review Agent

## Role
You are a security review agent for an enterprise development team.

## Goal
Review code changes and identify security risks, sensitive data exposure,
authentication and authorization issues, insecure network configuration,
and missing validation.

## Boundaries
- Do not modify files directly.
- Do not suggest bypassing security controls.
- Do not expose secrets or token values.
- Focus only on the provided diff and context files.

## Output Format
Return the result in Korean with the following sections:

1. Summary
2. High Risk Findings
3. Medium/Low Risk Findings
4. Missing Tests
5. Recommended Next Actions
```

Agent 정의에서 가장 중요한 부분은 `Boundaries`입니다. 제한된 네트워크 환경에서 운영하는 AI 자동화는 모델 출력이 곧바로 코드 변경이나 배포로 이어지지 않도록 경계를 명확히 해야 합니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">Skill 정의 예시

Skill은 특정 작업을 반복 가능하게 만드는 절차입니다. 예를 들어 `.github/skills/summarize-diff/SKILL.md`는 다음과 같이 정의할 수 있습니다.

```markdown
---
name: summarize-diff
description: "Use when: summarizing pull request diffs, changed files, risk areas, and suggested tests."
---

# SKILL.md

## Purpose
Summarize code changes from a pull request diff and identify review focus areas.

## Inputs
- `change-summary.txt`: git diff stat output
- `change.diff`: full git diff output
- `repository-context.md`: optional repository context

## Procedure
1. Read the change summary.
2. Identify changed components and likely ownership areas.
3. Review the full diff for behavioral changes.
4. Identify risky changes and missing tests.
5. Produce a concise Korean summary for reviewers.

## Output
- Change summary
- Risk areas
- Suggested tests
- Reviewer checklist
```

Skill은 agent보다 더 구체적인 작업 절차입니다. 같은 보안 리뷰 에이전트가 `summarize-diff`, `review-infra-change`, `analyze-test-log` 같은 여러 skill을 사용할 수 있습니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">Agentic Workflow 패턴

Agent와 Skill을 조합하면 다음과 같은 agentic workflow를 구성할 수 있습니다.

| Workflow | Agent | Skill | Trigger | 결과물 |
| --- | --- | --- | --- | --- |
| PR 변경 영향 분석 | Code Review Agent | `summarize-diff` | `pull_request` | 리뷰 요약, 테스트 제안 |
| 테스트 실패 분석 | Test Analysis Agent | `analyze-test-log` | `workflow_run` 실패 | 실패 원인 후보, 재현 방법 |
| 보안 리뷰 | Security Review Agent | `review-security-diff` | `pull_request` 또는 수동 실행 | 위험 항목, 보완 권고 |
| IaC 변경 검토 | Infra Review Agent | `review-infra-change` | IaC 파일 변경 | 네트워크/권한/비용 영향 분석 |
| 릴리즈 노트 초안 | Release Note Agent | `draft-release-note` | tag 또는 수동 실행 | 릴리즈 노트 초안 |

이 방식의 장점은 AI 작업을 “프롬프트 한 번”으로 끝내지 않고, 조직이 검토 가능한 작업 단위로 나눌 수 있다는 점입니다. 어떤 agent가 어떤 skill로 어떤 입력을 처리했는지 남기면 감사와 재현이 쉬워집니다.

> **제한된 네트워크 환경 확인 사항** `pull_request`, `workflow_dispatch`, `workflow_run`, tag 기반 trigger는 GHES 또는 GitHub Actions가 내부 환경에서 정상 동작한다는 전제가 필요합니다. 외부 GitHub.com 이벤트를 받는 구조가 아니더라도, GHES Actions 기능과 self-hosted runner 연결, webhook 처리, runner label 매칭이 사전에 구성되어 있어야 합니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">개발자 환경에서 먼저 구성해보기

GitHub Actions runner에 올리기 전에는 개발자 로컬 환경에서 agent와 skill이 의도대로 동작하는지 먼저 검증하는 것이 좋습니다. 로컬에서 검증된 agent와 skill을 runner workflow로 옮기면, 네트워크 문제와 프롬프트 설계 문제를 분리해서 확인할 수 있습니다.

로컬 개발환경에서는 두 가지 방식으로 검증할 수 있습니다.

| 방식 | 사용 도구 | 특징 |
| --- | --- | --- |
| Copilot CLI 방식 | `copilot` 명령어 | `.github/agents`, `.github/skills`를 CLI가 로드하도록 두고 `--agent` 또는 skill 이름을 명시해 실행합니다. runner 자동화와 같은 실행 방식을 미리 검증하기 좋습니다. |
| GHCP/VS Code 방식 | Copilot Chat, agent mode, prompts | `.github/instructions`, `.github/prompts`, `.github/agents`, `.github/skills` 파일을 활용해 IDE 안에서 반복 작업을 수행합니다. 개발자 경험 검증에 적합합니다. |

아래 예시는 runner 자동화와 같은 형태로 검증하기 위한 Copilot CLI 방식입니다.

먼저 저장소 루트에서 권장 디렉터리 구조를 생성합니다.

```bash
mkdir -p .github/agents .github/skills/summarize-diff .github/harness/outputs
```

그 다음 앞에서 정의한 agent와 skill을 파일로 저장합니다. 예를 들어 보안 리뷰를 먼저 검증하려면 다음 두 파일을 준비합니다.

```text
.github/agents/security-review.agent.md
.github/skills/summarize-diff/SKILL.md
```

이 경로는 GHCP와 Copilot CLI가 모두 사용할 수 있는 구조입니다. Copilot CLI에서는 custom agent를 `--agent security-review`처럼 명시적으로 지정할 수 있고, skill은 `/skills` 또는 `/summarize-diff` 같은 방식으로 확인하거나 호출할 수 있습니다. 다만 자동화에서는 재현성을 위해 agent와 작업 범위를 명시하는 것이 좋습니다.

Copilot CLI 환경변수는 개발자 터미널 세션에 직접 설정합니다. Microsoft Foundry 또는 Azure OpenAI 모델 호출에 필요한 `COPILOT_PROVIDER_*` 환경변수와 인증 방식은 [제한된 네트워크 환경에서 구성하는 AI 기반 개발환경](/ai-development-environment-in-priave-network/)의 `GitHub Copilot CLI 실행을 위한 환경변수 구성` 섹션을 참고하면 됩니다.

환경변수 설정이 끝나면 현재 작업 브랜치의 변경 내용을 context 파일로 만듭니다.

```bash
git diff --stat > .github/harness/outputs/change-summary.txt
git diff > .github/harness/outputs/change.diff

{
  echo "# Change Summary"
  cat .github/harness/outputs/change-summary.txt
  echo ""
  echo "# Diff"
  cat .github/harness/outputs/change.diff
} > .github/harness/outputs/context.md
```

이제 custom agent를 명시해 실행합니다. skill을 반드시 특정하고 싶다면 프롬프트 안에 사용할 skill 이름과 입력 파일을 함께 명시합니다.

```bash
copilot \
  --agent security-review \
  --add-dir ./.github/agents \
  --add-dir ./.github/skills \
  --add-dir ./.github/harness \
  -p "Use .github/agents/security-review.agent.md and .github/skills/summarize-diff/SKILL.md. Review .github/harness/outputs/context.md and produce a Korean security review checklist." \
  > .github/harness/outputs/agent-output.md
```

CLI 버전, 실행 모드, 조직 정책에 따라 skill 자동 선택을 더 명확히 통제해야 한다면, 기존 방식처럼 agent, skill, context 내용을 하나의 프롬프트 파일로 조합해 전달할 수도 있습니다.

```bash
{
  cat .github/agents/security-review.agent.md
  echo ""
  echo "--- Skill ---"
  cat .github/skills/summarize-diff/SKILL.md
  echo ""
  echo "--- Context ---"
  cat .github/harness/outputs/context.md
} > .github/harness/outputs/agent-prompt.md

copilot -p "$(cat .github/harness/outputs/agent-prompt.md)" > .github/harness/outputs/agent-output.md
```

결과는 바로 커밋하거나 PR에 반영하지 말고 먼저 사람이 확인합니다.

```bash
less .github/harness/outputs/agent-output.md
```

로컬 검증 시에는 다음 항목을 확인합니다.

- Agent와 Skill 정의가 너무 넓거나 모호하지 않은지
- 모델에 전달되는 context에 secret, token, 개인정보가 포함되지 않는지
- 출력이 기대한 형식으로 생성되는지
- 모델 응답이 실제 코드 변경과 맞지 않는 부분을 포함하지 않는지
- 반복 실행 시 결과가 리뷰 가능한 수준으로 일관적인지

간단한 출력 검증도 로컬에서 바로 수행할 수 있습니다. 예를 들어 필수 섹션이 누락되면 실패하도록 다음 명령을 실행합니다.

```bash
grep -q "Summary" .github/harness/outputs/agent-output.md
grep -q "Recommended Next Actions" .github/harness/outputs/agent-output.md
```

이 과정을 통과하면 같은 agent와 skill을 GitHub Actions runner에서 호출하도록 옮길 수 있습니다. 개발자 로컬에서는 prompt 품질과 출력 형식을 검증하고, runner에서는 trigger, secret 주입, artifact 저장, 승인 절차를 검증하는 식으로 역할을 나누는 것이 좋습니다.

> **제한된 네트워크 환경 확인 사항** 개발자 로컬 환경에서도 `copilot` 명령이 Microsoft Foundry 또는 Azure OpenAI 엔드포인트에 접근할 수 있어야 합니다. 사내 프록시, DNS, 인증 엔드포인트, 인증서 체인, API Key 또는 Bearer Token 발급 경로가 runner와 다를 수 있으므로, 로컬 검증 결과가 곧바로 runner 동작을 보장하지는 않습니다.

GHCP/VS Code 방식으로 검증하는 경우에는 `copilot` 명령어를 직접 실행하기보다 Copilot Chat 또는 agent mode에서 앞서 정의한 prompt, custom agent, skill을 호출합니다. 이 경우 결과는 IDE 안에서 확인하고, 실제 runner 자동화로 옮길 때는 동일한 작업 의도를 CLI의 `--agent`, skill 이름 또는 workflow step으로 다시 명시해야 합니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">GitHub Actions Runner에서 실행하는 예시

다음 예시는 self-hosted runner에서 diff를 수집하고, agent와 skill 정의를 조합해 Copilot CLI를 실행하는 workflow입니다. Copilot CLI 환경변수와 인증 방식은 [제한된 네트워크 환경에서 구성하는 AI 기반 개발환경](/ai-development-environment-in-priave-network/)을 참고하면 됩니다.

> **제한된 네트워크 환경 확인 사항** 아래 예시의 `actions/checkout@v4`와 `actions/upload-artifact@v4`는 GHES 또는 내부 네트워크에서 사용할 수 있도록 사전에 동기화되거나 허용되어야 합니다. 외부 GitHub Actions marketplace 접근이 차단된 환경이라면 GHES에 번들된 액션, 내부 mirror 또는 shell 기반 대체 절차를 사용해야 합니다.

{% raw %}
```yaml
name: Agentic PR Review

on:
  pull_request:
    types: [opened, synchronize, reopened]
  workflow_dispatch:

jobs:
  agentic-review:
    runs-on: [self-hosted, copilot-cli]

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure Copilot CLI provider
        shell: bash
        env:
          COPILOT_PROVIDER_TYPE: azure
          COPILOT_PROVIDER_BASE_URL: ${{ secrets.COPILOT_PROVIDER_BASE_URL }}
          COPILOT_OFFLINE: "true"
          COPILOT_MODEL: ${{ secrets.COPILOT_MODEL }}
          COPILOT_PROVIDER_AZURE_API_VERSION: ${{ secrets.COPILOT_PROVIDER_AZURE_API_VERSION }}
          COPILOT_PROVIDER_WIRE_API: responses
          COPILOT_PROVIDER_API_KEY: ${{ secrets.COPILOT_PROVIDER_API_KEY }}
        run: |
          {
            echo "COPILOT_PROVIDER_TYPE=$COPILOT_PROVIDER_TYPE"
            echo "COPILOT_PROVIDER_BASE_URL=$COPILOT_PROVIDER_BASE_URL"
            echo "COPILOT_OFFLINE=$COPILOT_OFFLINE"
            echo "COPILOT_MODEL=$COPILOT_MODEL"
            echo "COPILOT_PROVIDER_AZURE_API_VERSION=$COPILOT_PROVIDER_AZURE_API_VERSION"
            echo "COPILOT_PROVIDER_WIRE_API=$COPILOT_PROVIDER_WIRE_API"
            echo "COPILOT_PROVIDER_API_KEY=$COPILOT_PROVIDER_API_KEY"
          } >> "$GITHUB_ENV"

      - name: Collect pull request context
        shell: bash
        run: |
          git fetch --no-tags --depth=50 origin "$GITHUB_BASE_REF" || true
          git diff --stat "origin/$GITHUB_BASE_REF"...HEAD > change-summary.txt || git diff --stat > change-summary.txt
          git diff "origin/$GITHUB_BASE_REF"...HEAD > change.diff || git diff > change.diff

      - name: Build context file
        shell: bash
        run: |
          {
            echo "# Change Summary"
            cat change-summary.txt
            echo ""
            echo "# Diff"
            cat change.diff
          } > context.md

      - name: Run security review agent
        shell: bash
        run: |
          copilot \
            --agent security-review \
            --add-dir ./.github/agents \
            --add-dir ./.github/skills \
            --add-dir ./.github/harness \
            -p "Use .github/agents/security-review.agent.md and .github/skills/summarize-diff/SKILL.md. Review context.md and produce a Korean security review checklist." \
            > agent-output.md

      - name: Add agent output to summary
        shell: bash
        run: cat agent-output.md >> "$GITHUB_STEP_SUMMARY"

      - name: Upload agent artifacts
        uses: actions/upload-artifact@v4
        with:
          name: agentic-review-output
          path: |
            context.md
            agent-output.md
```
{% endraw %}

이 workflow는 agent의 결과를 바로 PR에 반영하지 않고 job summary와 artifact로 남깁니다. PR comment를 자동 작성하거나 코드 변경 PR을 생성하려면 별도 승인 조건과 권한 정책을 먼저 설계하는 것이 좋습니다.

> **제한된 네트워크 환경 확인 사항** `GITHUB_STEP_SUMMARY`는 runner 로컬 job summary 기능이므로 Actions가 정상 동작하면 사용할 수 있습니다. 반면 artifact 업로드, PR comment, issue comment, 자동 PR 생성은 GHES API 접근 권한과 `GITHUB_TOKEN`, `GH_TOKEN` 또는 GitHub App token 구성이 필요합니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">Agent와 Skill 설계 원칙

Agentic workflow를 운영 가능한 수준으로 만들려면 다음 원칙을 지키는 것이 좋습니다.

- **작업 범위를 좁게 정의**: “코드 전체를 개선해줘”보다 “이 diff에서 보안 리스크와 누락 테스트를 찾아줘”가 안전합니다.
- **입력 컨텍스트를 명시**: 어떤 파일, diff, 로그를 전달하는지 실행 절차나 workflow step에서 통제합니다.
- **출력 형식을 고정**: summary, findings, risks, tests, next actions 같은 섹션을 정해 검토자가 빠르게 확인할 수 있게 합니다.
- **민감정보를 제거**: token, connection string, 개인 정보, 내부 시스템 주소는 모델 호출 전에 마스킹합니다.
- **자동 반영을 제한**: agent 결과는 제안으로 남기고, 코드 변경과 배포는 사람의 승인이나 별도 정책을 거치게 합니다.
- **재현 가능성을 확보**: agent 파일, skill 파일, context 파일, output 파일을 artifact로 남겨 같은 입력에서 어떤 결과가 나왔는지 추적합니다.
- **평가 기준을 추가**: hallucination, 정책 위반, 출력 형식 오류, 민감정보 포함 여부를 검사하는 validation step을 둡니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">추가로 시도해볼 수 있는 Agent 활용 아이디어

다음과 같은 agent를 추가하면 단순 코드 생성보다 더 실무적인 자동화 흐름을 만들 수 있습니다.

### 테스트 실패 분석 Agent

테스트 로그와 변경 diff를 입력으로 받아 실패 원인 후보, 재현 방법, 추가 확인 명령을 정리합니다. CI 실패 시 개발자가 로그를 처음부터 읽는 시간을 줄이는 데 유용합니다.

### 배포 변경 검토 Agent

Bicep, Terraform, Helm, GitHub Actions workflow 변경을 분석해 권한 증가, 네트워크 노출, 비용 증가 가능성을 요약합니다. 운영 배포 전 리뷰 체크리스트를 자동 생성하는 용도로 사용할 수 있습니다.

> **제한된 네트워크 환경 확인 사항** Terraform provider 다운로드, Helm chart 원격 조회, Azure Resource Manager 조회 같은 동작은 외부 또는 Azure 엔드포인트 접근이 필요할 수 있습니다. 제한망에서는 필요한 provider, module, chart를 내부 registry나 artifact repository에 mirror하는 방식이 필요합니다.

### 보안 리뷰 Agent

인증/인가, secret 노출, 입력 검증, 민감정보 로깅, 네트워크 보안 설정을 기준으로 diff를 검토합니다. 보안팀 리뷰 전 1차 점검 용도로 적합합니다.

### 마이그레이션 보조 Agent

프레임워크 버전 업그레이드나 SDK 교체 시 변경 범위, 수정 후보, 테스트 전략을 제안합니다. 자동 수정보다는 계획 수립과 체크리스트 생성에 먼저 사용하는 것이 안전합니다.

### 운영 문서 생성 Agent

PR diff, 배포 스크립트, 설정 파일을 기반으로 운영자용 변경 요약, rollback 절차, 릴리즈 노트 초안을 생성합니다. 단, 문서는 반드시 담당자가 검토한 뒤 게시해야 합니다.

> **제한된 네트워크 환경 확인 사항** 릴리즈 노트나 운영 문서를 외부 wiki, issue tracker, documentation portal에 자동 게시하려면 해당 시스템의 내부 접근 경로와 API token이 필요합니다. 네트워크가 차단된 환경에서는 artifact로 생성한 뒤 담당자가 내부 문서 시스템에 반영하는 방식이 더 현실적입니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">마치며

제한된 네트워크 환경에서 Copilot CLI를 사용하는 것만으로는 agentic workflow가 완성되지 않습니다. 운영 가능한 AI 개발 자동화를 만들려면 agent와 skill을 명확히 정의하고, harness engineering 관점에서 각 단계의 입력과 출력을 통제해야 합니다.

정리하면 다음과 같습니다.

1. Agent는 역할과 경계를 정의합니다.
2. Skill은 반복 가능한 작업 절차를 정의합니다.
3. Harness engineering은 컨텍스트 수집, 모델 호출, 출력 검증, artifact 저장 같은 실행 안전장치를 설계하는 관점입니다.
4. GitHub Actions runner와 결합하면 PR 분석, 테스트 실패 분석, 보안 점검, 문서 생성 같은 agentic workflow를 구성할 수 있습니다.
5. 금융권과 엔터프라이즈 환경에서는 자동 반영보다 검토 가능한 제안과 승인 흐름을 우선 설계해야 합니다.

이 방식은 managed cloud agent를 그대로 대체하지는 않지만, 내부망과 보안 정책을 유지하면서 AI 기반 개발 자동화를 점진적으로 확장하는 현실적인 방법이 될 수 있습니다.

### 참고자료

- [GitHub Copilot CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference)
- [Creating and using custom agents for GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-custom-agents-for-cli)
- [Adding agent skills for GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-skills)