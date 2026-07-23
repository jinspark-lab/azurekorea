---
layout: post
title: "제한된 네트워크 환경에서 구성하는 AI 기반 개발환경"
author: haewonshin
tag: [ GitHub Copilot, Copilot CLI, Codex CLI, Foundry Model]
category: [ Solution ]
image: assets/images/thumnails/GitHub_Copilot_CLI_Icon.png
featured: true
---

최근 금융권 등 폐쇄망 또는 제한된 네트워크 환경에서 AI 기반 코드 어시스턴트를 활용하고자 하는 니즈가 증가하고 있습니다. 특히 개발자가 기존 터미널과 IDE 환경을 유지하면서도, 조직이 통제하는 모델 엔드포인트와 네트워크 정책 안에서 코드 생성, 명령어 작성, 오류 분석을 수행할 수 있는 구성이 중요해지고 있습니다.

본 포스팅은 제한된 네트워크 환경에서의 AI 개발 자동화 구성을 다루는 시리즈 중 하나입니다. 아래 시리즈 목차를 통해 각 글을 순서대로 확인할 수 있습니다.

### 시리즈 목차

1. **제한된 네트워크 환경에서 구성하는 AI 기반 개발환경**
2. [제한된 네트워크 환경에서 Agent와 Skill로 확장하는 AI 개발 자동화](/agentic-workflow-harness-engineering-with-copilot-agents-skills/)
3. [GitHub Enterprise Server에서 자율형 에이전트 구현](/github-enterprise-server-runner-copilot-cli-cloud-agent/)

이 포스팅에서는 폐쇄망 환경에서 Microsoft Foundry Model을 GitHub Copilot CLI와 Codex CLI에서 호출하는 구성을 설명합니다. 또한 두 CLI 간 방식을 비교하여, 엔터프라이즈와 금융권 고객이 어떤 기준으로 CLI 기반 AI 개발 환경을 선택할 수 있는지 정리합니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">`copilot` 명령어로 제한된 네트워크 환경에서 개발 환경 구성

제한된 네트워크 환경에서 AI 기반 코드 어시스턴트를 사용하려면 단순히 CLI 도구를 설치하는 것만으로는 충분하지 않습니다. 고객 환경에서는 네트워크 경로, 인증 방식, 모델이 배포된 리전, 프록시 구성, 데이터 처리 기준을 함께 설계해야 합니다.

GitHub Copilot CLI는 개발자가 터미널에서 `copilot` 명령어를 호출해 자연어로 명령을 요청하고, 코드 작성이나 명령어 생성, 오류 분석, 반복 작업 자동화 등을 수행할 수 있도록 돕습니다. 폐쇄망 또는 제한된 네트워크 환경에서는 다음과 같은 흐름으로 구성을 검토할 수 있습니다.

1. 개발자 단말 또는 개발 서버에서 `copilot` 명령어를 사용할 수 있도록 GitHub Copilot CLI를 설치합니다. GitHub Copilot CLI는 GHCP(GitHub Copilot) 라이센스와는 별도로 무료로 설치할 수 있습니다.
2. Azure 환경이 준비되어 있는 경우 Microsoft Foundry 모델을 배포하고, PTU 기반 모델 또는 한국 리전의 종량제 모델을 검토합니다. 제한된 네트워크 환경에서 Azure를 사용하는 국내 엔터프라이즈의 경우 대부분은 PTU(Provisioned Throughput Unit) 배포 모델을 선택합니다. 고객이 자체 모델 엔드포인트를 활용해야 하는 경우, 해당 엔드포인트와 API KEY 값을 이용해서 GitHub Copilot CLI에서 모델을 호출할 수 있습니다.

    <details markdown="1">
    <summary><strong>Microsoft Foundry와 PTU 모델 활용</strong></summary>

    금융권과 엔터프라이즈 고객은 예측 가능한 성능, 안정적인 처리량, 네트워크 통제, 감사 가능성을 중요하게 봅니다. 이 경우 Microsoft Foundry에서 모델을 배포하고, PTU 기반 구성을 검토할 수 있습니다.

    PTU는 일정한 처리량을 예약하여 모델 호출 성능을 안정적으로 확보하는 방식입니다. 개발자들이 CLI 기반 AI 기능을 반복적으로 사용하거나, 다수의 팀이 공통 개발 자동화 환경을 사용할 경우 PTU 기반 모델은 성능 예측과 운영 관리 측면에서 장점이 있습니다.

    구성 검토 시에는 다음 항목을 함께 확인해야 합니다.

    - 모델 배포 리전과 데이터 처리 위치
    - PTU 또는 종량제 모델 선택 기준
    - 인증 방식과 권한 관리
    - 프록시, 방화벽, Private Endpoint 등 네트워크 경로
    - 프롬프트와 응답 데이터의 저장 여부 및 보존 정책
    - 개발자 단말, IDE, CLI, GitHub 저장소와의 연계 방식

    즉, CLI 선택보다 중요한 것은 조직이 통제하는 모델, 인증 방식, 네트워크 경로, 보안 정책 안에서 개발자 경험을 안정적으로 제공할 수 있는지입니다.
    </details>

