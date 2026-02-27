# Claude Code 엔터프라이즈 보안 설정 가이드
                                                                                                                           
   > **대상**: 금융회사 환경 (LLM Gateway → Amazon Bedrock → Claude)                                                       
   > **작성일**: 2026-02-28                                                                                                
   > **MDM**: JAMF Pro v11.19.1 (DEP 등록, User Approved)                                                                  
                                                                                                                           
   ---

   ## 1. 아키텍처 개요

   ```
   ┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌────────┐
   │ Claude Code  │────▶│ LLM Gateway  │────▶│   Amazon    │────▶│ Claude │
   │  (개발자 PC)  │     │  (사내 프록시)  │     │   Bedrock   │     │  API   │
   └─────────────┘     └──────────────┘     └─────────────┘     └────────┘
          │
          ▼
   ┌─────────────────────────────────────┐
   │  managed-settings.json (JAMF 배포)   │
   │  /Library/Application Support/      │
   │  ClaudeCode/managed-settings.json   │
   └─────────────────────────────────────┘
   ```

   ---

   ## 2. managed-settings.json 설정 경로

   | OS | 경로 |
   |:---|:-----|
   | **macOS** | `/Library/Application Support/ClaudeCode/managed-settings.json` |
   | **Linux/WSL** | `/etc/claude-code/managed-settings.json` |
   | **Windows** | `C:\Program Files\ClaudeCode\managed-settings.json` |

   ### 설정 우선순위 (높은 순)

   | 순위 | 범위 | 재정의 가능 여부 |
   |:----:|:-----|:--------------|
   | 1 | **Managed** (서버 관리 > MDM/plist > managed-settings.json) | 불가 (최상위) |
   | 2 | 명령줄 인수 | 임시 세션 |
   | 3 | Local (`.claude/settings.local.json`) | 가능 |
   | 4 | Project (`.claude/settings.json`) | 가능 |
   | 5 | User (`~/.claude/settings.json`) | 가능 |

   > **핵심**: managed-settings.json은 **최상위 우선순위**이므로 사용자/프로젝트 설정으로 절대 재정의할 수 없습니다.

   ---

   ## 3. 금융회사 권장 보안 설정 항목

   ### 3-1. 권한 및 접근 제어

   | 설정 키 | 값 (예시) | 설명 | 금융사 필요성 |
   |:--------|:----------|:-----|:------------|
   | `permissions.deny` | `["Read(./.env)", "Read(./.env.*)", "Read(./secrets/**)", "Read(~/.aws/**)", "Bash(curl *)",
   "Bash(wget *)", "WebFetch"]` | 민감 파일 읽기 및 외부 네트워크 호출 차단 | **필수** - 자격증명/비밀키 유출 방지 |
   | `permissions.defaultMode` | `"acceptEdits"` | 기본 권한 모드 (편집만 자동 승인) | **권장** - 무분별한 자동실행 방지 |
   | `permissions.disableBypassPermissionsMode` | `"disable"` | `--dangerously-skip-permissions` 플래그 비활성화 | **필수**
    - 권한 우회 차단 |
   | `allowManagedPermissionRulesOnly` | `true` | 사용자/프로젝트의 allow/deny 규칙 정의 방지, 관리자 규칙만 적용 |
   **권장** - 중앙 집중 정책 |

   ### 3-2. MCP 서버 제어

   | 설정 키 | 값 (예시) | 설명 | 금융사 필요성 |
   |:--------|:----------|:-----|:------------|
   | `allowManagedMcpServersOnly` | `true` | 관리자 정의 MCP 서버 허용 목록만 적용 | **필수** - 비인가 도구 차단 |
   | `allowedMcpServers` | `[{"serverName": "github"}, {"serverName": "SecurityGuard"}]` | 허용된 MCP 서버 화이트리스트 |
   **필수** - 승인된 도구만 사용 |
   | `deniedMcpServers` | `[{"serverName": "filesystem"}]` | 명시적 차단 MCP 서버 | **권장** - 위험 도구 차단 |

   ### 3-3. Hook 및 플러그인 제어

   | 설정 키 | 값 (예시) | 설명 | 금융사 필요성 |
   |:--------|:----------|:-----|:------------|
   | `allowManagedHooksOnly` | `true` | 관리자 hook만 허용, 사용자/프로젝트/플러그인 hook 차단 | **필수** - 악성 hook 방지
   |
   | `disableAllHooks` | `false` | 관리자 hook은 유지하되 통제 | 상황별 |
   | `strictKnownMarketplaces` | `[{"source": "github", "repo": "company/approved-plugins"}]` | 승인된 플러그인
   마켓플레이스만 허용 | **필수** - 비인가 플러그인 차단 |
   | `blockedMarketplaces` | `[{"source": "github", "repo": "untrusted/plugins"}]` | 차단된 마켓플레이스 (다운로드 전 확인)
    | **권장** |

   ### 3-4. 네트워크 및 샌드박스

   | 설정 키 | 값 (예시) | 설명 | 금융사 필요성 |
   |:--------|:----------|:-----|:------------|
   | `sandbox.enabled` | `true` | bash 명령 샌드박싱 활성화 | **필수** - 명령 격리 |
   | `sandbox.allowUnsandboxedCommands` | `false` | `dangerouslyDisableSandbox` 이스케이프 완전 비활성화 | **필수** -
   샌드박스 우회 차단 |
   | `sandbox.network.allowedDomains` | `["github.com", "*.company.com"]` | 아웃바운드 도메인 화이트리스트 | **필수** -
   데이터 유출 방지 |
   | `sandbox.network.allowManagedDomainsOnly` | `true` | 관리자 도메인만 허용, 사용자 추가 도메인 무시 | **필수** |

   ### 3-5. 인증 및 모델 제어

   | 설정 키 | 값 (예시) | 설명 | 금융사 필요성 |
   |:--------|:----------|:-----|:------------|
   | `forceLoginMethod` | `"console"` | Console(API) 계정으로만 로그인 제한 | **권장** - 인증 통일 |
   | `forceLoginOrgUUID` | `"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"` | 조직 자동 선택 | **권장** |
   | `model` | `"us.anthropic.claude-sonnet-4-6-v1"` | 기본 모델 지정 | 선택 |
   | `availableModels` | `["sonnet", "haiku"]` | 사용 가능 모델 제한 (비용 통제) | **권장** - 비용 관리 |

   ### 3-6. 텔레메트리 및 기타

   | 설정 키 | 값 (예시) | 설명 | 금융사 필요성 |
   |:--------|:----------|:-----|:------------|
   | `env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | `"1"` | 자동 업데이트, 버그 리포트, 에러 리포트, 텔레메트리 전부
   비활성화 | **필수** - 외부 통신 차단 |
   | `env.DISABLE_TELEMETRY` | `"1"` | Statsig 텔레메트리 거부 | **필수** |
   | `env.DISABLE_AUTOUPDATER` | `"1"` | 자동 업데이트 비활성화 (JAMF로 관리) | **필수** |
   | `cleanupPeriodDays` | `7` | 7일 이상 비활성 세션 자동 삭제 | **권장** - 데이터 최소화 |
   | `companyAnnouncements` | `["[보안공지] 소스코드 및 고객정보를 Claude에 입력하지 마세요"]` | 시작 시 보안 공지 표시 |
   **권장** |

   ---

   ## 4. 권장 managed-settings.json (전체)

   ```json
   {
     "$schema": "https://json.schemastore.org/claude-code-settings.json",

     "permissions": {
       "deny": [
         "Read(./.env)",
         "Read(./.env.*)",
         "Read(./secrets/**)",
         "Read(~/.aws/**)",
         "Read(~/.ssh/**)",
         "Read(./config/credentials*)",
         "Bash(curl *)",
         "Bash(wget *)",
         "Bash(nc *)",
         "Bash(ncat *)",
         "Bash(ssh *)",
         "Bash(scp *)",
         "Bash(rsync *)",
         "Bash(aws s3 cp *)",
         "Bash(aws s3 sync *)",
         "WebFetch"
       ],
       "defaultMode": "acceptEdits",
       "disableBypassPermissionsMode": "disable"
     },

     "allowManagedPermissionRulesOnly": true,
     "allowManagedHooksOnly": true,
     "allowManagedMcpServersOnly": true,

     "allowedMcpServers": [
       { "serverName": "github" },
       { "serverName": "SecurityGuard" }
     ],
     "deniedMcpServers": [
       { "serverName": "filesystem" }
     ],

     "strictKnownMarketplaces": [
       {
         "source": "github",
         "repo": "your-company/approved-claude-plugins"
       }
     ],

     "sandbox": {
       "enabled": true,
       "autoAllowBashIfSandboxed": false,
       "allowUnsandboxedCommands": false,
       "excludedCommands": ["git"],
       "network": {
         "allowedDomains": [
           "github.com",
           "*.your-company.com",
           "*.amazonaws.com"
         ],
         "allowManagedDomainsOnly": true,
         "allowLocalBinding": false
       }
     },

     "availableModels": ["sonnet", "haiku"],
     "cleanupPeriodDays": 7,

     "env": {
       "CLAUDE_CODE_USE_BEDROCK": "1",
       "CLAUDE_CODE_SKIP_BEDROCK_AUTH": "0",
       "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
       "DISABLE_TELEMETRY": "1",
       "DISABLE_AUTOUPDATER": "1",
       "DISABLE_ERROR_REPORTING": "1",
       "DISABLE_BUG_COMMAND": "1",
       "CLAUDE_CODE_HIDE_ACCOUNT_INFO": "1",
       "CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY": "1"
     },

     "companyAnnouncements": [
       "[보안공지] 소스코드, 고객정보, 인증정보를 Claude에 직접 입력하지 마세요.",
       "[안내] Claude Code는 LLM Gateway → Bedrock 경로로 운영됩니다. 문의: 보안팀"
     ]
   }
   ```

   ---

   ## 5. JAMF MDM을 통한 디렉토리 보호

   ### 5-1. 배포 방법 (2가지)

   #### 방법 A: Configuration Profile (plist 기반) — 권장

   JAMF에서 `com.anthropic.claudecode` managed preferences 도메인으로 구성 프로필 배포:

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
     "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0">
   <dict>
     <key>PayloadContent</key>
     <array>
       <dict>
         <key>PayloadType</key>
         <string>com.anthropic.claudecode</string>
         <key>PayloadDisplayName</key>
         <string>Claude Code Security Policy</string>
         <key>PayloadIdentifier</key>
         <string>com.company.claudecode.security</string>
         <key>PayloadUUID</key>
         <string>GENERATE-UUID-HERE</string>
         <key>PayloadVersion</key>
         <integer>1</integer>

         <!-- managed-settings.json 내용을 plist 형태로 변환하여 배포 -->
         <key>permissions</key>
         <dict>
           <key>deny</key>
           <array>
             <string>Read(./.env)</string>
             <string>Read(./.env.*)</string>
             <string>Read(./secrets/**)</string>
             <string>Read(~/.aws/**)</string>
             <string>Bash(curl *)</string>
             <string>Bash(wget *)</string>
             <string>WebFetch</string>
           </array>
           <key>disableBypassPermissionsMode</key>
           <string>disable</string>
         </dict>

         <key>allowManagedPermissionRulesOnly</key>
         <true/>
         <key>allowManagedHooksOnly</key>
         <true/>
         <key>allowManagedMcpServersOnly</key>
         <true/>
       </dict>
     </array>
   </dict>
   </plist>
   ```

   > **장점**: MDM 프로필은 사용자가 제거 불가, managed-settings.json보다 우선순위 높음

   #### 방법 B: 파일 기반 배포 + 보호

   JAMF 스크립트로 managed-settings.json 배포 후 파일 보호:

   ```bash
   #!/bin/bash
   # JAMF 배포 스크립트: deploy_claude_managed_settings.sh

   CLAUDE_DIR="/Library/Application Support/ClaudeCode"
   SETTINGS_FILE="${CLAUDE_DIR}/managed-settings.json"

   # 1. 디렉토리 생성
   mkdir -p "${CLAUDE_DIR}"

   # 2. managed-settings.json 배포
   cat > "${SETTINGS_FILE}" << 'SETTINGS_EOF'
   {
     // 위 섹션 4의 전체 JSON 내용 삽입
   }
   SETTINGS_EOF

   # 3. 권한 설정 (root만 읽기/쓰기)
   chown root:wheel "${CLAUDE_DIR}"
   chmod 755 "${CLAUDE_DIR}"
   chown root:wheel "${SETTINGS_FILE}"
   chmod 644 "${SETTINGS_FILE}"

   # 4. 시스템 불변 플래그 설정 (root도 수정 불가)
   chflags schg "${SETTINGS_FILE}"
   chflags schg "${CLAUDE_DIR}"

   echo "Claude Code managed settings deployed and locked."
   ```

   ### 5-2. root도 수정 불가하게 하는 방법

   | 방법 | 명령어 | root 수정 가능? | 해제 조건 |
   |:-----|:-------|:--------------|:---------|
   | **chflags schg** (시스템 불변 플래그) | `chflags schg <file>` | **불가** (SIP 활성 시) | SIP 비활성화 필요 (`csrutil
   disable`) |
   | **chflags uchg** (사용자 불변 플래그) | `chflags uchg <file>` | 가능 (root가 해제 가능) | `chflags nouchg <file>` |
   | **MDM 구성 프로필** | JAMF Configuration Profile | **불가** (MDM에서만 제거) | JAMF 콘솔에서만 제거 가능 |
   | **chmod 444** | `chmod 444 <file>` | 가능 (root가 변경 가능) | `chmod` 명령 |

   #### 결론: `chflags schg` + SIP 조합이 가장 강력

   ```
   SIP 활성 상태 (기본값)에서 chflags schg 설정 시:
     ✅ 일반 사용자 수정 불가
     ✅ root 사용자 수정 불가
     ✅ chflags noschg 명령도 실패 (SIP이 차단)
     ❌ 해제하려면 복구 모드 부팅 후 csrutil disable 필요
   ```

   현재 시스템 SIP 상태 확인:
   ```bash
   csrutil status
   # 결과: System Integrity Protection status: enabled. (정상)
   ```

   ### 5-3. JAMF 배포 절차 요약

   ```
   1. JAMF Pro 콘솔 → Settings → Computer Management → Scripts
      → "deploy_claude_managed_settings.sh" 업로드

   2. Policies → New Policy
      → Trigger: Enrollment Complete + Recurring Check-in
      → Scope: 개발자 그룹 (Smart Group)
      → Scripts: deploy_claude_managed_settings.sh 실행

   3. (선택) Configuration Profile 추가 배포
      → com.anthropic.claudecode 도메인으로 plist 배포
      → MDM 프로필은 사용자 제거 불가

   4. 검증: jamf recon && cat "/Library/Application Support/ClaudeCode/managed-settings.json"
   ```

   ---

   ## 6. history.jsonl 토큰 마스킹

   ### 6-1. history.jsonl 구조 분석

   | 위치 | 내용 |
   |:-----|:-----|
   | `~/.claude/history.jsonl` | 전역 명령 히스토리 (사용자 입력, 붙여넣기 내용, 타임스탬프) |
   | `~/.claude/projects/<project>/<session>.jsonl` | 프로젝트별 대화 히스토리 (전체 대화 내용 포함) |
   | `~/.claude/debug/` | 디버그 로그 (333개 파일) |

   **history.jsonl 레코드 구조:**
   ```json
   {
     "display": "curl -H 'Authorization: Bearer sk-ant-xxxx' https://api...",
     "pastedContents": {
       "1": {
         "id": 1,
         "type": "text",
         "content": "ANTHROPIC_API_KEY=sk-ant-api03-xxxx..."
       }
     },
     "timestamp": 1772208506062,
     "project": "/Users/sb.lee/go/src/project",
     "sessionId": "uuid-here"
   }
   ```

   > **위험**: `display` 필드와 `pastedContents.content` 필드에 사용자가 입력/붙여넣기한 토큰, API 키, 비밀번호가
   **평문으로 저장**됩니다.

   ### 6-2. 마스킹 방법

   #### 방법 1: Hook 기반 실시간 마스킹 (권장)

   Claude Code의 `hooks` 기능으로 세션 종료 시 자동 마스킹:

   **managed-settings.json에 추가:**
   ```json
   {
     "hooks": {
       "Stop": [
         {
           "matcher": "",
           "hooks": [
             {
               "type": "command",
               "command": "/usr/local/bin/claude-history-masker.sh"
             }
           ]
         }
       ]
     }
   }
   ```

   **마스킹 스크립트 (`/usr/local/bin/claude-history-masker.sh`):**
   ```bash
   #!/bin/bash
   # Claude Code History Token Masker
   # 세션 종료 시 history.jsonl에서 민감 정보를 마스킹

   CLAUDE_DIR="${HOME}/.claude"
   HISTORY_FILE="${CLAUDE_DIR}/history.jsonl"

   if [ ! -f "$HISTORY_FILE" ]; then
     exit 0
   fi

   # 토큰 패턴 정의 (정규표현식)
   # - AWS Access Key: AKIA로 시작하는 20자
   # - AWS Secret Key: 40자 영숫자+슬래시+플러스
   # - Anthropic API Key: sk-ant-로 시작
   # - Bearer Token: Bearer 뒤의 토큰
   # - Generic API Key: api_key, apikey, api-key 패턴
   # - 일반 비밀번호: password= 패턴

   TEMP_FILE=$(mktemp)

   sed -E \
     -e 's/(AKIA)[A-Z0-9]{16}/\1****MASKED****/g' \
     -e 's/(sk-ant-)[a-zA-Z0-9_-]{10,}/\1****MASKED****/g' \
     -e 's/(Bearer\s+)[a-zA-Z0-9._\-]{10,}/\1****MASKED****/g' \
     -e 's/(api[_-]?[Kk]ey["\s:=]+)[a-zA-Z0-9_\-]{10,}/\1****MASKED****/g' \
     -e 's/(password["\s:=]+)[^\s",}]{3,}/\1****MASKED****/g' \
     -e 's/(secret[_-]?[Kk]ey["\s:=]+)[a-zA-Z0-9\/+_\-]{10,}/\1****MASKED****/g' \
     -e 's/(token["\s:=]+)[a-zA-Z0-9._\-]{10,}/\1****MASKED****/g' \
     -e 's/(AWS_SECRET_ACCESS_KEY["\s:=]+)[a-zA-Z0-9\/+]{10,}/\1****MASKED****/g' \
     -e 's/(AWS_SESSION_TOKEN["\s:=]+)[a-zA-Z0-9\/+=]{10,}/\1****MASKED****/g' \
     "$HISTORY_FILE" > "$TEMP_FILE"

   mv "$TEMP_FILE" "$HISTORY_FILE"
   chmod 600 "$HISTORY_FILE"
   ```

   #### 방법 2: Cron 기반 주기적 마스킹

   ```bash
   # crontab -e (또는 JAMF로 LaunchDaemon 배포)
   */30 * * * * /usr/local/bin/claude-history-masker.sh
   ```

   #### 방법 3: CLAUDE_CODE_SHELL_PREFIX로 Bash 명령 래핑

   ```json
   {
     "env": {
       "CLAUDE_CODE_SHELL_PREFIX": "/usr/local/bin/claude-cmd-auditor.sh"
     }
   }
   ```

   이 래퍼 스크립트에서 실행되는 명령에 포함된 토큰을 감사 로그에 마스킹하여 기록할 수 있습니다.

   ### 6-3. 마스킹 방법 비교

   | 방법 | 실시간 여부 | 구현 난이도 | 커버리지 | 권장 |
   |:-----|:----------|:----------|:---------|:-----|
   | **Hook (Stop 이벤트)** | 세션 종료 시 | 중 | history.jsonl | **권장** |
   | **Cron/LaunchDaemon** | 주기적 (30분) | 하 | 전체 .jsonl | 보조 수단 |
   | **SHELL_PREFIX 래퍼** | 실시간 | 상 | Bash 명령만 | 감사 로그용 |
   | **cleanupPeriodDays: 0** | 시작 시 전체 삭제 | 하 | 전체 | 극단적 (비권장) |

   ### 6-4. 한계 및 주의사항

   - Claude Code에는 **내장 마스킹 기능이 없음** — 외부 스크립트로 구현 필요
   - 프로젝트별 세션 파일 (`~/.claude/projects/.../*.jsonl`)도 마스킹 대상에 포함해야 함
   - `sed` 기반 마스킹은 정규표현식 패턴에 의존하므로 **모든 토큰 형식을 커버하지 못할 수 있음**
   - 가장 확실한 방법은 `permissions.deny`로 **토큰이 포함될 수 있는 명령 자체를 차단**하는 것

   ---

   ## 7. 종합 체크리스트

   | # | 항목 | 상태 | 담당 |
   |:-:|:-----|:----:|:----:|
   | 1 | `/Library/Application Support/ClaudeCode/` 디렉토리 생성 | ⬜ | IT/보안팀 |
   | 2 | managed-settings.json 배포 | ⬜ | IT/보안팀 |
   | 3 | `chflags schg` 불변 플래그 설정 | ⬜ | IT/보안팀 |
   | 4 | SIP 활성 상태 확인 (`csrutil status`) | ⬜ | IT/보안팀 |
   | 5 | JAMF Configuration Profile 배포 (선택) | ⬜ | IT/보안팀 |
   | 6 | history.jsonl 마스킹 스크립트 배포 | ⬜ | IT/보안팀 |
   | 7 | 개발자 그룹 대상 배포 테스트 | ⬜ | IT/보안팀 |
   | 8 | `/status` 명령으로 적용 확인 | ⬜ | 개발자 |

   ---

   ## 참고 문서

   - [Claude Code 설정 공식 문서](https://code.claude.com/docs/ko/settings)
   - [Claude Code 권한 관리](https://code.claude.com/docs/ko/permissions)
   - [Claude Code MCP 설정](https://code.claude.com/docs/ko/mcp)
   - [Claude Code 샌드박싱](https://code.claude.com/docs/ko/sandboxing)
   - [Amazon Bedrock 연동](https://code.claude.com/docs/ko/amazon-bedrock)
   - [JSON Schema (자동완성)](https://json.schemastore.org/claude-code-settings.json)
