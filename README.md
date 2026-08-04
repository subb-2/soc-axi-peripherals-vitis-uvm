# SPI & I2C Master IP (AXI4-Lite) 설계 및 UVM 검증

📅 프로젝트 정보

* 진행 기간: 2026.04.21 ~ 2026.05.07
* 설계 및 검증 대상: `SPI Master IP`, `I2C Master IP` (AXI4-Lite 인터페이스 기반)
* 기술 스택: `SystemVerilog`, `Vivado XSim`, `UVM`, `C (MicroBlaze / Vitis)`, `Xilinx Vivado Block Design`

---

## 📝 프로젝트 개요

AXI4-Lite 인터페이스를 통해 CPU(MicroBlaze)가 레지스터를 제어하는 방식으로 **SPI Master IP** 및 **I2C Master IP**를 설계하고, UVM 환경에서 검증한 프로젝트입니다.  
단순 IP 동작 구현을 넘어, **Vivado Block Design을 활용한 MicroBlaze SoC 구성**, **C 코드로 레지스터를 직접 제어하는 소프트웨어 스택 설계**, 그리고 **2개의 Basys3 보드 간 실제 Write / Read 통신 구현**까지 하드웨어-소프트웨어 통합 관점에서 프로젝트를 진행하였습니다.  
SPI IP에 대해서는 UVM 검증 환경을 구축하여 Coverage 100%를 달성하였습니다.

### 🎥 Demo Video - SPI
<video src="https://github.com/user-attachments/assets/ab84f7cf-b22c-4739-bcce-1653732f37ad"
       width="10"
       controls>
</video>

### 🎥 Demo Video - I2C
<video src="https://github.com/user-attachments/assets/b7b43d35-ceff-4916-a216-cf21b8fe9277"
       width="400"
       controls>
</video>

---

## 🔑 주요 구현 내용

### 1. AXI4-Lite 기반 SPI / I2C Master IP 설계

* **Interface**: AXI4-Lite는 AXI4의 경량 버전으로, 레지스터 제어에 특화된 Point-to-Point 인터페이스. 5개의 독립 채널(AW · W · B · AR · R)을 통해 Write / Read를 동시에 처리하며, Burst 없이 한 번에 1개의 데이터를 전송하는 단순한 구조.
* **SPI**: Master가 생성한 클럭(SCLK)을 기준으로 MOSI / MISO를 통해 Full Duplex 통신을 수행. CPOL / CPHA 설정 및 클럭 분주(clk_div)를 레지스터로 제어.
* **I2C**: SCL / SDA 두 선을 이용한 Half Duplex 주소 기반 통신. cmd_start / cmd_write / cmd_read / cmd_stop 명령어를 레지스터에 순차적으로 기록하여 통신을 제어.

### 2. 레지스터 맵

**SPI 레지스터 맵**

| 주소  | 필드 | 설명 |
|-------|------|------|
| 0x00 | `start[10]` `cpha[9]` `cpol[8]` `clk_div[7:0]` | 제어 레지스터 |
| 0x04 | `tx_data[7:0]` | 송신 데이터 |
| 0x08 | `done[9]` `busy[8]` `rx_data[7:0]` | 상태 / 수신 데이터 |
| 0x0c | — | 예약 |

**I2C 레지스터 맵**

| 주소  | 필드 | 설명 |
|-------|------|------|
| 0x00 | `cmd_stop[3]` `cmd_read[2]` `cmd_write[1]` `cmd_start[0]` | 명령 레지스터 |
| 0x04 | `ack_in[8]` `tx_data[7:0]` | 송신 데이터 / ACK 입력 |
| 0x08 | `done[10]` `busy[9]` `ack_out[8]` `rx_data[7:0]` | 상태 / 수신 데이터 |
| 0x0c | — | 예약 |

### 3. 소프트웨어 스택 (HAL / Driver / AP 계층 분리)

MicroBlaze C 코드를 3계층으로 분리하여 하드웨어 의존성을 낮추고 유지보수성을 높였습니다.