3. 개발자가 사용하는 VS Code, 터미널, GitHub 저장소, CI/CD 파이프라인과 연계하여 실제 개발 워크플로우에서 사용할 수 있는지 검증합니다.

### GitHub Copilot CLI 실행을 위한 환경변수 구성

GitHub Copilot CLI에서 Microsoft Foundry에 배포된 Azure OpenAI 모델을 직접 호출하려면, 먼저 Foundry 모델 엔드포인트와 인증 정보를 환경변수로 구성해야 합니다. 

#### 공통 환경변수 

아래 환경변수는 터미널 세션이나 셸 프로파일, 보안이 적용된 환경변수 주입 방식으로 직접 설정합니다. 인증 방식과 무관하게 공통으로 필요한 값은 다음과 같습니다. 특히, 제한된 네트워크 환경이므로 `COPILOT_OFFLINE` 값은 반드시 `true`가 되어야 합니다.

| 환경변수 | 필수 여부 | 변수 값 포맷 | 설명 |
| --- | --- | --- | --- |
| `AZURE_OPEN_AI_ENDPOINT` | 필수 | `https://<resource-name>.cognitiveservices.azure.com/openai/responses?api-version=<api-version>` | Microsoft Foundry 또는 Azure OpenAI 모델 호출 엔드포인트입니다. 이 값에서 `/openai/...` 경로와 query string을 제외한 값이 `COPILOT_PROVIDER_BASE_URL`이 되며, `api-version` query parameter 값이 `COPILOT_PROVIDER_AZURE_API_VERSION`이 됩니다. |
| `AZURE_OPEN_AI_MODEL` | 선택 | `<deployment-name>` 또는 `<model-name>` | Copilot CLI가 호출할 모델 또는 배포명입니다. 직접 환경변수를 설정할 때는 이 값과 동일하게 `COPILOT_MODEL`도 설정하는 것을 권장합니다. |
| `COPILOT_PROVIDER_WIRE_API` | 선택 | `responses` 또는 `completions` | Copilot CLI가 사용할 wire API 형식을 지정합니다. `/openai/responses` 엔드포인트나 Codex 계열 모델을 사용할 때는 일반적으로 `responses`를 사용합니다. |
| `COPILOT_PROVIDER_TYPE` | 필수 | `azure` | Provider 유형입니다. Microsoft Foundry 또는 Azure OpenAI 모델을 호출하므로 `azure`로 고정합니다. |
| `COPILOT_PROVIDER_BASE_URL` | 필수 | `https://<resource-name>.cognitiveservices.azure.com` | `AZURE_OPEN_AI_ENDPOINT`에서 `/openai/...` 경로와 query string을 제거한 base URL입니다. |
| `COPILOT_OFFLINE` | 필수 | `true` | GitHub 서버를 경유하지 않고 지정한 provider endpoint로 직접 호출하도록 설정합니다. |
| `COPILOT_PROVIDER_AZURE_API_VERSION` | 선택 | `YYYY-MM-DD` 또는 `YYYY-MM-DD-preview` | `AZURE_OPEN_AI_ENDPOINT`의 `api-version` query parameter에서 추출한 Azure OpenAI API 버전입니다. 예: `2025-03-01-preview`. |
| `COPILOT_MODEL` | 선택 | `<deployment-name>` 또는 `<model-name>` | Copilot CLI가 실제로 사용할 모델 또는 배포명입니다. `AZURE_OPEN_AI_MODEL`과 같은 값으로 맞춰 설정하는 것을 권장합니다. |

