# RMF Server (Fleet Management & Coordination)

본 저장소는 **Open-RMF 기반 다중 로봇 관제 시스템**에서 **서버(RMF Server)** 측 코드를 담당하는 패키지 모음입니다.  

- **역할**: 다중 로봇의 상태 수집, 작업/경로 분배, 작업 관리 및 대시보드 연동  
- **구성**: Fleet Manager(FastAPI API), FSM 기반 Nav2 제어 연동, 브리지(Flask-SocketIO), 패널 API, Traffic Editor, Docker 실행 환경  
- **활용**: 실내·외 물류 로봇의 중앙 관제 및 다중 로봇 운영 최적화  

---

## 📌 1. 필요하게 된 상황
실외/실내 환경에서 여러 대의 로봇을 효율적으로 관리하려면, 개별 로봇만 잘 움직이는 것만으로는 충분하지 않습니다.  
- **중앙 관제 서버**가 모든 로봇의 상태를 실시간으로 수집하고, 충돌 없는 경로를 분배해야 함  
- **웹 대시보드와 연계**하여 운영자가 작업을 명령하고, 실행 상태를 모니터링해야 함  
- 실외/실내 환경의 다양한 제약(장애물, 지도, 층간 이동 등)을 고려해야 함  

---

## 🔧 2. 시스템 구성

- **RMF Core (`rmf_core`)**
  - 다중 로봇 작업 스케줄링과 경로 계획(시스템의 두뇌)
  - 맵/교통 그래프(`building.yaml`)와 상태를 바탕으로 충돌 없는 운행 계산
  - 계획/스케줄·상태 관련 ROS 토픽 퍼블리시

- **Fleet Manager (`rmf_demos_fleet_adapter`)**
  - 서버↔로봇 허브(FASTAPI 기반)
  - PathRequest·액티비티 지시 전송, 로봇 상태 수집 및 로봇별 상태 관리
  - 좌표계 변환 지원(멀티맵/실외·실내 정합)
  - 주요 엔드포인트: `/status`, `/navigate`, `/stop_robot`, `/start_activity`, `/toggle_teleop`, `/sub_robot_state`

- **Bridges (`rmf_demos_bridges`)**
  - ROS 토픽을 Socket.IO 이벤트로 중계
  - `RobotState`, `FleetState` 등 실시간 스트림을 웹으로 전달
  - 선택적 GPS/좌표 변환 지원

- **RMF Web Dashboard (`rmf-web`)**
  - 메인 관제 UI(태스크 제출/취소·조회)
  - 브리지의 실시간 스트림을 구독하여 상태/경로 모니터링
  - 구성 파일(`main.json`, `dashboard_config.json`)로 접속 엔드포인트 설정

- **RMF Panel (`rmf_demos_panel`)**
  - 경량 보조 패널(모니터링 + 제어)
  - ROS 디스패처/토픽/서비스 및 내부 WebSocket으로 상태 수신
  - 수신 데이터를 Socket.IO로 브라우저에 푸시, 태스크 제출/취소·맵 조회 제공

- **RMF RViz Visualizer (`rviz` + `rviz_satellite`)**
  - ROS 네이티브 시각화(맵/포즈/경로/상태)
  - 위성 지도 오버레이로 실제 지형과 주행을 매칭
  - 디버깅/운영 보조용 뷰 제공

- **Traffic Editor (`rmf_traffic_editor`)**
  - `building.yaml` 제작·편집 도구
  - 실내·외/캠퍼스 지도와 교통 그래프(레벨/레인/도어/리프트) 정의

- **Docker/Compose 실행 환경**
  - `rmf_core`·Fleet Manager·Bridges·Dashboard·Panel·RViz 일괄 구동
  - `.env` 및 구성 파일로 포트/네트워크/도메인 설정

---

## 🔀 3. 시스템 아키텍처 & 데이터 흐름
```mermaid
flowchart LR
  %% ===== Server Core =====
  subgraph Core["RMF Server Core"]
    rmfcore["rmf_core"]
    fm["Fleet Manager (FastAPI)"]
    bridge["Bridges (Socket.IO)"]
  end

  %% ===== Monitoring & Control =====
  subgraph Mon["Monitoring & Control"]
    dash["RMF Web Dashboard"]
    panel["RMF Panel (Flask)"]
    rviz["RViz (Satellite)"]
  end

  %% ===== Robot Clients =====
  subgraph Clients["Robot Clients (rmf_robot)"]
    adapter["fleet_adapter"]
  end

  %% Server internals
  rmfcore <--> fm
  rmfcore <--> bridge

  %% Monitoring paths (간결/일관)
  dash <--> bridge        
  panel <--> rmfcore      
  rviz <--> rmfcore       

  %% Server ↔ Robot
  fm -- "PathRequest" --> adapter
  adapter -- "RobotState" --> fm
  ```
