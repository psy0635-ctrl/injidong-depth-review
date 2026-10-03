# 공통 입력·출력 규격 검토안

상태: **제안, ADE-51 및 강석현 검토 필요**. 승인된 규격이나 실행 가능한 구현이 아니다. 기존 스키마를 확보하면 우선 재사용하고 차이를 팀장에게 검토받는다.

## 공통 입력 (제안)

장비별 adapter는 원본을 읽어 아래 구조로 정규화한다. 계산 코어는 장비 SDK에 의존하지 않는다. `schema_version: draft-1`은 승인 전 문서 예제에만 사용한다.

| 필드 | 제안 규칙 |
| --- | --- |
| `frame_id`, `source_path` | 프레임 식별자, `robot` / `mobile_pc_replay` / `mobile_device`; 원래 수집 경로도 별도 기록 |
| `provenance` | `real` / `synthetic`, 출처·장비 모델·원본 경로·원본 파일 SHA-256; 합성이라면 생성 조건·seed |
| `depth` | 정규화된 H×W 실수 배열, 단위 m. 원본 dtype·해상도·깊이 스케일·원본 invalid 규칙 보존. 무효값은 별도 boolean `valid_mask`로 표시하고 계산에서 제외 |
| `depth_semantics` | 코어 입력은 광축 Z. 광선 거리 입력은 해당 보정 모델에 근거한 변환이 승인된 adapter만 허용; 단순 단위 변환으로 Z 취급 금지 |
| `intrinsics` | 해당 Depth 해상도의 `width,height,fx,fy,cx,cy`, 픽셀 단위. 깊이 카메라 보정 출처·보정 시각·왜곡 모델/계수·정류 여부 기록 |
| `timestamp` | `capture_ns`, `clock_domain`, `origin`(sensor/host/synthetic), 누락이면 null. 파일 재생 시각은 `replay_ns`로 분리 |
| `pose` | null 또는 명시적 `from_frame,to_frame,translation_m,rotation_quaternion_xyzw,timestamp_ns,clock_domain,calibration_id`; IMU orientation과 카메라↔IMU 외부 파라미터 분리 |
| `sync` | `verified` / `unverified` / `missing`, offset·불확실도·측정 근거. 자세와 프레임 시각이 있다는 이유만으로 verified 금지 |
| `capture_conditions` | 촬영 거리·노출 등 가용 메타데이터, 알 수 없으면 null. 촬영 환경·대상 식별자 기록 |
| `target_mask` | null 또는 H×W boolean 마스크, 생성/라벨 출처와 버전. ROI만으로 배경이 완전히 분리됐다고 간주하지 않음 |
| `support_plane` | null 또는 카메라 좌표계에서 단위 법선 `normal`과 거리 `offset_m`, 추정/측정 근거·오차. 자세 없이 임의 world 평면 생성 금지 |

`fx,fy > 0`, 모든 내부 파라미터는 finite, 해상도와 배열·마스크 형상은 일치해야 한다. 단위·depth semantics·보정 상태가 알려지지 않으면 추측하지 않고 실패한다. 원본 깊이 0/sentinel/NaN/Inf의 의미는 원본 스키마로 명시해야 한다. 정규화 후 mask=true인 깊이는 finite하고 0보다 커야 한다. 측정 유효 범위는 장비와 촬영 조건에 따라 승인된 설정으로 받는다.

## 좌표·단위 (제안)

- 배열 인덱스 `depth[v,u]`: u는 오른쪽, v는 아래, 픽셀 중심은 정수 인덱스. 정류된 pinhole Depth에서 `Z=depth[v,u]`, `X=(u-cx)Z/fx`, `Y=(v-cy)Z/fy`.
- 카메라 좌표 X=오른쪽, Y=아래, Z=전방; 길이·거리는 m, Point Cloud는 m, 부피는 m³. cm³ 표시가 필요하면 `1 m³ = 10^6 cm³`로 별도 변환.
- mm 원본은 명시된 스케일 `0.001 m/value`로 변환. `uint16`이라는 이유만으로 mm라고 추정하지 않는다.
- robot base/world/IMU 좌표는 별개다. 변환 방향과 외부 보정이 없으면 카메라 좌표만 사용하며 world 정규화 요청은 실패한다.
- 왜곡이 남은 입력은 버전 검증된 보정 경로가 준비될 때까지 거부한다. RGB 정렬과 Depth native intrinsics를 혼용하지 않는다.

## 시간·자세 누락 정책 (제안)

저장 프레임 읽기·카메라 좌표 Point Cloud는 timestamp/pose 누락을 표시하고 허용할 수 있다. 실측 수집의 추적성과 다중 센서 동기화 검증에는 sensor timestamp와 clock 정보가 필수다. world 정규화·다중 시점 정합·자세 의존 계산에는 검증된 자세·외부 보정·동기화가 필수이며 누락/미검증이면 해당 단계가 실패한다. 지원 평면이 카메라 좌표에서 직접 제공되고 계산이 자세를 사용하지 않는 경우에는 자세 없이 부피 계산할 수 있는지 팀장 검토가 필요하다.

