# RMF Server (Fleet Management & Coordination)

본 저장소는 **Open-RMF 기반 다중 로봇 관제 시스템**에서 **서버(RMF Server)** 측 코드를 담당하는 패키지 모음입니다.  

- **역할**: 다중 로봇의 상태 수집, 작업/경로 분배, 태스크 관리 및 대시보드 연동  
- **구성**: Fleet Manager(FastAPI API), FSM 기반 Nav2 제어 연동, 브리지(Flask-SocketIO), 패널 API, Traffic Editor, Docker 실행 환경  
- **활용**: 실내·외 물류 로봇의 중앙 관제 및 다중 로봇 운영 최적화  

---

## 📌 1. 필요하게 된 상황
실외/실내를 오가는 여러 대의 물류 로봇을 효율적으로 관리하려면, 개별 로봇만 잘 움직이는 것만으로는 충분하지 않습니다.  
- **중앙 관제 서버**가 모든 로봇의 상태를 실시간으로 수집하고, 충돌 없는 경로를 분배해야 함  
- **웹 대시보드와 연계**하여 운영자가 태스크를 제출하고, 실행 상태를 모니터링해야 함  
- 실외/실내 환경의 다양한 제약(장애물, 지도, 층간 이동 등)을 고려해야 함  

---

## 🔧 2. 시스템 구성
서버단(RMF Server)은 다음과 같은 주요 구성 요소로 동작합니다:

- **Fleet Manager (`rmf_demos_fleet_adapter`)**  
  - FastAPI 기반 REST API 제공  
  - 로봇 상태 조회(`/status`), 경로 명령(`/navigate`), 액티비티 실행(`/start_activity`) 등 지원  
  - 좌표계 변환 및 로봇별 상태 관리  

- **FSM Waypoint (`fsm_waypoint`)**  
  - Nav2 주행 액션과 RMF 태스크를 연결하는 상태 기계  
  - 목표 Pose 실행, 취소, 재시작 관리  

- **Bridges (`rmf_demos_bridges`)**  
  - Flask-SocketIO 기반  
  - ROS2 ↔ 웹 대시보드 간 상태 동기화 (RobotState, TaskSummary 등)  

- **Panel API (`rmf_demos_panel`)**  
  - Flask 기반 경량 API  
  - 태스크 제출, 취소, 맵 조회 등 보조 기능  

- **Traffic Editor (`rmf_traffic_editor`)**  
  - building.yaml 생성 및 편집 도구  
  - 실제 건물/캠퍼스 맵(`handong.building.yaml`, `clinic.building.yaml` 등) 포함  

- **Docker 환경**  
  - `docker-compose.yml`을 통한 API 서버, 대시보드, 패널, RMF core 일괄 실행  

---

## 🔀 3. 시스템 아키텍처 & 데이터 흐름
```mermaid
flowchart LR
  %% ----- Core Layer -----
  subgraph CoreLayer["Core Layer / Server"]
    Core["RMF Core"]
    FM["Fleet Manager (FastAPI)"]
    Bridge["Bridges (Socket.IO / WS)"]
  end

  %% ----- Monitoring -----
  subgraph Mon["Monitoring & Control"]
    Dash["RMF Web Dashboard"]
    Panel["RMF Panel"]
    RViz["RViz (with Satellite)"]
  end

  %% ----- Clients -----
  subgraph Clients["Robot Clients (rmf_robot)"]
    Adapter["fleet_adapter"]
  end

  %% Server internals
  Core <--> FM
  Core <--> Bridge

  %% Monitoring
  Dash <--> Bridge
  Panel <--> FM
  RViz <--> Core

  %% Server ↔ Robot
  FM <--> Adapter
  ```