<br/>

#### 인증 방식 별 필요 환경변수

Microsoft Foundry의 모델 엔드포인트 인증 방법은 다음 두 가지를 지원합니다.

- Azure CLI 로그인을 통해 발급한 Microsoft Entra ID Bearer Token 인증
- 키 기반 인증 - BYOK(Bring Your Own Key)

##### 인증 방식 1: Microsoft Entra ID Bearer Token

Microsoft Entra ID 인증을 사용할 수 있는 환경에서는 Azure CLI 로그인 후 Bearer Token을 발급받아 Copilot CLI에 전달합니다. API Key를 `.env`에 저장하지 않아도 되므로, 운영 환경에서는 이 방식을 우선 검토하는 것이 좋습니다.

| 환경변수 | 필수 여부 | 변수 값 포맷 | 설명 |
| --- | --- | --- | --- |
| `COPILOT_PROVIDER_BEARER_TOKEN` | 필수 | `<entra-id-access-token>` | `az account get-access-token --resource https://cognitiveservices.azure.com`으로 발급한 Microsoft Entra ID Bearer Token입니다. Copilot CLI가 Azure OpenAI 또는 Foundry 엔드포인트를 호출할 때 사용합니다. |
| `BEARER_TOKEN` | 선택 | `<entra-id-access-token>` | `COPILOT_PROVIDER_BEARER_TOKEN`과 동일한 토큰입니다. 다른 도구나 하위 프로세스에서 공통 Bearer Token 변수로 참조해야 할 때만 함께 설정합니다. |
| `AZURE_OPEN_AI_KEY` | 불필요 | 없음 | Bearer Token 인증을 사용할 때는 API Key가 필요하지 않습니다. |
| `COPILOT_PROVIDER_API_KEY` | 불필요 | 없음 | Bearer Token이 정상 발급된 경우 설정하지 않습니다. |

예시는 다음과 같습니다. 토큰 값이 필요하므로 먼저 az login 명령어를 수행해서 azure 환경에 로그인합니다.
```bash
az login

export AZURE_OPEN_AI_ENDPOINT="https://<resource-name>.cognitiveservices.azure.com/openai/responses?api-version=2025-03-01-preview"
export AZURE_OPEN_AI_MODEL="<deployment-name>"
export COPILOT_PROVIDER_TYPE="azure"
export COPILOT_PROVIDER_BASE_URL="https://<resource-name>.cognitiveservices.azure.com"
export COPILOT_OFFLINE="true"
export COPILOT_PROVIDER_AZURE_API_VERSION="2025-03-01-preview"
export COPILOT_MODEL="<deployment-name>"
export COPILOT_PROVIDER_WIRE_API="responses"
export COPILOT_PROVIDER_BEARER_TOKEN="$(az account get-access-token --resource https://cognitiveservices.azure.com --query accessToken -o tsv)"
export BEARER_TOKEN="$COPILOT_PROVIDER_BEARER_TOKEN"
```

##### 인증 방식 2: API Key 기반 BYOK

Azure CLI 로그인을 사용할 수 없거나 서비스 계정 기반으로 단순한 키 인증을 사용해야 하는 경우에는 API Key를 Copilot CLI provider key로 전달합니다. 이 방식은 구현이 단순하지만, `.env` 파일과 키 저장소 관리가 중요합니다.

