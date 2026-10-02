# หลักสูตร Erlang ครบวงจร: จากพื้นฐานสู่ระดับโลก
## Complete Erlang Course: From Beginner to World-Class

> **สอนเขียนและพัฒนาโปรแกรมและเว็บแอพพลิเคชันด้วย Erlang**
> ระบบ distributed ที่เสถียรมาก — ตั้งแต่พื้นฐานจนถึงระดับโลก

---

## เกี่ยวกับหลักสูตรนี้

หลักสูตรนี้ออกแบบมาเพื่อสอน Erlang ตั้งแต่ระดับเริ่มต้นไปจนถึงระดับมืออาชีพและระดับโลก  
เนื้อหาแต่ละ Part มีโค้ดที่ใช้งานได้จริง 100% พร้อมคำอธิบายภาษาไทยอย่างละเอียด

### ทำไมต้องเรียน Erlang?

- **Fault-tolerant**: ระบบทนต่อความผิดพลาดได้ดีที่สุดในโลก
- **Distributed**: รองรับ distributed computing ได้แบบ native
- **Concurrent**: ใช้ lightweight process นับล้านตัวพร้อมกัน
- **Hot Code Reloading**: อัปเดตโค้ดโดยไม่ต้องหยุดระบบ
- **Battle-tested**: WhatsApp รองรับ 2 พันล้าน user ด้วย Erlang
- **Nine Nines**: บางระบบมี uptime 99.9999999% (ไม่หยุดทำงาน 31 ปี!)

---

## โครงสร้างหลักสูตร

### 🟢 ระดับพื้นฐาน (Parts 1-30)

| Part | หัวข้อ | สถานะ |
|------|--------|--------|
| [Part 01](parts/part_01_introduction.md) | บทนำ Erlang และประวัติศาสตร์ | ✅ |
| [Part 02](parts/part_02_installation.md) | การติดตั้งและตั้งค่าสภาพแวดล้อม | ✅ |
| [Part 03](parts/part_03_basic_syntax.md) | Syntax พื้นฐานของ Erlang | ✅ |
| [Part 04](parts/part_04_data_types.md) | ชนิดข้อมูล (Data Types) | ✅ |
| [Part 05](parts/part_05_variables_pattern_matching.md) | Variables และ Pattern Matching | ✅ |
| [Part 06](parts/part_06_functions.md) | Functions และ Function Clauses | ✅ |
| [Part 07](parts/part_07_modules.md) | Modules และการจัดระเบียบโค้ด | ✅ |
| [Part 08](parts/part_08_lists.md) | Lists และการดำเนินการกับ Lists | ✅ |
| [Part 09](parts/part_09_tuples_records.md) | Tuples และ Records | ✅ |
| [Part 10](parts/part_10_maps.md) | Maps - โครงสร้างข้อมูล Key-Value | ✅ |
| [Part 11](parts/part_11_strings_binaries.md) | Strings และ Binaries | ✅ |
| [Part 12](parts/part_12_arithmetic.md) | การคำนวณและ Arithmetic Operations | ✅ |
| [Part 13](parts/part_13_control_flow.md) | Control Flow: if, case, cond | ✅ |
| [Part 14](parts/part_14_guards.md) | Guards - การตรวจสอบเงื่อนไข | ✅ |
| [Part 15](parts/part_15_recursion.md) | Recursion - การเรียกซ้ำ | ✅ |
| [Part 16](parts/part_16_higher_order_functions.md) | Higher-Order Functions | ✅ |
| [Part 17](parts/part_17_list_comprehensions.md) | List Comprehensions | ✅ |
| [Part 18](parts/part_18_bifs.md) | Built-in Functions (BIFs) | ✅ |
| [Part 19](parts/part_19_exception_handling.md) | Exception Handling | ✅ |
| [Part 20](parts/part_20_file_io.md) | File I/O - การอ่านและเขียนไฟล์ | ✅ |
| [Part 21](parts/part_21_processes_intro.md) | Processes - บทนำ | ✅ |
| [Part 22](parts/part_22_message_passing.md) | Message Passing | ✅ |
| [Part 23](parts/part_23_process_links_monitors.md) | Process Links และ Monitors | ✅ |
| [Part 24](parts/part_24_ets_tables.md) | ETS Tables - In-Memory Database | ✅ |
| [Part 25](parts/part_25_otp_introduction.md) | OTP - Introduction | ✅ |
| [Part 26](parts/part_26_genserver.md) | GenServer | ✅ |
| [Part 27](parts/part_27_supervisor.md) | Supervisor | ✅ |
| [Part 28](parts/part_28_application.md) | Application Behaviour | ✅ |
| [Part 29](parts/part_29_rebar3.md) | Rebar3 - Build Tool | ✅ |
| [Part 30](parts/part_30_testing_eunit.md) | Testing ด้วย EUnit | ✅ |

