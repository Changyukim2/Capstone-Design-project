# 실시간 낙상 감지 시스템

Django 기반의 병동 낙상 감지 대시보드입니다. 카메라 영상에서 MediaPipe Pose로 관절 좌표를 추출하고, GRU 기반 PyTorch 모델로 낙상 여부와 충격 부위를 판단합니다. 감지된 이벤트는 웹 화면, SSE 스트림, 알림 목록에 실시간으로 반영됩니다.

## 주요 기능

- 실시간 카메라 스트림 기반 낙상 감지
- 머리, 골반, 손목 등 충격 부위별 위험도 분류
- 낙상 알림 생성, 읽음 처리, 위험도 통계 제공
- 환자 낙상 이력 등록, 필터링, CSV 내보내기
- 병동 담당자 회원가입, 로그인, 로그아웃, 마이페이지
- 프라이버시 모드 지원: 원본 영상 대신 스켈레톤 중심 표시

## 기술 스택

| 구분 | 기술 |
| --- | --- |
| Backend | Django 4, Django Channels |
| AI/영상 처리 | PyTorch, OpenCV, MediaPipe, NumPy |
| Realtime | SSE, StreamingHttpResponse |
| Database | SQLite, Django ORM |
| 기타 | pygame, Docker |

## 빠른 실행

### 1. 가상환경 생성

```bash
python3 -m venv venv
source venv/bin/activate
```

Windows에서는 다음 명령을 사용합니다.

```bash
venv\Scripts\activate
```

### 2. 패키지 설치

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

### 3. DB 초기화 및 서버 실행

```bash
python manage.py migrate
python manage.py runserver
```

접속 주소:

- 로그인: `http://127.0.0.1:8000/member/login/`
- 낙상 감지 화면: `http://127.0.0.1:8000/member/fall_prevention/`
- 관리자: `http://127.0.0.1:8000/admin/`

관리자 계정이 필요하면 다음 명령으로 생성합니다.

```bash
python manage.py createsuperuser
```

## Docker 실행

```bash
docker build -t fall-monitor .
docker run --rm -it --net=host --device=/dev/video0 fall-monitor
```

카메라 장치 경로는 환경에 따라 달라질 수 있습니다. macOS나 Windows에서는 Docker의 카메라 접근 방식이 다르므로 로컬 실행을 우선 권장합니다.

## 프로젝트 구조

```text
.
|-- config/                  Django 프로젝트 설정
|-- fall/                    실시간 낙상 감지, 모델 추론, 알림 스트림
|   |-- fall_temporal_hybrid_best.pth
|   |-- feature_stats.json
|   |-- model_gru.py
|   |-- views.py
|   `-- models.py
|-- member/                  회원, 낙상 기록, 사용자 로그
|-- templates/               화면 템플릿
|-- static/                  CSS, JS, 이미지
|-- fall_temporal_hybrid_model.py
|-- manage.py
|-- requirements.txt
`-- Dockerfile
```

저장소에는 실행에 필요한 소스 코드와 모델 파일만 포함합니다. 가상환경, 캐시, SQLite DB, 학습용 CSV, 중복 모델 파일은 `.gitignore`에서 제외합니다.

## 시스템 흐름

1. Django 서버가 실행됩니다.
2. `fall.apps.FallConfig.ready()`에서 카메라 처리 스레드가 시작됩니다.
3. OpenCV가 프레임을 읽고 MediaPipe Pose가 관절 좌표를 추출합니다.
4. 좌표와 속도 피처를 `feature_stats.json` 기준으로 정규화합니다.
5. `FallTemporalHybridNet` 모델이 낙상 여부와 충격 부위를 추론합니다.
6. 낙상으로 판단되면 `FallAlert`가 생성되고 알림 화면과 SSE로 전달됩니다.
7. 담당자는 대시보드에서 알림과 낙상 이력을 확인합니다.

## 주요 URL

| URL | 설명 |
| --- | --- |
| `/member/reg/` | 회원가입 |
| `/member/login/` | 로그인 |
| `/member/logout/` | 로그아웃 |
| `/member/mypage/` | 마이페이지 |
| `/member/fall_prevention/` | 실시간 낙상 감지 화면 |
| `/fall/pose_feed/` | MJPEG 영상 스트림 |
| `/fall/fall_status/` | 현재 낙상 감지 상태 JSON |
| `/fall/toggle_privacy/` | 프라이버시 모드 전환 |
| `/fall/sse/fall_alert/` | 실시간 낙상 알림 SSE |
| `/member/fall/list/` | 낙상 이력 목록 |
| `/member/fall/add/` | 낙상 이력 수동 등록 |
| `/member/fall/export/` | 낙상 이력 CSV 내보내기 |
| `/member/fall/alert/list/` | 낙상 알림 목록 |

## 데이터 모델

| 모델 | 역할 |
| --- | --- |
| `Member` | 병동 담당자 계정 정보 |
| `UserLog` | 회원가입, 로그인, 로그아웃 기록 |
| `FallRecord` | 환자 낙상 이력 |
| `FallAlert` | 실시간 모델이 생성한 낙상 알림 |

## 낙상 감지 모델

모델 파일은 `fall/fall_temporal_hybrid_best.pth`에 있으며, 정규화 통계는 `fall/feature_stats.json`을 사용합니다.

- 네트워크: Temporal Conv1d + Bi-GRU + Attention
- 입력: 선택 관절 좌표와 속도 기반 30차원 시퀀스
- 안정화: 슬라이딩 윈도우 다수결과 골반 높이 하락 검증 적용
- 위험도: 충격 부위 기준으로 경미, 중간, 심각 단계 분류

## 개발 메모

- 기본 DB는 SQLite입니다. 실행 중 생성되는 `*.sqlite3` 파일은 Git에 포함하지 않습니다.
- 기본 카메라는 OpenCV의 `VideoCapture(0)`을 사용합니다.
- 카메라 인덱스가 맞지 않으면 `fall/views.py`의 카메라 설정을 환경에 맞게 조정해야 합니다.
- 모델 재학습용 데이터와 실험 결과물은 저장소에 올리지 않고 별도로 관리합니다.