| 환경변수 | 필수 여부 | 변수 값 포맷 | 설명 |
| --- | --- | --- | --- |
| `AZURE_OPEN_AI_KEY` | 필수 | `<api-key>` | Microsoft Foundry 또는 Azure OpenAI 리소스의 API Key입니다. `.env` 파일에 저장할 경우 저장소에 커밋되지 않도록 관리해야 합니다. |
| `COPILOT_PROVIDER_API_KEY` | 필수 | `<api-key>` | `AZURE_OPEN_AI_KEY` 값을 기반으로 Copilot CLI에 전달되는 provider API Key입니다. |
| `COPILOT_PROVIDER_BEARER_TOKEN` | 불필요 | 없음 | API Key 인증에서는 Bearer Token을 사용하지 않습니다. |
| `BEARER_TOKEN` | 불필요 | 없음 | API Key 인증에서는 설정하지 않습니다. |

예시는 다음과 같습니다.

```bash
export AZURE_OPEN_AI_ENDPOINT="https://<resource-name>.cognitiveservices.azure.com/openai/responses?api-version=2025-03-01-preview"
export AZURE_OPEN_AI_MODEL="<deployment-name>"
export AZURE_OPEN_AI_KEY="<api-key>"
export COPILOT_PROVIDER_TYPE="azure"
export COPILOT_PROVIDER_BASE_URL="https://<resource-name>.cognitiveservices.azure.com"
export COPILOT_OFFLINE="true"
export COPILOT_PROVIDER_AZURE_API_VERSION="2025-03-01-preview"
export COPILOT_MODEL="<deployment-name>"
export COPILOT_PROVIDER_WIRE_API="responses"
export COPILOT_PROVIDER_API_KEY="$AZURE_OPEN_AI_KEY"
```

환경변수 설정 후에는 동일한 터미널 세션에서 `copilot` 명령어를 실행합니다. 실제 실행은 아래처럼 직접 설정한 환경변수를 기반으로 수행합니다.
```bash
copilot ask "이 코드의 보안 취약점을 검토해줘"
```

운영 환경에서는 `.env` 파일과 API Key가 저장소에 커밋되지 않도록 관리하고, 가능하면 Entra ID 기반 인증과 Key Vault 같은 보안 저장소를 함께 사용하는 것이 좋습니다. 또한 프록시나 방화벽을 사용하는 제한된 네트워크 환경에서는 Copilot CLI가 접근하는 Foundry 엔드포인트, 인증 엔드포인트, DNS 해석 경로를 사전에 허용 목록으로 정리해야 합니다.

예를 들어 개발자는 터미널에서 다음과 같이 `copilot` 명령어를 사용해 코드 생성과 명령어 설명을 요청할 수 있습니다. 실제 옵션명은 고객이 사용하는 Copilot CLI 버전과 배포 방식에 따라 달라질 수 있으므로, PoC 단계에서는 `copilot --help`로 사용 가능한 명령을 먼저 확인하는 것이 좋습니다.

```bash
copilot --help
copilot "Azure 리소스 그룹에서 실패한 배포 목록을 조회하는 Azure CLI 명령어를 작성해줘"
```

PTU 또는 한국 리전 모델을 별도 배포해 사용하는 시나리오에서는 `copilot` 명령어가 호출할 모델 프로필이나 엔드포인트 정책을 조직 기준에 맞게 지정할 수 있어야 합니다. 예를 들어 개발자는 기본 Copilot 경험은 유지하되, 조직에서 허용한 모델 프로필을 사용하도록 다음과 같은 형태의 구성을 검토할 수 있습니다.

```bash
copilot --model "<approved-model-profile>" "이 코드의 예외 처리 흐름을 개선해줘"
copilot --model "<approved-model-profile>" "./scripts/deploy.sh"
```

실제 폐쇄망 환경에서는 위 명령 자체보다, 이 명령이 통과하는 인증, 프록시, 모델 호출 경로, 로그 처리 정책이 더 중요합니다. 따라서 PoC 단계에서는 개발자 경험과 함께 네트워크 및 보안팀이 확인할 수 있는 연결 경로, 데이터 처리 위치, 감사 기준을 문서화하는 것이 좋습니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">Codex CLI로 모델 호출하기