| 계층 | SPI | I2C |
|------|-----|-----|
| **AP** | 버튼 → SPI 전송 / SW → LED 반영 | 버튼 → I2C 전송 / SW → LED 반영 |
| **Driver** | `SPI_Transfer` / `Button_GetState` / `LED_WriteData` / `SW_ReadData` | `I2C_Write_Execute` / `I2C_Read_Execute` / `Button_GetState` / `LED_WriteData` / `SW_ReadData` |
| **HAL** | Register 접근 / PIN Control | Register 접근 / Command 신호 / `Wait_Done` |
| **HW** | AXI4-Lite Slave IP | AXI4-Lite Slave IP |

### 4. SPI UVM 검증 환경

* **Architecture**: test → env(agent + scoreboard + coverage) 계층 구조. Monitor가 AXI W채널(`awaddr == 0x04`)을 감시하여 `tx_data`를 추적하고, `busy = 0 → 1` 전환 시점에 `rx_data`를 캡처하여 Scoreboard에 전달.
* **검증 시나리오 1**: Write 채널로 TX 레지스터에 기록한 값 == SPI 통신을 통해 Slave에 수신된 값.
* **검증 시나리오 2**: Slave가 MISO로 송신한 값 == SPI 통신을 통해 Read 채널로 수신된 값.
* **Coverage**: `cp_tx_data_m` / `cp_tx_data_s` 각 21개 빈 정의. 주요 빈: `0x55`(alt_01), `0xAA`(alt_10), `0x01`(lsb_only), `0x80`(msb_only), `0x00`(zero), range0~rangef.

<div align="center">

<img width="500" alt="Image" src="https://github.com/user-attachments/assets/76556fce-6c91-4aab-b10e-b792f2c1f3bc" />

</div>

---

## 🏗️ 시스템 구조

### AXI Block Diagram

<div align="center">

<img width="700" alt="Image" src="https://github.com/user-attachments/assets/0f3b4f9c-9e88-468c-bdcc-2bfd481d392e" />

</div>

### SPI + AXI Block Diagram

<div align="center">

<img width="900" alt="Image" src="https://github.com/user-attachments/assets/ba8ed9e6-6c2f-4972-977e-d8975ded8c9e" />

</div>

### I2C + AXI Block Diagram

<div align="center">

<img width="900" alt="Image" src="https://github.com/user-attachments/assets/a59f408f-7da0-4140-ab24-69fb295868f6" />

</div>

---

## 🚀 문제 해결 (Troubleshooting)

### 1. SPI의 Done 신호 포착 문제

- **문제**: SPI 데이터 처리 완료 시점을 `done` 신호 폴링으로 완료를 감지하려 했으나 신호를 놓치는 경우 발생.
- **원인**: SPI IP의 `done` 신호가 1클럭 펄스로만 발생하여 C 코드의 폴링 루프가 이를 놓침.
- **해결**: `busy` 신호가 전송 중에만 HIGH를 유지하는 특성을 이용하여 `while (busy == 0)` 폴링으로 변경.

### 2. I2C의 Done 신호 포착 문제

- **문제**: I2C 명령어 처리 완료 시점을 `done` 신호 폴링으로 완료를 감지하려 했으나 신호를 놓치는 경우 발생. 다음 명령어가 실행되지 않음.
- **원인**: I2C IP의 `done` 신호가 1클럭 펄스로만 발생하여 C 코드의 폴링 루프가 이를 놓침.
- **해결**: FPGA 측 AXI Wrapper에 `done_real` 래치 레지스터를 추가하여 `done` 펄스를 유지시키고, 소프트웨어에서 확인 후 COMMAND 레지스터를 `0x00`으로 클리어하여 명시적으로 처리.

```systemverilog
// AXI Wrapper 내 done_real 래치
reg done_real;
always @(posedge S_AXI_ACLK) begin
    if (S_AXI_ARESETN == 1'b0)
        done_real <= 1'b0;
    else begin
        if (done)
            done_real <= 1'b1;
        else if (slv_reg_wren && (axi_awaddr[...] == 2'h0))
            done_real <= 1'b0;  // COMMAND 레지스터 쓰기 시 클리어
    end
end
```

```c
void I2C_CMD_START(I2C_Typedef_t *I2Cx) {
    I2Cx->COMMAND = (1 << I2C_CMD_START_BIT);
    I2C_Wait_Done(I2Cx);
    I2Cx->COMMAND = 0x00;  // done 확인 후 클리어
}
```

