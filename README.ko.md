# Blender Agent

[English README](README.md)

Blender 안에서 Python 코드를 HTTP로 실행하는 작은 애드온입니다. MCP나 별도
프로토콜 없이 `curl`만으로 Blender를 제어할 수 있습니다.

Codex, Gemini CLI, Claude Code와 함께 사용해서 3D 장면, 모션 그래픽, 영상 편집
타임라인을 자연어 대화로 설계하고 수정하는 것을 목표로 합니다.

대상 버전은 **Blender 5.1+** 입니다.

## Fork 정보

이 저장소는 [ptrthomas/blender-agent](https://github.com/ptrthomas/blender-agent)에서
fork했습니다. 원본 프로젝트는 Peter Thomas가 관리하며, Blender HTTP 애드온과
초기 Claude Code Skill 워크플로우를 제공합니다.

이 fork는 다음 방향으로 확장합니다.

- Codex와 Gemini CLI에서도 사용할 수 있는 문서 구조
- Blender API 사용 시 학습 데이터보다 Skill 문서를 우선하는 규칙
- 사용자가 원하는 형상, 비율, 재질, 조명, 움직임을 대화로 설계하는 워크플로우
- `.agents/skills/`를 canonical Skill 트리로 사용
- `.claude/skills/`는 Claude Code 호환 mirror로 유지

## 빠른 시작

### 1. 애드온 설치

Blender 확장 디렉터리에 심볼릭 링크를 만듭니다. 한 번만 실행하면 됩니다.

```bash
mkdir -p ~/Library/Application\ Support/Blender/5.1/extensions/user_default
ln -sf $(pwd)/blender_agent ~/Library/Application\ Support/Blender/5.1/extensions/user_default/blender_agent
```

그 다음 Blender에서 **Edit > Preferences > Add-ons**로 이동해 `Blender Agent`를
검색하고 활성화합니다.

### 2. 서버 시작

```bash
python3 start_server.py
```

이 명령은 Blender를 실행하거나 이미 실행 중인 Blender에 연결하고, HTTP 서버가
준비될 때까지 기다린 뒤 Blender 버전을 출력합니다.

수동으로 시작하려면 Blender 3D Viewport에서 `N` 키를 누른 뒤 **Agent** 탭의
**Start** 버튼을 누르면 됩니다.

### 3. 동작 확인

```bash
curl -s localhost:5656 --data-binary @- <<< 'bpy.app.version_string'
```

예상 응답:

```json
{"ok": true, "result": "5.1.0", "output": ""}
```

## 에이전트와 함께 사용하기

이 저장소에는 Blender 작업을 위한 Agent Skills가 포함되어 있습니다.

| Skill | 사용 시점 |
|-------|-----------|
| `blender` | 일반 Blender 자동화, 장면 검사, 스크린샷, 로그 확인 |
| `blender-3d` | 오브젝트, 재질, 카메라, 조명, 애니메이션, 렌더링 |
| `blender-vse` | 영상 타임라인, 자막, strip, 전환, VSE 렌더링 |
| `blender-geometry-nodes` | Geometry Nodes, 절차적 형상, instancing, scatter |
| `blender-laser` | 레이저 빔, raycast 반사, bouncing light path |
| `blender-projector` | spotlight projection, gobo, volumetric beam |
| `blender-audioviz` | 오디오 분석, beat-sync 조명, reactive material |

에이전트별 진입 문서:

- Codex: `AGENTS.md`를 읽고 `.agents/skills/`를 canonical Skill 트리로 사용합니다.
- Gemini CLI: `GEMINI.md`를 읽습니다. 이 파일은 `AGENTS.md`를 import합니다.
- Claude Code: `CLAUDE.md`를 읽습니다. `.claude/skills/`는 호환 mirror입니다.

## 대화형 설계 방식

이 fork의 핵심은 단순한 prompt-to-render가 아닙니다. 사용자가 만들고 싶은 형상,
수정하고 싶은 부분, 재질감, 비율, 움직임, 조명 방향을 에이전트와 대화하면서
구체화하고, 그 내용을 작은 Blender 수정으로 반복 적용하는 방식입니다.

작업 흐름:

1. 사용자가 원하는 형상이나 수정 방향을 설명합니다.
2. 에이전트는 필요한 질문을 하고 대안을 제안합니다.
3. 관련 Skill 문서를 먼저 확인합니다.
4. Blender에 작은 수정 단위를 적용합니다.
5. 렌더나 스크린샷을 확인합니다.
6. 결과를 설명하고 다음 수정 방향을 다시 대화로 정합니다.

Blender API 사용은 학습 데이터보다 Skill 문서를 우선합니다. Skill에 없는 API나
Blender 5.1 변경점이 나오면 공식 문서를 확인하고, 확인된 패턴은 Skill 문서에
반영합니다.

## 질문 기반 Add-on 설계

우리가 설계하려는 것은 단순한 결과 이미지가 아니라, Blender 안에서 계속 사용할 수
있는 Add-on 기능과 작업 흐름입니다. 따라서 에이전트는 바로 코드를 작성하기보다
질문을 통해 문제를 구체화해야 합니다.

진행 방식:

1. 사용자가 Blender 안에서 무엇을 바꾸고 싶은지 질문합니다.
   예: 형상, 조작 패널, operator 동작, procedural geometry, 재질 반응, 렌더 흐름.
2. 그 요구를 Blender Add-on 기능으로 다시 정의합니다.
3. 관련 Skill 문서를 먼저 확인합니다.
4. Skill에 없는 부분은 Blender 공식 문서에서 확인합니다.
5. Blender 안에서 바로 테스트할 수 있는 작은 구현 단위를 제안합니다.
6. Add-on 코드나 Skill 기반 Blender script로 적용합니다.
7. 결과를 확인하고, 설계 의도와 맞는지 다시 대화합니다.

참고할 Blender 공식 문서:

- Extension 생성: https://docs.blender.org/manual/en/latest/advanced/extensions/
- Add-ons 설치와 Preferences: https://docs.blender.org/manual/en/latest/editors/preferences/addons.html
- Extension 명령줄 사용: https://docs.blender.org/manual/en/dev/advanced/command_line/extension_arguments.html
- Blender 5.1 Python API 변경점: https://developer.blender.org/docs/release_notes/5.1/python_api/

예시 요청:

```text
> 빛나는 네온 큐브를 만들고 천천히 회전하게 해 주세요.

> 이 장면의 바닥 패턴을 음악 박자에 맞춰 반응하게 만들고 싶습니다.

> 지금 형상이 너무 딱딱합니다. 가장자리를 부드럽게 하고 유리 같은 재질로 바꿔 주세요.
```

## 수동 사용법

여러 줄 Python 코드는 heredoc으로 보내는 것이 안전합니다.

```bash
curl -s localhost:5656 --data-binary @- <<'PYEOF'
import bpy
bpy.ops.mesh.primitive_cube_add(location=(1, 2, 3))
bpy.data.objects.keys()
PYEOF
```

또는 스크립트 파일을 직접 보낼 수 있습니다.

```bash
curl -s localhost:5656 --data-binary @script.py
```

응답 형식:

```json
{"ok": true, "result": "<last expression>", "output": "<stdout>"}
{"ok": false, "error": "<message>", "output": "<stdout before error>"}
```

- `result`: 마지막 표현식의 반환값
- `output`: `print()` 출력
- `error`: 오류 메시지

## 동작 방식

애드온은 Blender 내부에서 작은 HTTP 서버를 실행합니다. POST 요청을 받으면 요청
본문의 Python 코드를 Blender 메인 스레드에서 실행하고, 결과를 JSON으로 돌려줍니다.

렌더, 스크린샷, 로그는 `output/` 디렉터리를 사용합니다. HTTP 서버가 주입하는
`OUTPUT` 변수를 사용하면 됩니다.

## 라이선스

이 프로젝트는 MIT License를 따릅니다. 자세한 내용은 [LICENSE](LICENSE)를 확인하세요.