일부 고객은 이미 Codex CLI와 같은 터미널 기반 AI 도구를 사용하고 있거나, 특정 모델 엔드포인트를 직접 호출하는 방식을 선호할 수 있습니다. 이 경우 Codex CLI를 활용해 Microsoft Foundry 엔드포인트를 호출하는 구성을 검토할 수 있습니다.

Codex CLI 방식에서 중요한 점은 Codex 모델 자체를 쓰는지 여부가 아니라, CLI가 어떤 모델 엔드포인트를 호출할 수 있고 고객의 인증 및 네트워크 정책에 맞게 구성할 수 있는지입니다. 일반적인 검토 흐름은 다음과 같습니다.

1. Codex CLI가 사용하는 모델 공급자와 엔드포인트 설정 방식을 확인합니다.
2. Microsoft Foundry에서 사용할 모델을 배포합니다.
3. API 키, Microsoft Entra ID 인증, 프록시 등 고객 환경에 맞는 인증 및 연결 방식을 구성합니다.
4. CLI 설정 파일 또는 환경 변수를 통해 모델명, 엔드포인트, API 버전, 인증 정보를 지정합니다.
5. 실제 코드 생성, 명령어 생성, 오류 설명 시나리오에서 응답 품질과 데이터 처리 경로를 검증합니다.

환경변수는 고객 환경과 CLI 버전에 따라 달라질 수 있지만, 개념적으로는 다음과 같은 설정 흐름을 생각할 수 있습니다. Codex CLI를 사용하는 경우에는 키 기반 인증 방식만 지원합니다.

```bash
export AZURE_OPENAI_ENDPOINT="https://<resource-name>.openai.azure.com/"
export AZURE_OPENAI_API_KEY="<api-key>"
export AZURE_OPENAI_DEPLOYMENT="<deployment-name>"
export AZURE_OPENAI_API_VERSION="2024-xx-xx"

codex --model "<deployment-name>" "이 저장소 구조를 설명하고 빌드 명령어를 제안해줘"
```

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">Codex CLI와 GitHub Copilot CLI 사용 방식 비교

GitHub Copilot CLI와 Codex CLI는 모두 터미널 기반 AI 개발 경험을 제공할 수 있습니다. 코드 생성, 명령어 생성, 오류 설명 같은 기본 사용 경험만 보면 두 CLI 사이에 큰 차이는 없습니다. 따라서 엔터프라이즈 환경에서는 워크플로우 통합보다는 지원 모델, 인증 방식, 운영 표준화, 고객 보안 정책과의 적합성을 중심으로 비교하는 것이 더 현실적입니다.

| 구분 | GitHub Copilot CLI | Codex CLI|
| --- | --- | --- |
| 주요 목적 | 터미널에서 Copilot 기반 AI 개발 경험 제공 | 터미널에서 Codex 기반 AI 개발 경험 제공 |
| 제공 모델 | 단일 CLI에서 <b>멀티 모델</b> 지원 | 주로 OpenAI 계열 모델 지원 |
| 인증 방식 | API Key 기반 BYOK와 Microsoft Entra ID Bearer Token 기반 호출을 함께 검토 가능 | 현재 구성에서는 주로 API Key 기반 호출을 검토 |

<br>따라서 GitHub Copilot CLI를 선택할 때의 판단 기준은 다음과 같이 정리할 수 있습니다.

- **멀티 모델이 필요한지 확인**: 단일 CLI에서 여러 모델 선택지를 검토해야 하거나, 향후 모델 변경 및 확장 가능성을 열어두어야 한다면 GitHub Copilot CLI가 더 적합할 수 있습니다. 반대로 OpenAI 계열 모델 호출이 목적이라면 Codex CLI도 충분한 대안이 될 수 있습니다.
- **인증 방식이 핵심 판단 기준**: 금융권처럼 API Key 관리 부담을 줄이고 Microsoft Entra ID Bearer Token 기반 인증을 검토해야 하는 환경이라면 GitHub Copilot CLI를 우선 검토할 수 있습니다.
- **GitHub 플랫폼 장점은 별도 판단**: GitHub 저장소, PR, Actions, GitHub Advanced Security의 장점은 GitHub 플랫폼 사용의 장점이긴 하지만, 동일 벤더 제공이라는 점에서 GitHub Copilot CLI 사용 시 연동이 용이할 수 있습니다.

