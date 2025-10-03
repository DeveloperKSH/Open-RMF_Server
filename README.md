# RMF Server (RMF Core & Monitoring)

본 저장소는 **Open-RMF 기반 다중 로봇 관제 시스템**에서 **서버(RMF Server)** 측 코드를 담당하는 패키지 모음입니다.  

- **역할**: 다중 로봇의 상태 수집, 작업/경로 스케줄링, 로봇/작업 모니터링 연동  
- **구성**: 서버 코어(RMF Core), 인터페이스(Fleet Manager), 외부 브릿지(MQTT/Socket.IO), 웹 대시보드(RMF Web), 웹 패널(RMF Panel), 시각화 툴(RViz), 맵 편집기(Traffic Editor), Docker 실행 환경  
- **활용**: 실내/실외 로봇의 중앙 관제 및 다중 로봇 운영 최적화  

---

## 📌 1. 필요하게 된 상황
실외/실내 환경에서 여러 대의 로봇을 효율적으로 관리하려면, 개별 로봇만 잘 움직이는 것만으로는 충분하지 않습니다.  
- **중앙 관제 서버**가 모든 로봇의 상태를 실시간으로 수집하고, 충돌 없는 경로를 분배해야 함  
- **웹 대시보드와 연계**하여 운영자가 작업을 명령하고, 실행 상태를 모니터링해야 함  
- 실외/실내 환경의 다양한 제약(장애물, 위험 지역, 층간 이동 등)을 고려해야 함  

---

## 🔧 2. 시스템 구성

- **서버 코어 (`rmf_core`)**
  - 다중 로봇 작업 스케줄링과 경로 계획
  - 맵/교통 그래프(`building.yaml`)와 상태를 바탕으로 충돌 없는 운행 계산
  - 계획/스케줄 및 상태 관련 ROS 토픽 퍼블리시

- **인터페이스 (`fleet_manager`)**
  - 작업/경로 지시 전송, 로봇 상태 수집 및 로봇별 상태 관리
  - 좌표계 변환 지원(ex. 한국 좌표계 `EPSG:5174`)
  - 주요 엔드포인트: `/status`, `/navigate`, `/stop_robot`, `/start_activity`, `/toggle_teleop`, `/sub_robot_state`

- **외부 브릿지 (`rmf_demos_bridges`)**
  - ROS 토픽을 MQTT/Socket.IO로 중계
  - `RobotState`, `FleetState` 등 텔레메트리 데이터를 실시간 웹으로 전달

- **웹 대시보드 (`rmf-web`)**
  - 메인 관제 UI(작업 명령 및 로봇/작업 모니터링)
  - 구성 파일(`main.json`, `dashboard_config.json`)로 접속 엔드포인트 설정

- **웹 패널 (`rmf_demos_panel`)**
  - 경량 보조 패널(작업 명령 및 로봇/작업 모니터링)
  - ROS 디스패처/토픽/서비스 및 내부 WebSocket으로 상태 수신

- **시각화 툴 (`rviz` + `rviz_satellite`)**
  - ROS 네이티브 시각화(지도/로봇/경로/스케줄)
  - 위성 지도 오버레이로 실제 지형과 주행을 매칭

- **맵 편집기 (`rmf_traffic_editor`)**
  - `building.yaml` 제작/편집 도구
  - 실내/실외 맵과 교통 그래프(층/경로/인프라) 정의

- **Docker 환경**  
  - 전체 시스템을 컨테이너로 패키징하여 손쉽게 실행·배포 가능

---

## 🔀 3. 시스템 아키텍처 & 데이터 흐름
```mermaid
flowchart LR
  %% ----- Server Core -----
  subgraph Core["Server"]
    rmfcore["rmf_core"]
    fm["fleet_manager"]
  end

  %% ----- External (robot viewpoint) -----
  subgraph Ext["External"]
    bridge["MQTT / Socket.IO Bridge"]
  end

  %% ----- Monitoring -----
  subgraph Mon["Monitoring"]
    dash["rmf_web"]
    panel["rmf_panel"]
    rviz["rviz(satellite)"]
  end

  %% ----- Robot Clients -----
  subgraph Clients["Client"]
    adapter["fleet_adapter"]
  end

  %% Server internals
  rmfcore <--> fm

  %% Core ↔ External
  rmfcore <--> bridge

  %% Monitoring
  dash <--> bridge
  panel <--> rmfcore
  rviz <--> rmfcore

  %% Server ↔ Robot
  fm -- "PathRequest" --> adapter
  adapter -- "RobotState" --> fm
  ```
