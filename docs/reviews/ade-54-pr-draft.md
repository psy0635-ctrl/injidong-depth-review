# 로컬 PR 초안 — 미게시

제목: `docs(ADE-54): PC Depth 입력 설계와 공통 규격 검토안`

현재 작업 폴더는 커밋·remote·구현이 없는 초기 Git 저장소이며 ADE-51 승인 규격을 확인하지 못했다. PC 저장 프레임 재생 및 공통 입력 변환 착수 전에 검토할 환경 정의, 제안 스키마, 실패 정책, 검증 계획, 실기기 연결 체크리스트를 문서화한다.

변경: `repository-audit.md`, `common-contract-proposal.md`, `ade-54-input-design.md`, 이 PR 초안. SDK/API·변환 코드는 구현하지 않았다. Orbbec Astra Mini/Mini Pro 사용자 전달 정보와 Linear의 RealSense 기록 차이를 명시했다.

검증: Git/파일 조사, Linear ADE-51·54·55/프로젝트 읽기 조회, 현재 Python의 설치 패키지 metadata 확인. 문서 예제 JSON의 구문 확인과 `git diff --check` 결과는 작업 보고에서 기록한다. 프레임 재생·실기기·시간 동기화 테스트는 규격/승인/샘플 부재로 미실행이다.

검토 요청: 착수 조사 문서의 질문 8개에 대한 답변과 승인 근거 링크. 완료 또는 성능 목표 달성 PR이 아니다. 실제 remote·기준 브랜치가 정해진 뒤 PR base를 결정한다. push·PR 생성·Linear 변경·팀장 메시지 전송 없음.