### 🔵 ระดับกลาง (Parts 31-60)

| Part | หัวข้อ | สถานะ |
|------|--------|--------|
| [Part 31](parts/part_31_distributed_erlang.md) | Distributed Erlang - บทนำ | ✅ |
| [Part 32](parts/part_32_node_communication.md) | Node Communication | ✅ |
| [Part 33](parts/part_33_mnesia_intro.md) | Mnesia - Distributed Database | ✅ |
| [Part 34](parts/part_34_gen_tcp.md) | Networking ด้วย gen_tcp | ✅ |
| [Part 35](parts/part_35_gen_udp.md) | Networking ด้วย gen_udp | ✅ |
| [Part 36](parts/part_36_cowboy_http.md) | HTTP Server ด้วย Cowboy | ✅ |
| [Part 37](parts/part_37_rest_api.md) | REST API ด้วย Erlang | ✅ |
| [Part 38](parts/part_38_websockets.md) | WebSockets | ✅ |
| [Part 39](parts/part_39_json_handling.md) | JSON Handling | ✅ |
| [Part 40](parts/part_40_database_postgresql.md) | PostgreSQL ด้วย Erlang | ✅ |
| [Part 41](parts/part_41_common_test.md) | Common Test Framework | ✅ |
| [Part 42](parts/part_42_proper_testing.md) | PropEr - Property-Based Testing | ✅ |
| [Part 43](parts/part_43_debugging.md) | Debugging Techniques | ✅ |
| [Part 44](parts/part_44_profiling.md) | Profiling และ Performance | ✅ |
| [Part 45](parts/part_45_hot_code_reloading.md) | Hot Code Reloading | ✅ |
| [Part 46](parts/part_46_releases.md) | Release Management | ✅ |
| [Part 47](parts/part_47_logging.md) | Logging ด้วย Logger | ✅ |
| [Part 48](parts/part_48_configuration.md) | Configuration Management | ✅ |
| [Part 49](parts/part_49_ports_nifs.md) | Ports และ NIFs | ✅ |
| [Part 50](parts/part_50_binary_protocols.md) | Binary Protocols | ✅ |
| [Part 51](parts/part_51_gen_statem.md) | gen_statem - State Machines | ✅ |
| [Part 52](parts/part_52_gen_event.md) | gen_event - Event Manager | ✅ |
| [Part 53](parts/part_53_supervision_trees.md) | Supervision Trees ขั้นสูง | ✅ |
| [Part 54](parts/part_54_ets_advanced.md) | ETS Tables ขั้นสูง | ✅ |
| [Part 55](parts/part_55_mnesia_advanced.md) | Mnesia ขั้นสูง | ✅ |
| [Part 56](parts/part_56_security.md) | Security และ TLS | ✅ |
| [Part 57](parts/part_57_metrics_telemetry.md) | Metrics และ Telemetry | ✅ |
| [Part 58](parts/part_58_health_checks.md) | Health Checks | ✅ |
| [Part 59](parts/part_59_rate_limiting.md) | Rate Limiting | ✅ |
| [Part 60](parts/part_60_caching.md) | Caching Strategies | ✅ |

### 🔴 ระดับสูง (Parts 61-100)