따라서 Copilot CLI와 Codex CLI를 비교할 때는 “어느 쪽이 개발 워크플로우를 더 잘 통합하는가”보다 “어느 쪽이 고객의 모델 선택, 인증 방식, 네트워크 통제, 보안 운영 기준에 더 잘 맞는가”를 중심으로 판단하는 것이 좋습니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">금융권 엔터프라이즈에서 한국 리전 모델 활용

금융권 등 일부 엔터프라이즈에서는 모델 성능보다 생성형AI에서 활용되는 데이터 처리 위치, 통제 가능성, 감사 대응이 더 중요한 판단 기준이 되는 경우가 많습니다. 특히 프롬프트에는 소스 코드뿐 아니라 개인정보, 고객 정보, 내부 시스템 구조, 장애 정보, 업무상 민감정보가 포함될 수 있기 때문에 모델이 어느 리전에서 처리되는지가 중요합니다.

혁신금융서비스 또는 내부 규제 검토를 준비하는 고객이라면 PTU 기반 모델을 우선 검토할 수 있습니다. PTU는 예측 가능한 성능과 안정적인 처리량을 제공하므로, 운영 안정성과 감사 가능성을 설명하기 좋습니다.

하지만, 금융권에서 반드시 PTU만 사용할 수 있는 것은 아닙니다. 고객의 내부 정책과 규제 검토 결과에 따라, **한국 리전에 배포된 종량제 모델**도 사용할 수 있는 선택지가 될 수 있습니다. 즉, 혁신금융서비스 신청이나 내부 심의 과정에서 중요한 것은 PTU 여부 자체가 아니라, 다음 사항을 명확히 설명할 수 있는지입니다.

- 모델이 한국 리전에서 처리되는지
- 프롬프트와 응답 데이터가 저장되는지
- 데이터 보존 기간과 처리 정책이 무엇인지
- 개인정보와 민감정보가 포함될 가능성을 어떻게 통제하는지
- 네트워크 경로와 인증 방식이 고객 보안 정책을 충족하는지
- 장애나 감사 상황에서 처리 경로를 설명할 수 있는지

따라서 모델 선택 시, “PTU 기반이면 더 안정적인 운영 모델을 설계하기 쉽다”는 점과 “한국 리전 종량제 모델도 고객의 규제와 내부 통제 요건을 충족한다면 활용 가능하다”는 점을 함께 고려하는 것이 좋겠습니다.

## <img src="../assets/images/haewonshin/github.svg" alt="GitHub" width="28" height="28" style="vertical-align:-5px;margin-right:8px;">마치며

본 포스팅에서는 다음과 같은 내용을 설명했습니다.

1. 폐쇄망 또는 제한된 네트워크 환경에서도 AI 기반 코드 어시스턴트에 대한 요구가 증가하고 있습니다.
2. GitHub Copilot CLI는 개발자가 익숙한 터미널 환경에서 AI 기반 코드 생성, 명령어 생성, 오류 분석을 사용할 수 있게 해줍니다.
3. 금융권과 엔터프라이즈 환경에서는 CLI 기능뿐 아니라 모델 리전, 네트워크 경로, 인증, 감사 가능성, 데이터 처리 정책을 함께 검토해야 합니다. 
4. Microsoft Foundry 모델을 활용하면 PTU 기반 모델 또는 한국 리전 종량제 모델을 고객 요구사항에 맞게 검토할 수 있습니다.
6. Codex CLI 기반 구성도 가능하며, GitHub Copilot CLI는 멀티 모델 선택지와 Entra ID 기반 인증 가능성을 함께 검토할 때 의미 있는 대안이 될 수 있습니다.

### 참고자료

- [GitHub Copilot CLI programmatic reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-programmatic-reference)
- [Using your own LLM models in GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/use-byok-models#connecting-to-azure-openai)
- [Codex with Azure OpenAI in Microsoft Foundry Models](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/codex?tabs=npm)