누락 시 현재 시각·항등 자세를 대입하지 않는다. 파일 mtime은 촬영 시각이 아니다. sync 허용 오차 수치는 현재 근거가 없으므로 확인 필요다.

## 공통 출력·오류 (제안)

출력 필드: `schema_version,frame_id,source_path,status,stage,volume,point_cloud,quality,errors,warnings,processing_config`.

- `status`: `success` / `failure`. 입력 변환 성공과 부피 계산 성공을 별도 stage로 구분한다.
- `volume`: 성공 시 `{value,unit:"m3",method,assumptions}`; 부피 계산 미요청 또는 실패 시 null. `volume: 0`으로 실패를 숨기지 않는다.
- `point_cloud`: 계산 완료 시 좌표계·단위·점 수·저장 경로 또는 배열 참조. 점 수는 부피가 아니다.
- `quality`: 입력/유효/대상 점 수, invalid 비율, 관측 coverage, 평면·시간 보정 상태, 샘플 provenance. “관측 범위 내 추정”과 실측 정확도 평가를 구분한다.
- `errors`: `{code,field,message}` 목록. CLI가 구현되면 실패 시 0이 아닌 종료 코드와 이 구조를 제안한다.
- `processing_config`: 필터·분리·적분 파라미터, 코드/스키마 버전, 입력 checksum. 재현 가능한 동일 설정을 보존한다.

| 조건 | 제안 오류 코드 |
| --- | --- |
| 파일 없음 / 읽기 실패 | `INPUT_NOT_FOUND` / `INPUT_READ_FAILED` |
| 손상·배열 불일치·지원하지 않는 형식 | `INPUT_CORRUPT` / `SHAPE_MISMATCH` / `UNSUPPORTED_FORMAT` |
| 빈 입력 / 유효 깊이 없음 | `EMPTY_INPUT` / `NO_VALID_DEPTH` |
| 단위·깊이 의미 불명 | `DEPTH_UNIT_UNKNOWN` / `DEPTH_SEMANTICS_UNKNOWN` |
| 내부 파라미터 누락·잘못된 값 | `INTRINSICS_MISSING` / `INTRINSICS_INVALID` |
| 보정 경로 미지원 | `CALIBRATION_UNSUPPORTED` |
| 단계 필수 시각·자세·동기화 누락 | `TIMESTAMP_REQUIRED` / `POSE_REQUIRED` / `SYNC_UNVERIFIED` |
| 마스크·평면·관측 범위·부피 가정 미충족 | `TARGET_UNAVAILABLE` / `SUPPORT_PLANE_REQUIRED` / `INSUFFICIENT_COVERAGE` / `VOLUME_ASSUMPTION_VIOLATED` |

## 문서용 입출력 예제 (합성, 실행 안 함)

이 예제는 측정 데이터나 승인된 정상 샘플이 아니다. ADE-51의 기준/예측 중량(g) 필드는 향후 상위 데이터 구조에서 연결할 위치를 확인해야 하며, 본 모듈은 중량을 계산하지 않는다.

```json
{
  "schema_version": "draft-1",
  "frame_id": "synthetic-2x2-0001",
  "source_path": "mobile_pc_replay",
  "provenance": {"kind": "synthetic", "source": "이 문서의 수작업 예시"},
  "raw_depth": [[1000, 1000], [1000, 0]],
  "raw_unit": "mm",
  "depth_scale_m": 0.001,
  "raw_invalid_values": [0],
  "depth_semantics": "optical_axis_z",
  "intrinsics": {
    "width": 2, "height": 2, "fx": 100, "fy": 100,
    "cx": 0, "cy": 0, "rectified": true,
    "calibration_source": "synthetic"
  },
  "timestamp": null,
  "pose": null
}
```

변환 후 유효점의 기대 좌표는 `[0,0,1]`, `[0.01,0,1]`, `[0,0.01,1]` m, valid mask는 `[[true,true],[true,false]]`다. 배열 무효칸의 수치값은 계산에 사용하지 않는다. 타깃/평면이 없어 이 입력의 부피는 계산할 수 없다.

```json
{
  "schema_version": "draft-1",
  "frame_id": "synthetic-2x2-0001",
  "source_path": "mobile_pc_replay",
  "status": "failure",
  "stage": "volume",
  "volume": null,
  "point_cloud": {"unit": "m", "frame": "camera", "point_count": 3},
  "errors": [{"code": "TARGET_UNAVAILABLE", "field": "target_mask", "message": "대상 영역과 지지 평면 필요"}],
  "warnings": ["촬영 시각·자세 없음; 시간 동기화 검증 미완료"]
}
```

실제 계약 확정 때 전체 스키마·성공 예제·실패 예제에 required/nullable를 명시하고 ADE-51 승인 링크를 함께 기록한다.
