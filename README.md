# SPI & I2C Master IP (AXI4-Lite)

> AXI4-Lite 기반 SPI / I2C Master IP 설계 및 UVM 검증 프로젝트  
> Basys3 (Xilinx Artix-7) 두 보드 간 실제 통신 동작 확인

---

## 📌 프로젝트 개요

AXI4-Lite 인터페이스를 통해 CPU(MicroBlaze)가 레지스터를 제어하는 방식으로  
**SPI Master IP** 및 **I2C Master IP**를 설계하고 UVM 환경에서 검증한 프로젝트입니다.  
2개의 Basys3 보드를 Master / Slave로 구성하여 실제 Write / Read 통신을 구현하였습니다.

---

## 🎯 구현 목표

- AXI4-Lite 기반 SPI / I2C Master IP 설계 (SystemVerilog)
- Vivado Block Design을 활용한 MicroBlaze SoC 구성
- C 코드(MicroBlaze)로 AXI 레지스터에 Write / Read하여 동작 제어
- 2개의 보드(Master / Slave) 간 실제 통신 검증
- SPI IP에 대한 UVM 검증 환경 구축 (Coverage 100%)

---

## 🏗️ 시스템 구조

### AXI4-Lite 개요

```
AXI : Bus가 아닌 Point-to-Point Interface → Signal과 Timing만 맞추면 됨

  ┌───────────────────────────────────────────────────┐
  │  APB / AHB Bus          vs        AXI Interface   │
  │  Broadcasting 전송              Point-to-Point 연결 │
  │  Decoder + Mux 필요            Interconnect 라우팅  │
  │  Chip Select 개념 있음          Chip Select 없음    │
  └───────────────────────────────────────────────────┘
```

AXI4-Lite는 AXI4의 경량 버전으로, 레지스터 제어에 특화되어 있습니다.  
한 번에 1개의 데이터를 전송(burst 없음)하며 구현이 단순합니다.  
5개의 독립 채널(AW · W · B · AR · R)로 Write / Read를 동시에 처리할 수 있습니다.

---

## 📡 SPI

### SPI + AXI Block Diagram

```
  MicroBlaze          AXI Slave                   SPI Master       SPI Slave
  ┌────────┐   ┌──────────────────────┐   ┌──────────────────┐   ┌──────────┐
  │   AW ◄─┼──►│ 0x00 [start|cpha|cpol|clk_div] ──► clk_div │   │          │
  │    W ◄─┼──►│ 0x04      [tx_data[7:0]]       ──► tx_data  │──►│   SCLK   │
  │    B ◄─┼──►│ 0x08 [done|busy|rx_data[7:0]]  ◄── rx_data  │◄──│   MOSI   │
  │   AR ◄─┼──►│ 0x0c                           ◄── busy/done│   │   MISO   │
  │    R ◄─┼──►│                                              │   │   CS_n   │
  └────────┘   └──────────────────────────────────────────────┘   └──────────┘
```

### SPI 레지스터 맵

| 주소  | 필드 | 설명 |
|-------|------|------|
| 0x00 | `start[10]` `cpha[9]` `cpol[8]` `clk_div[7:0]` | 제어 레지스터 |
| 0x04 | `tx_data[7:0]` | 송신 데이터 |
| 0x08 | `done[9]` `busy[8]` `rx_data[7:0]` | 상태 / 수신 데이터 |
| 0x0c | — | 예약 |

### SPI 소프트웨어 스택

| 계층 | 내용 |
|------|------|
| **AP** | 버튼 → SPI 전송 / SW → LED 반영 |
| **Driver** | `SPI_Transfer` / `Button_GetState` / `LED_WriteData` / `SW_ReadData` |
| **HAL** | Register 접근해서 PIN Control |
| **HW** | AXI4-Lite Slave IP |

### Microblaze + AXI + SPI + GPIO Block Diagram

