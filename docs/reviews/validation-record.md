# 문서 작업 검증 기록

일자: 2026-10-03 (Asia/Seoul). 이 기록은 착수 조사·설계 문서 검사이며 ADE-54 재생 테스트 로그나 ADE-55 계산 테스트 결과가 아니다.

## 실행한 조사와 결과

| 명령/조회 | 결과 |
| --- | --- |
| `Get-Location`, `git rev-parse --show-toplevel` | 요청한 폴더가 Git 루트 |
| `git remote -v`, `Get-Content .git/config` | remote 없음 |
| `git branch --show-current`, `Get-Content .git/HEAD` | 시작 시 master unborn; 문서는 별도 브랜치에서 작업 |
| `git rev-parse HEAD`, `git log -5 --oneline` (시작 시) | 커밋 없어 실패. detached HEAD나 저장소 손상으로 판정하지 않음 |
| `git status --short --branch`, `git ls-files`, `git ls-files --others --exclude-standard` (시작 시) | 사용자 파일·기존 변경 없음 |
| `Get-ChildItem -Force`, `rg --files --hidden`, 부모 경로 AGENTS.md 확인 | `.git` 외 파일 없음; 적용할 AGENTS.md 없음 |
| `git worktree list --porcelain`, `git show-ref` | 현재 worktree 하나; 내부 capture 빈 트리 ref 존재; 유효 프로젝트 기준 커밋 없음 |
| `python --version` | 3.11.15 |
| `python -m pip show pyorbbecsdk openni pyrealsense2 numpy pytest` | pip 미설치로 실패; SDK 설치 여부의 증거로 사용하지 않음 |
| `importlib.metadata.distributions()` 패키지 조회 | 현재 인터프리터에 pyorbbecsdk/openni/primesense/pyrealsense2/numpy/pytest 배포 없음 |
| Linear `get_issue` ADE-51·54·55, `list_comments` 각 이슈 | 본문 조회 성공; 세 이슈 Todo, 댓글 없음, 다음 페이지 없음 |
| Linear `get_project`, `list_comments`, `list_documents` | 프로젝트 본문 조회 성공; 리소스·댓글·문서 없음, 다음 페이지 없음 |

실패/미설치 결과는 “SDK가 시스템 전체에 없다”거나 “승인 기록이 다른 곳에도 없다”는 근거가 아니다.

## 문서 검사 재현 명령

프로젝트 루트에서 PowerShell로 실행한다. 외부 패키지 없이 Python 표준 라이브러리만 사용한다.

```powershell
@'
import pathlib,re,json
root=pathlib.Path('docs/reviews')
paths=sorted(root.glob('*.md'))
blocks=0
links=0
for p in paths:
    text=p.read_text(encoding='utf-8')
    assert '\ufffd' not in text, str(p)
    for block in re.findall(r'^```json[ \t]*\r?\n(.*?)^```[ \t]*$',text,re.S|re.M):
        json.loads(block)
        blocks+=1
    for link in re.findall(r'\]\(([^)]+)\)',text):
        if '://' in link or link.startswith('#'):
            continue
        target=link.split('#')[0]
        if target:
            assert (p.parent/target).is_file(), f'{p}: missing {target}'
            links+=1
print(f'PASS: {len(paths)} UTF-8 Markdown files, {blocks} JSON examples, {links} relative file links')
'@ | python -
git diff --check
git diff --cached --check
```

JSON 검사는 줄 시작의 JSON 코드 블록만 읽으며 실행 모듈을 검증하지 않는다. 상대 링크 검사는 대상 파일 존재만 확인하고 문서 내 앵커나 원격 링크의 유효성을 검증하지 않는다.

최종 확인: UTF-8 문서 7개, JSON 예제 3개, 상대 파일 링크 4개 검사 통과. ADE-54 stage diff 검사 통과 후 로컬 문서 커밋 `85beaeaba68496026e8c105bd9cfbc3d0c006363`을 만들었다. ADE-55도 stage diff 검사 및 변경 diff 검토 후 별도 로컬 문서 커밋으로 보존한다. 최종 HEAD와 working tree는 최종 보고의 Git 항목을 참조한다. LF→CRLF 경고는 호스트 Git 설정에 따른 안내이며 문서 오류로 처리하지 않았다.

## 미실행 검증과 이유

| 검증 | 상태·이유 |
| --- | --- |
| 가상환경 설치·입력 CLI 실행·저장 프레임 재생 로그 | 미실행: 환경 정의·의존성·입력 규격·승인·샘플 미확정, 코드 없음 |
| 정상/실패 입력의 실제 공통 구조 출력 | 미실행: 문서 예제만 있고 승인된 샘플/실행 모듈 없음 |
| 단위 변환·Point Cloud 좌표·기준 부피·재현성 | 미실행: 승인 전 계산 코드 없음. 문서 수치는 기대값일 뿐 실행 결과가 아님 |
| 로봇·모바일 두 경로 계약 준수 | 미완료: 실제 두 경로 샘플/adapter 없음 |
| 촬영·로봇·IMU 시간 동기화 | 미완료: clock mapping·offset·오차 자료 없음 |
| Astra Mini/Mini Pro PC 및 실제 휴대폰 연결 | 미완료: 장치 연결·모델별 SDK/driver 버전·공식 호환 자료 미확인 |
| SDK 설치 버전별 공식 API 대조 | 미실행: SDK/API를 사용하지 않았고 채택할 설치 버전 미확정 |
| 실제 장비 부피 정확도 / 중량 오차 10% | 미평가: 기준 실측·모델·평가 자료 없음; 부피 모듈 테스트로 중량 목표를 주장하지 않음 |

push·PR 생성·Linear 댓글/상태 변경·팀장 메시지·최종 병합은 수행하지 않았다.