### 3. UART printf로 인한 SPI 타이밍 오류

* **문제**: 디버깅 목적으로 삽입한 `UART printf`가 SPI 전송 타이밍을 지연시켜 오히려 통신 오류를 유발함.
* **원인**: UART 출력 지연이 `busy` 폴링 루프 사이에 끼어들어 SPI 타이밍 제약 위반 발생.
* **해결**: 디버깅 단계에서 printf를 제거하고, 통신 완료 후 LED 출력으로만 상태 확인. 관측 행위 자체가 시스템 타이밍에 영향을 준다는 점을 재확인.

---

## ✅ 검증 결과

<table>
<tr>

<td width="45%">

**SPI UVM 검증**

| 항목 | 결과 |
|------|------|
| Total Transaction | 2,560 |
| Overall Coverage | **100.0%** |
| `cp_tx_data_m` | 100.0% (21 bins) |
| `cp_tx_data_s` | 100.0% (21 bins) |
| Scoreboard PASS | **2,560** |
| Scoreboard FAIL | **0** |

</td>

<td width="10%">
</td>

<td width="45%">

**실보드 통신 검증 (Basys3 2보드)**

| 프로토콜 | 방향 | 결과 |
|----------|------|------|
| SPI | Master → Slave Write | ✅ PASS |
| SPI | Slave → Master Read | ✅ PASS |
| I2C | Master → Slave Write | ✅ PASS |
| I2C | Master ← Slave Read | ✅ PASS |

</td>

</tr>
</table>

---

## 📚 배운 점

* **디버깅의 부작용**: 디버깅을 위해 넣은 UART printf가 SPI 타이밍을 밀리게 해 오류를 유발한 경험을 통해, 관측 행위 자체가 시스템에 영향을 준다는 점을 실감. 임베디드 환경에서 비침습적(non-intrusive) 디버깅 방법의 중요성을 체감.
* **하드웨어 제어를 위한 소프트웨어 구조화**: C 언어 구조체를 활용해 하드웨어 레지스터를 계층적으로 제어하는 구조(HAL / Driver / AP)의 필요성을 이번 프로젝트를 통해 직접 이해. 레지스터 직접 접근과 추상화된 드라이버 계층의 역할 분리가 유지보수성과 가독성에 미치는 영향을 체감.

---

## 🖥️ 개발 환경

| 항목 | 내용 |
|------|------|
| HDL | SystemVerilog (IEEE 1800) |
| EDA Tool | Xilinx Vivado |
| 타겟 보드 | Basys3 (Xilinx Artix-7) × 2 |
| 소프트 CPU | MicroBlaze |
| 소프트웨어 개발 환경 | Xilinx Vitis (SDK) |
| 시뮬레이터 | Vivado Simulator (XSim) |
| 인터페이스 | AXI4-Lite |

---

## 📁 파일 구성

```text
SPI
├── axi_spi_top.sv               # AXI4-Lite + SPI Master 최상위 모듈
├── spi_master.sv                # SPI Master FSM (CPOL/CPHA 지원)
├── spi_slave.sv                 # SPI Slave (검증용)
├── axi_slave_if.sv              # AXI4-Lite Slave 인터페이스 처리
└── tb_spi_uvm.sv                # SPI UVM Testbench (test/env/agent/scoreboard/coverage)

I2C
├── axi_i2c_top.sv               # AXI4-Lite + I2C Master 최상위 모듈
├── i2c_master.sv                # I2C Master FSM (START/WRITE/READ/STOP)
├── i2c_slave.sv                 # I2C Slave (SystemVerilog, 검증용)
└── axi_slave_if.sv              # AXI4-Lite Slave 인터페이스 처리

SW (MicroBlaze / Vitis)
├── spi_hal.c / spi_hal.h        # SPI HAL (레지스터 직접 접근)
├── spi_driver.c / spi_driver.h  # SPI Driver (SPI_Transfer 등)
├── i2c_hal.c / i2c_hal.h        # I2C HAL (레지스터 접근 / Wait_Done)
├── i2c_driver.c / i2c_driver.h  # I2C Driver (I2C_Write_Execute 등)
└── main.c                       # AP (버튼 → 전송 / SW → LED)
```