```
  sys_clock ──► ┌────────────┐     ┌──────────────┐   ┌──────────────┐ ──► SCLK
  reset     ──► │ MicroBlaze │ ──► │     AXI      │──►│ CR / TX / RX │    MOSI
                └────────────┘     │ InterConnect │   │  SPI Master  │◄── MISO
                                   │              │──►│    GPIOA     │◄── BTN  ──► CS_n
                                   │              │──►│    GPIOB     │◄── SW
                                   │              │──►│    GPIOC     │──► LED
                                   └──────────────┘   └──────────────┘
```

---

## 🔄 I2C

### I2C + AXI Block Diagram

```
  MicroBlaze          AXI Slave                         I2C Master    I2C Slave
  ┌────────┐   ┌───────────────────────────────┐   ┌──────────────┐  ┌──────────┐
  │   AW ◄─┼──►│ 0x00 [cmd_stop|cmd_read|      │   │ cmd_start    │  │          │
  │    W ◄─┼──►│       cmd_write|cmd_start]    │──►│ cmd_write    │  │   SCL    │
  │    B ◄─┼──►│ 0x04 [ack_in|tx_data[7:0]]   │──►│ cmd_read     │  │          │
  │   AR ◄─┼──►│ 0x08 [done|busy|ack_out|      │──►│ cmd_stop     │  │   SDA    │
  │    R ◄─┼──►│       rx_data[7:0]]           │◄──│ rx_data      │  └──────────┘
  └────────┘   │ 0x0c                          │◄──│ ack_out/busy │
               └───────────────────────────────┘   └──────────────┘
```

### I2C 레지스터 맵

| 주소  | 필드 | 설명 |
|-------|------|------|
| 0x00 | `cmd_stop[3]` `cmd_read[2]` `cmd_write[1]` `cmd_start[0]` | 명령 레지스터 |
| 0x04 | `ack_in[8]` `tx_data[7:0]` | 송신 데이터 / ACK 입력 |
| 0x08 | `done[10]` `busy[9]` `ack_out[8]` `rx_data[7:0]` | 상태 / 수신 데이터 |
| 0x0c | — | 예약 |

### I2C 소프트웨어 스택

| 계층 | 내용 |
|------|------|
| **AP** | 버튼 → I2C 전송 / SW → LED 반영 |
| **Driver** | `I2C_Write_Execute` / `I2C_Read_Execute` / `Button_GetState` / `LED_WriteData` / `SW_ReadData` |
| **HAL** | Register 접근해서 PIN Control / Command 신호 / Wait_Done |
| **HW** | AXI4-Lite Slave IP |

### Microblaze + AXI + I2C + GPIO Block Diagram

```
  sys_clock ──► ┌────────────┐     ┌──────────────┐   ┌─────────────────┐ ◄──► SDA
  reset     ──► │ MicroBlaze │ ──► │     AXI      │──►│ COMMAND/TX/RX   │ ──►  SCL
                └────────────┘     │ InterConnect │   │   I2C Master    │
                                   │              │──►│     GPIOA       │◄── BTN
                                   │              │──►│     GPIOB       │◄── SW
                                   │              │──►│     GPIOC       │──► LED
                                   └──────────────┘   └─────────────────┘
```

---

## ✅ SPI UVM 검증

### UVM 환경 구성

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  test                                                           │
  │  ┌──────────────────────────────────────────────┐              │
  │  │  env                                         │              │
  │  │  ┌───────────┐  ┌────────────────────────┐   │ ┌──────────┐ │
  │  │  │   agent   │  │       Scoreboard       │   │ │ Coverage │ │
  │  │  │┌─────────┐│  │ tx_data_m == rx_data_s │   │ │cp_tx_m{} │ │
  │  │  ││Sequencer││  │ tx_data_s == rx_data_m │   │ │cp_tx_s{} │ │
  │  │  │├─────────┤│  │ PASS / FAIL count      │   │ └──────────┘ │
  │  │  ││ Monitor ││  └────────────────────────┘   │              │
  │  │  │├─────────┤│  1. AXI W채널 감시             │              │
  │  │  ││ Driver  ││     awaddr==0x04 감지          │              │
  │  │  │└─────────┘│     tx_data 추적              │              │
  │  │  └───────────┘  2. busy=0→1 감지             │              │
  │  │                    rx_data 캡처              │              │
  │  └──────────────────────────────────────────────┘              │
  │  spi_if (interface)                                             │
  │  DUT : axi_spi_top (AXI4-Lite SPI Master + SPI Slave)          │
  └─────────────────────────────────────────────────────────────────┘