| Part | หัวข้อ | สถานะ |
|------|--------|--------|
| [Part 61](parts/part_61_actor_model_deep_dive.md) | Actor Model ขั้นลึก | ✅ |
| [Part 62](parts/part_62_fault_tolerance_patterns.md) | Fault Tolerance Patterns | ✅ |
| [Part 63](parts/part_63_scalability_patterns.md) | Scalability Patterns | ✅ |
| [Part 64](parts/part_64_cluster_management.md) | Cluster Management | ✅ |
| [Part 65](parts/part_65_service_discovery.md) | Service Discovery | ✅ |
| [Part 66](parts/part_66_circuit_breakers.md) | Circuit Breakers | ✅ |
| [Part 67](parts/part_67_event_sourcing.md) | Event Sourcing | ✅ |
| [Part 68](parts/part_68_cqrs.md) | CQRS Pattern | ✅ |
| [Part 69](parts/part_69_real_time_systems.md) | Real-time Systems | ✅ |
| [Part 70](parts/part_70_chat_application.md) | Chat Application ครบวงจร | ✅ |
| [Part 71](parts/part_71_job_queues.md) | Job Queues | ✅ |
| [Part 72](parts/part_72_stream_processing.md) | Stream Processing | ✅ |
| [Part 73](parts/part_73_grpc.md) | gRPC ด้วย Erlang | ✅ |
| [Part 74](parts/part_74_graphql.md) | GraphQL | ✅ |
| [Part 75](parts/part_75_microservices.md) | Microservices Architecture | ✅ |
| [Part 76](parts/part_76_docker_deployment.md) | Docker Deployment | ✅ |
| [Part 77](parts/part_77_kubernetes.md) | Kubernetes Deployment | ✅ |
| [Part 78](parts/part_78_cicd.md) | CI/CD Pipeline | ✅ |
| [Part 79](parts/part_79_monitoring_observability.md) | Monitoring และ Observability | ✅ |
| [Part 80](parts/part_80_chaos_engineering.md) | Chaos Engineering | ✅ |
| [Part 81](parts/part_81_zero_downtime.md) | Zero-Downtime Deployment | ✅ |
| [Part 82](parts/part_82_multi_region.md) | Multi-Region Setup | ✅ |
| [Part 83](parts/part_83_iot_erlang.md) | IoT ด้วย Erlang | ✅ |
| [Part 84](parts/part_84_game_server.md) | Game Server Development | ✅ |
| [Part 85](parts/part_85_telecom_systems.md) | Telecommunication Systems | ✅ |
| [Part 86](parts/part_86_financial_systems.md) | Financial Systems | ✅ |
| [Part 87](parts/part_87_notification_systems.md) | Notification Systems | ✅ |
| [Part 88](parts/part_88_data_pipeline.md) | Data Pipeline | ✅ |
| [Part 89](parts/part_89_event_driven_arch.md) | Event-Driven Architecture | ✅ |
| [Part 90](parts/part_90_ddd.md) | Domain-Driven Design | ✅ |
| [Part 91](parts/part_91_performance_tuning.md) | Performance Tuning ขั้นสูง | ✅ |
| [Part 92](parts/part_92_memory_management.md) | Memory Management | ✅ |
| [Part 93](parts/part_93_scheduler_optimization.md) | Scheduler Optimization | ✅ |
| [Part 94](parts/part_94_network_optimization.md) | Network Optimization | ✅ |
| [Part 95](parts/part_95_whatsapp_architecture.md) | กรณีศึกษา: WhatsApp Architecture | ✅ |
| [Part 96](parts/part_96_erricsson_case_study.md) | กรณีศึกษา: Ericsson Telecom | ✅ |
| [Part 97](parts/part_97_discord_case_study.md) | กรณีศึกษา: Discord Scaling | ✅ |
| [Part 98](parts/part_98_production_best_practices.md) | Production Best Practices | ✅ |
| [Part 99](parts/part_99_world_class_patterns.md) | World-Class System Patterns | ✅ |
| [Part 100](parts/part_100_mastery_project.md) | Mastery Project: ระบบครบวงจร | ✅ |

---

## วิธีใช้หลักสูตรนี้

### เส้นทางการเรียน (Learning Path)

```
[Part 01-05] → พื้นฐาน Syntax
      ↓
[Part 06-15] → Functions, Modules, Control Flow
      ↓
[Part 16-20] → Lists, BIFs, Exception, I/O
      ↓
[Part 21-28] → Processes, OTP, GenServer, Supervisor
      ↓
[Part 29-30] → Build Tools, Testing
      ↓
[Part 31-50] → Distributed, Networking, Databases
      ↓
[Part 51-60] → Advanced OTP, Security, Monitoring
      ↓
[Part 61-80] → Patterns, Architecture, Deployment
      ↓
[Part 81-100] → World-Class Systems, Case Studies
```

### Prerequisites

- ความรู้ programming พื้นฐาน (ภาษาอะไรก็ได้)
- ความเข้าใจ command line พื้นฐาน
- ความอยากรู้อยากเห็น!

---

## สภาพแวดล้อมที่แนะนำ

```bash
# Erlang OTP 26+
$ erl -version
Erlang/OTP 26

# Rebar3 (Build Tool)
$ rebar3 --version
rebar 3.22.0

# Editor: VS Code + vscode-erlang extension
# หรือ Emacs + erlang-mode
```

---

## ลิขสิทธิ์และการใช้งาน

หลักสูตรนี้สร้างขึ้นเพื่อการศึกษา สามารถนำไปใช้ได้อย่างอิสระ

---

*หลักสูตรนี้อัปเดตต่อเนื่อง — ติดตามการเพิ่มเนื้อหาใหม่ได้จาก repository นี้*