```

### 검증 시나리오

| 시나리오 | 검증 내용 |
|----------|-----------|
| **시나리오 1** | Write 채널로 tx 레지스터에 기록한 값 == SPI 통신을 통해 Slave에 수신된 값 |
| **시나리오 2** | Slave가 MISO로 송신한 값 == SPI 통신을 통해 Read 채널로 수신된 값 |

### Coverage 결과

| 항목 | 결과 |
|------|------|
| Total Transaction | 2,560 |
| Overall Coverage | **100.0%** |
| `cp_tx_data_m` | 100.0% (21 bins) |
| `cp_tx_data_s` | 100.0% (21 bins) |
| Scoreboard PASS | **2,560** |
| Scoreboard FAIL | **0** |

주요 커버리지 빈: `0x55` (alt_01), `0xAA` (alt_10), `0x01` (lsb_only), `0x80` (msb_only), `0x00` (zero), range0~rangef

---

## 🖥️ 개발 환경

| 항목 | 내용 |
|------|------|
| HDL | SystemVerilog |
| EDA Tool | Xilinx Vivado |
| 타겟 보드 | Basys3 (Xilinx Artix-7) |
| 소프트 CPU | MicroBlaze |
| 시뮬레이터 | Vivado Simulator (XSim) |
| 인터페이스 | AXI4-Lite |

---

## 🐛 Trouble Shooting

### 1. I2C Done 신호 포착 문제

**문제**: I2C 명령어 처리가 완료된 시점을 소프트웨어에서 정확히 감지하지 못해 다음 명령어가 중복 실행됨.

**원인 분석**:
- SPI의 경우 `done` 신호 대신 `busy` 신호의 폴링으로 완료 판단이 가능했으나,
- I2C IP는 `done` 신호가 다음 명령어 쓰기 시점까지 유지되는 특성이 있었음.

**해결**: `done` 신호를 확인한 직후 COMMAND 레지스터를 `0x00`으로 클리어하는 방식으로 1클럭 펄스를 명시적으로 처리.

```c
void I2C_CMD_START(I2C_Typedef_t *I2Cx) {
    I2Cx->COMMAND = (1 << I2C_CMD_START_BIT);
    I2C_Wait_Done(I2Cx);
    I2Cx->COMMAND = 0x00;  // done 확인 후 클리어
}
```

### 2. UART printf로 인한 SPI 타이밍 오류

**문제**: 디버깅 목적으로 삽입한 `UART printf`가 SPI 전송 타이밍을 지연시켜 오히려 통신 오류를 유발함.

**원인**: UART 출력 지연이 `busy` 폴링 루프 사이에 끼어들어 SPI 타이밍 제약 위반 발생.

**해결**: 디버깅 단계에서 printf를 제거하고, 통신 완료 후에만 LED 출력으로 상태 확인. 관측 행위 자체가 시스템 타이밍에 영향을 준다는 점을 재확인.

---

## 📄 느낀점

- 디버깅 수단(UART printf)이 오히려 SPI 타이밍을 밀리게 해 오류를 유발한 경험을 통해, **관측 행위 자체가 시스템에 영향을 준다**는 점을 실감했습니다.
- C언어 구조체를 활용해 하드웨어 레지스터를 계층적으로 제어하는 구조의 필요성을 이번 프로젝트에서 직접 이해할 수 있었습니다.
- 팀원들과 모르는 부분을 서로 질문하고 답하는 과정에서 놓치고 있던 개념들을 보완할 수 있었습니다.
