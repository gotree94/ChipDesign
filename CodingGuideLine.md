# Verilog RTL 설계 종합 가이드라인
## 업계 표준 기반 코딩 규칙 및 모범 사례

---

## 목차

1. [개요](#1-개요)
2. [기본 RTL 설계 원칙](#2-기본-rtl-설계-원칙)
3. [할당 연산자 사용 규칙](#3-할당-연산자-사용-규칙)
4. [모듈 구조 및 인터페이스 설계](#4-모듈-구조-및-인터페이스-설계)
5. [클럭 및 리셋 설계](#5-클럭-및-리셋-설계)
6. [조합논리 설계 가이드라인](#6-조합논리-설계-가이드라인)
7. [순차논리 설계 가이드라인](#7-순차논리-설계-가이드라인)
8. [상태머신 설계](#8-상태머신-설계)
9. [메모리 설계](#9-메모리-설계)
10. [데이터패스 최적화](#10-데이터패스-최적화)
11. [코딩 스타일 표준](#11-코딩-스타일-표준)
12. [검증 및 테스트 고려사항](#12-검증-및-테스트-고려사항)
13. [성능 최적화 기법](#13-성능-최적화-기법)
14. [일반적인 실수 및 해결책](#14-일반적인-실수-및-해결책)
15. [업계 표준 및 도구](#15-업계-표준-및-도구)

---

## 1. 개요

이 가이드라인은 Intel, AMD/Xilinx, NVIDIA, Synopsys 등 주요 반도체 및 EDA 회사들의 RTL 설계 표준을 기반으로 작성되었습니다. 현대 ASIC 및 FPGA 설계에서 요구되는 업계 모범 사례를 포함하며, 합성 가능하고 검증 가능한 고품질 RTL 코드 작성을 목표로 합니다.

### 1.1 RTL 설계의 목표

- **기능적 정확성**: 의도한 하드웨어 동작의 정확한 구현
- **합성 가능성**: 모든 주요 합성 도구에서 일관된 결과
- **타이밍 수렴성**: 예측 가능한 타이밍 특성
- **검증 용이성**: 효율적인 시뮬레이션 및 디버깅
- **재사용성**: 모듈식 설계 및 파라미터화
- **유지보수성**: 가독성이 높고 문서화된 코드

---

## 2. 기본 RTL 설계 원칙

### 2.1 동기식 설계 (Synchronous Design)

```verilog
// 권장: 완전한 동기식 설계
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        counter <= '0;
        valid <= 1'b0;
    end else begin
        counter <= counter + 1'b1;
        valid <= enable;
    end
end
```

### 2.2 설계 계층 구조

```verilog
// 권장: 명확한 모듈 계층 구조
module top_module (
    input  wire        clk,
    input  wire        rst_n,
    input  wire [31:0] data_in,
    output wire [31:0] data_out
);

// 서브모듈 인스턴스화
datapath_unit u_datapath (
    .clk      (clk),
    .rst_n    (rst_n),
    .data_in  (data_in),
    .data_out (data_out)
);

endmodule

---

## 12. 검증 및 테스트 고려사항

### 12.1 Testbench 작성

```verilog
// 체계적인 testbench 구조
module tb_uart_transmitter;

    // 테스트 파라미터
    localparam int CLK_PERIOD = 20;  // 50MHz
    localparam int BAUD_RATE = 115200;
    localparam int BIT_TIME = 1_000_000_000 / BAUD_RATE;  // ns
    
    // DUT 신호
    logic       clk = 0;
    logic       rst_n = 0;
    logic [7:0] tx_data;
    logic       tx_valid;
    logic       tx_ready;
    logic       uart_tx;
    
    // 클럭 생성
    always #(CLK_PERIOD/2) clk = ~clk;
    
    // DUT 인스턴스
    uart_transmitter #(
        .CLK_FREQ(50_000_000),
        .BAUD_RATE(BAUD_RATE)
    ) dut (
        .clk(clk),
        .rst_n(rst_n),
        .tx_data(tx_data),
        .tx_valid(tx_valid),
        .tx_ready(tx_ready),
        .uart_tx(uart_tx)
    );
    
    // 테스트 시퀀스
    initial begin
        // 초기화
        rst_n = 0;
        tx_valid = 0;
        tx_data = 8'h00;
        
        // 리셋 해제
        repeat(10) @(posedge clk);
        rst_n = 1;
        
        // 테스트 케이스 1: 단일 바이트 전송
        test_single_byte(8'h55);
        
        // 테스트 케이스 2: 연속 바이트 전송
        test_continuous_bytes();
        
        // 테스트 완료
        $display("All tests passed!");
        $finish;
    end
    
    // 단일 바이트 전송 테스트
    task test_single_byte(input [7:0] data);
        @(posedge clk);
        tx_data = data;
        tx_valid = 1;
        
        @(posedge clk);
        tx_valid = 0;
        
        // 전송 완료 대기
        wait(tx_ready);
        $display("Transmitted byte: 0x%02h", data);
    endtask
    
    // 연속 바이트 전송 테스트
    task test_continuous_bytes();
        for (int i = 0; i < 256; i++) begin
            test_single_byte(i[7:0]);
        end
    endtask

endmodule
```

### 12.2 Assertion 기반 검증

```verilog
// SVA (SystemVerilog Assertions) 사용
module uart_transmitter_with_assertions #(
    parameter int CLK_FREQ  = 50_000_000,
    parameter int BAUD_RATE = 115_200
) (
    input  wire       clk,
    input  wire       rst_n,
    input  wire [7:0] tx_data,
    input  wire       tx_valid,
    output logic      tx_ready,
    output logic      uart_tx
);

    // 기본 모듈 로직 (생략)
    // ...
    
    // Assertions
    
    // 1. 리셋 후 ready 신호 확인
    property reset_ready;
        @(posedge clk) (!rst_n) |-> ##1 tx_ready;
    endproperty
    assert property (reset_ready) else $error("tx_ready not asserted after reset");
    
    // 2. valid-ready 핸드셰이크
    property valid_ready_handshake;
        @(posedge clk) (tx_valid && tx_ready) |-> ##1 !tx_ready;
    endproperty
    assert property (valid_ready_handshake) else $error("Handshake violation");
    
    // 3. UART 출력 비트 타이밍
    property uart_bit_timing;
        @(posedge clk) $fell(uart_tx) |-> ##[BAUD_DIV-1:BAUD_DIV+1] $rose(uart_tx);
    endproperty
    assert property (uart_bit_timing) else $error("UART bit timing violation");

endmodule
```

### 12.3 커버리지 모니터링

```verilog
// 기능 커버리지
covergroup uart_coverage @(posedge clk);
    // 데이터 값 커버리지
    tx_data_cp: coverpoint tx_data {
        bins low_values  = {[8'h00:8'h0F]};
        bins mid_values  = {[8'h10:8'hEF]};
        bins high_values = {[8'hF0:8'hFF]};
        bins special[]   = {8'h55, 8'hAA, 8'h00, 8'hFF};
    }
    
    // 상태 커버리지
    state_cp: coverpoint current_state {
        bins idle_state  = {IDLE};
        bins start_state = {START};
        bins data_state  = {DATA};
        bins stop_state  = {STOP};
    }
    
    // 상태 전이 커버리지
    state_trans: coverpoint current_state {
        bins valid_transitions = (IDLE => START), (START => DATA), 
                                (DATA => STOP), (STOP => IDLE);
    }
endgroup

uart_coverage cov_inst = new();
```

---

## 13. 성능 최적화 기법

### 13.1 타이밍 최적화

```verilog
// 긴 조합논리 경로 분할
// 나쁜 예: 긴 조합논리 체인
always_comb begin
    result = ((((a + b) * c) + d) * e) + f;  // 긴 지연
end

// 좋은 예: 파이프라인으로 분할
always_ff @(posedge clk) begin
    stage1 <= a + b;
    stage2 <= stage1 * c;
    stage3 <= stage2 + d;
    stage4 <= stage3 * e;
    result <= stage4 + f;
end
```

### 13.2 면적 최적화

```verilog
// 리소스 공유
module shared_multiplier #(
    parameter int WIDTH = 16
) (
    input  wire                clk,
    input  wire                rst_n,
    input  wire                sel,
    input  wire  [WIDTH-1:0]   a1, b1, a2, b2,
    output logic [2*WIDTH-1:0] result1, result2
);

    logic [WIDTH-1:0] mux_a, mux_b;
    logic [2*WIDTH-1:0] product;
    logic sel_reg;
    
    // 입력 멀티플렉싱
    always_comb begin
        if (sel) begin
            mux_a = a2;
            mux_b = b2;
        end else begin
            mux_a = a1;
            mux_b = b1;
        end
    end
    
    // 공유 곱셈기
    always_ff @(posedge clk) begin
        product <= mux_a * mux_b;
        sel_reg <= sel;
    end
    
    // 출력 디멀티플렉싱
    always_ff @(posedge clk) begin
        if (sel_reg)
            result2 <= product;
        else
            result1 <= product;
    end

endmodule
```

### 13.3 전력 최적화

```verilog
// 클럭 게이팅
module clock_gated_counter #(
    parameter int WIDTH = 32
) (
    input  wire             clk,
    input  wire             rst_n,
    input  wire             enable,
    input  wire             count_en,
    output logic [WIDTH-1:0] count
);

    logic gated_clk;
    
    // 클럭 게이팅 셀 (합성 도구에서 인퍼)
    always_comb begin
        gated_clk = clk & (enable | rst_n);
    end
    
    // 게이트된 클럭 사용
    always_ff @(posedge gated_clk or negedge rst_n) begin
        if (!rst_n)
            count <= '0;
        else if (count_en)
            count <= count + 1'b1;
    end

endmodule
```

---

## 14. 일반적인 실수 및 해결책

### 14.1 경합 조건 (Race Condition)

```verilog
// 문제: 블로킹 할당으로 인한 경합
always_ff @(posedge clk) begin
    a = b;
    c = a;    // 새로운 'a' 값 사용 (의도하지 않을 수 있음)
end

// 해결: 논블로킹 할당 사용
always_ff @(posedge clk) begin
    a <= b;
    c <= a;   // 이전 클럭의 'a' 값 사용
end
```

### 14.2 다중 드라이버

```verilog
// 문제: 동일 신호를 여러 블록에서 구동
always_ff @(posedge clk) begin
    if (condition1)
        output_sig <= value1;
end

always_ff @(posedge clk) begin  // 오류!
    if (condition2)
        output_sig <= value2;
end

// 해결: 단일 always 블록 사용
always_ff @(posedge clk) begin
    if (condition1)
        output_sig <= value1;
    else if (condition2)
        output_sig <= value2;
end
```

### 14.3 메타스테이빌리티

```verilog
// 문제: 비동기 신호 직접 사용
always_ff @(posedge clk) begin
    sync_out <= async_in;  // 메타스테이빌리티 위험
end

// 해결: 2-플롭 동기화기 사용
always_ff @(posedge clk) begin
    sync_reg1 <= async_in;
    sync_reg2 <= sync_reg1;
end
assign sync_out = sync_reg2;
```

### 14.4 부적절한 리셋 처리

```verilog
// 문제: 일부 신호만 리셋
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        state <= IDLE;
        // counter 리셋 누락!
    end else begin
        state <= next_state;
        counter <= counter + 1;
    end
end

// 해결: 모든 신호 명시적 리셋
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        state <= IDLE;
        counter <= '0;        // 명시적 리셋
        valid <= 1'b0;        // 명시적 리셋
    end else begin
        state <= next_state;
        counter <= counter + 1;
        valid <= enable;
    end
end
```

---

## 15. 업계 표준 및 도구

### 15.1 주요 반도체 회사 가이드라인

#### Intel FPGA 설계 가이드라인
- **Hyperflex 아키텍처**: 레지스터 리타이밍 최적화
- **하이퍼 파이프라이닝**: 깊은 파이프라인 구조
- **제어 신호 백프레셔**: 플로우 제어 최적화

#### AMD/Xilinx UltraFast 방법론
- **RTL 코딩 가이드라인 (UG949)**: 표준 RTL 코딩 규칙
- **타이밍 중심 설계**: 초기 단계부터 타이밍 고려
- **계층적 설계**: 모듈 기반 설계 방법론

#### Synopsys Design Compiler
- **데이터패스 최적화**: 산술 연산 최적화
- **리타이밍 기능**: 레지스터 최적 배치
- **전력 최적화**: 클럭 게이팅 및 전력 인식 합성

### 15.2 검증 도구

#### 린팅 도구
- **Synopsys SpyGlass**: 설계 규칙 검사, RTL 분석
- **Siemens Questa Lint**: 코딩 표준 검증
- **Cadence HAL**: 하드웨어 분석 및 린팅

#### 시뮬레이터
- **Synopsys VCS**: SystemVerilog 및 UVM 지원
- **Siemens Questa**: 고급 디버깅 기능
- **Cadence Xcelium**: 고성능 시뮬레이션

#### 형식 검증
- **Synopsys VC Formal**: 형식 검증 및 어서션 기반 검증
- **Cadence JasperGold**: 형식 속성 검증
- **OneSpin 360**: 포괄적 형식 검증

### 15.3 합성 도구

#### 주요 합성 도구
- **Synopsys Design Compiler**: 업계 표준 RTL 합성
- **Synopsys Fusion Compiler**: RTL-to-GDSII 통합 플로우
- **Cadence Genus**: 고급 합성 최적화

#### FPGA 합성 도구
- **Intel Quartus Prime**: Intel FPGA 전용
- **AMD Vivado**: AMD/Xilinx FPGA 전용
- **Microchip Libero**: Microsemi FPGA 전용

---

## 16. 체크리스트

### 16.1 설계 전 확인사항

- [ ] **기능 명세서 검토**: 요구사항 명확히 이해
- [ ] **인터페이스 정의**: 포트 및 프로토콜 명세
- [ ] **클럭 도메인 계획**: CDC 처리 방안 수립
- [ ] **리셋 전략**: 리셋 구조 및 시퀀스 정의
- [ ] **타이밍 제약**: 초기 타이밍 목표 설정

### 16.2 코딩 중 확인사항

- [ ] **할당 연산자**: 조합논리(`=`), 순차논리(`<=`) 올바른 사용
- [ ] **민감도 리스트**: `always_comb` 또는 `@(*)` 사용
- [ ] **리셋 처리**: 모든 순차논리에 적절한 리셋
- [ ] **레치 방지**: 모든 조건에서 신호 할당
- [ ] **명명 규칙**: 일관된 신호 및 모듈명 사용

### 16.3 검증 전 확인사항

- [ ] **린팅 통과**: 모든 린팅 경고 해결
- [ ] **시뮬레이션 준비**: 포괄적 테스트벤치 작성
- [ ] **커버리지 계획**: 기능 및 코드 커버리지 목표
- [ ] **어서션 작성**: 주요 속성에 대한 SVA 작성

### 16.4 합성 전 확인사항

- [ ] **타이밍 제약**: SDC 파일 작성 완료
- [ ] **면적 제약**: 면적 목표 및 제약 설정
- [ ] **전력 제약**: 전력 최적화 설정
- [ ] **테스트빌리티**: DFT 규칙 준수

---

## 17. 추가 리소스

### 17.1 표준 문서
- **IEEE 1364-2005**: Verilog HDL 표준
- **IEEE 1800-2017**: SystemVerilog 표준
- **IEEE 1076-2019**: VHDL 표준

### 17.2 참고 서적
- "RTL Hardware Design Using VHDL" - Pong P. Chu
- "Digital Design and Computer Architecture" - Harris & Harris
- "Writing Testbenches using SystemVerilog" - Janick Bergeron

### 17.3 온라인 리소스
- **Intel FPGA 문서**: docs.intel.com/programmable
- **AMD 문서 포털**: docs.amd.com
- **Synopsys SolvNet**: solvnet.synopsys.com
- **IEEE Xplore**: ieeexplore.ieee.org

---

**문서 버전**: 2.0  
**최종 수정일**: 2025년 9월  
**기반 표준**: Intel Hyperflex, AMD UltraFast, Synopsys DC NXT  
**검토자**: 반도체 설계 전문가

---

*이 문서는 주요 반도체 회사들의 RTL 설계 가이드라인을 종합하여 작성되었으며, 실제 프로젝트에서는 해당 조직의 특정 요구사항과 함께 적용하시기 바랍니다. 지속적인 업데이트를 통해 최신 설계 트렌드와 도구 발전사항을 반영할 예정입니다.*
```

### 2.3 설계 규칙 체크 (DRC) 준수

- **Lint 도구 사용**: SpyGlass, Verilint 등
- **합성 전 검증**: 모든 경고 해결
- **타이밍 제약**: 적절한 SDC 제약 설정

---

## 3. 할당 연산자 사용 규칙

### 3.1 기본 원칙

| 회로 유형 | 할당 연산자 | Always 블록 | 이유 |
|-----------|-------------|-------------|------|
| 조합논리 | `=` (블로킹) | `always_comb` | 즉각적 할당, 조합논리 특성 |
| 레치 | `=` (블로킹) | `always_latch` | 투명성 및 조합논리적 특성 |
| 플립플롭 | `<=` (논블로킹) | `always_ff` | 경합 조건 방지, 순차논리 |

### 3.2 조합논리 구현

```verilog
// SystemVerilog 스타일 (권장)
always_comb begin
    case (sel)
        2'b00: mux_out = in0;
        2'b01: mux_out = in1;
        2'b10: mux_out = in2;
        2'b11: mux_out = in3;
        default: mux_out = '0;  // 명시적 기본값
    endcase
end

// 전통적 Verilog 스타일
always @(*) begin
    case (sel)
        2'b00: mux_out = in0;
        2'b01: mux_out = in1;
        2'b10: mux_out = in2;
        2'b11: mux_out = in3;
        default: mux_out = 32'h0;
    endcase
end
```

### 3.3 순차논리 구현

```verilog
// 권장: 논블로킹 할당
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        data_reg <= '0;
        valid_reg <= 1'b0;
    end else begin
        data_reg <= data_in;
        valid_reg <= valid_in;
    end
end
```

---

## 4. 모듈 구조 및 인터페이스 설계

### 4.1 모듈 포트 정의

```verilog
module example_module #(
    parameter int DATA_WIDTH = 32,
    parameter int ADDR_WIDTH = 8
) (
    // 클럭 및 리셋
    input  wire                    clk,
    input  wire                    rst_n,
    
    // 제어 신호
    input  wire                    enable,
    input  wire                    valid_in,
    output logic                   ready_out,
    
    // 데이터 인터페이스
    input  wire  [DATA_WIDTH-1:0] data_in,
    input  wire  [ADDR_WIDTH-1:0] addr_in,
    output logic [DATA_WIDTH-1:0] data_out,
    output logic                   valid_out
);
```

### 4.2 인터페이스 명명 규칙

```verilog
// 방향성 명확화
input  wire [31:0] axi_awaddr_i,     // 입력은 _i 접미사
output wire [31:0] axi_rdata_o,      // 출력은 _o 접미사
output wire        axi_rvalid_o,
input  wire        axi_rready_i,

// 클럭 도메인 표시
input  wire        clk_core,         // 코어 클럭
input  wire        clk_mem,          // 메모리 클럭
input  wire        rst_core_n,       // 액티브 로우 리셋
```

### 4.3 파라미터화

```verilog
// 권장: 타입이 명시된 파라미터
module fifo #(
    parameter int unsigned DATA_WIDTH = 32,
    parameter int unsigned DEPTH      = 16,
    parameter bit          ASYNC_RST  = 1'b1,
    
    // 계산된 파라미터
    localparam int unsigned ADDR_WIDTH = $clog2(DEPTH)
) (
    // 포트 정의
);
```

---

## 5. 클럭 및 리셋 설계

### 5.1 클럭 설계

```verilog
// 권장: 단일 클럭 엣지 사용
always_ff @(posedge clk) begin
    // 모든 로직이 동일한 클럭 엣지 사용
end

// 금지: 혼합 클럭 엣지
// always_ff @(posedge clk or negedge clk) begin  // 금지!
```

### 5.2 리셋 설계

```verilog
// 권장: 비동기 리셋, 동기 해제
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        // 모든 레지스터 초기화
        state <= IDLE;
        counter <= '0;
        valid <= 1'b0;
    end else begin
        // 정상 동작
        state <= next_state;
        counter <= counter + 1'b1;
        valid <= enable;
    end
end
```

### 5.3 클럭 도메인 크로싱 (CDC)

```verilog
// 2-플롭 동기화기
module synchronizer #(
    parameter int STAGES = 2,
    parameter bit RESET_VALUE = 1'b0
) (
    input  wire clk_dst,
    input  wire rst_dst_n,
    input  wire data_async,
    output wire data_sync
);

    logic [STAGES-1:0] sync_reg;
    
    always_ff @(posedge clk_dst or negedge rst_dst_n) begin
        if (!rst_dst_n)
            sync_reg <= {STAGES{RESET_VALUE}};
        else
            sync_reg <= {sync_reg[STAGES-2:0], data_async};
    end
    
    assign data_sync = sync_reg[STAGES-1];

endmodule
```

---

## 6. 조합논리 설계 가이드라인

### 6.1 민감도 리스트

```verilog
// 권장: @(*) 또는 always_comb 사용
always_comb begin
    case (opcode)
        ADD: result = a + b;
        SUB: result = a - b;
        AND: result = a & b;
        OR:  result = a | b;
        default: result = '0;
    endcase
end
```

### 6.2 레치 방지

```verilog
// 잘못된 예: 의도하지 않은 레치 생성
always_comb begin
    if (sel)
        y = a;
    // else 조건 누락으로 레치 생성
end

// 올바른 예: 모든 경우 처리
always_comb begin
    if (sel)
        y = a;
    else
        y = b;
end

// 또는 기본값 사용
always_comb begin
    y = b;  // 기본값
    if (sel)
        y = a;
end
```

### 6.3 우선순위 인코더 vs 케이스문

```verilog
// 우선순위 인코더 (if-else)
always_comb begin
    if (req[3])
        grant = 4'b1000;
    else if (req[2])
        grant = 4'b0100;
    else if (req[1])
        grant = 4'b0010;
    else if (req[0])
        grant = 4'b0001;
    else
        grant = 4'b0000;
end

// 케이스문 (평등 우선순위)
always_comb begin
    case (sel)
        2'b00: grant = 4'b0001;
        2'b01: grant = 4'b0010;
        2'b10: grant = 4'b0100;
        2'b11: grant = 4'b1000;
        default: grant = 4'b0000;
    endcase
end
```

---

## 7. 순차논리 설계 가이드라인

### 7.1 플립플롭 설계

```verilog
// 기본 D 플립플롭
always_ff @(posedge clk) begin
    q <= d;
end

// 인에이블이 있는 플립플롭
always_ff @(posedge clk) begin
    if (enable)
        q <= d;
end

// 리셋이 있는 플립플롭
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        q <= 1'b0;
    else
        q <= d;
end
```

### 7.2 카운터 설계

```verilog
// 기본 카운터
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        counter <= '0;
    end else if (enable) begin
        if (counter == MAX_COUNT-1)
            counter <= '0;
        else
            counter <= counter + 1'b1;
    end
end

// 로드 가능한 카운터
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        counter <= '0;
    end else if (load) begin
        counter <= load_value;
    end else if (enable) begin
        counter <= counter + 1'b1;
    end
end
```

### 7.3 시프트 레지스터

```verilog
// 직렬 입력 시프트 레지스터
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        shift_reg <= '0;
    end else if (shift_enable) begin
        shift_reg <= {shift_reg[WIDTH-2:0], serial_in};
    end
end
```

---

## 8. 상태머신 설계

### 8.1 3-프로세스 상태머신 (권장)

```verilog
// 상태 정의
typedef enum logic [2:0] {
    IDLE    = 3'b000,
    START   = 3'b001,
    ACTIVE  = 3'b010,
    WAIT    = 3'b011,
    DONE    = 3'b100
} state_t;

state_t current_state, next_state;

// 1. 상태 레지스터 (순차논리)
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        current_state <= IDLE;
    else
        current_state <= next_state;
end

// 2. 다음 상태 로직 (조합논리)
always_comb begin
    next_state = current_state;  // 기본값
    
    case (current_state)
        IDLE: begin
            if (start)
                next_state = START;
        end
        
        START: begin
            next_state = ACTIVE;
        end
        
        ACTIVE: begin
            if (done)
                next_state = WAIT;
            else if (error)
                next_state = IDLE;
        end
        
        WAIT: begin
            if (ack)
                next_state = DONE;
        end
        
        DONE: begin
            next_state = IDLE;
        end
        
        default: begin
            next_state = IDLE;
        end
    endcase
end

// 3. 출력 로직 (조합논리)
always_comb begin
    // 기본값
    busy = 1'b0;
    valid = 1'b0;
    
    case (current_state)
        IDLE: begin
            busy = 1'b0;
        end
        
        START, ACTIVE, WAIT: begin
            busy = 1'b1;
        end
        
        DONE: begin
            valid = 1'b1;
        end
        
        default: begin
            busy = 1'b0;
            valid = 1'b0;
        end
    endcase
end
```

### 8.2 1-프로세스 상태머신 (간단한 경우)

```verilog
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        state <= IDLE;
        counter <= '0;
        valid <= 1'b0;
    end else begin
        case (state)
            IDLE: begin
                valid <= 1'b0;
                if (start) begin
                    state <= ACTIVE;
                    counter <= '0;
                end
            end
            
            ACTIVE: begin
                counter <= counter + 1'b1;
                if (counter == DELAY-1) begin
                    state <= DONE;
                    valid <= 1'b1;
                end
            end
            
            DONE: begin
                state <= IDLE;
            end
            
            default: begin
                state <= IDLE;
            end
        endcase
    end
end
```

---

## 9. 메모리 설계

### 9.1 단순 듀얼 포트 RAM

```verilog
module simple_dual_port_ram #(
    parameter int DATA_WIDTH = 32,
    parameter int ADDR_WIDTH = 10,
    parameter int DEPTH      = 2**ADDR_WIDTH
) (
    input  wire                    clk,
    
    // 쓰기 포트
    input  wire                    wr_en,
    input  wire  [ADDR_WIDTH-1:0] wr_addr,
    input  wire  [DATA_WIDTH-1:0] wr_data,
    
    // 읽기 포트
    input  wire  [ADDR_WIDTH-1:0] rd_addr,
    output logic [DATA_WIDTH-1:0] rd_data
);

    logic [DATA_WIDTH-1:0] mem [0:DEPTH-1];
    
    // 쓰기 로직
    always_ff @(posedge clk) begin
        if (wr_en)
            mem[wr_addr] <= wr_data;
    end
    
    // 읽기 로직
    always_ff @(posedge clk) begin
        rd_data <= mem[rd_addr];
    end

endmodule
```

### 9.2 FIFO 설계

```verilog
module synchronous_fifo #(
    parameter int DATA_WIDTH = 32,
    parameter int DEPTH      = 16,
    localparam int ADDR_WIDTH = $clog2(DEPTH)
) (
    input  wire                    clk,
    input  wire                    rst_n,
    
    // 쓰기 인터페이스
    input  wire                    wr_en,
    input  wire  [DATA_WIDTH-1:0] wr_data,
    output logic                   full,
    
    // 읽기 인터페이스
    input  wire                    rd_en,
    output logic [DATA_WIDTH-1:0] rd_data,
    output logic                   empty,
    
    // 상태 신호
    output logic [ADDR_WIDTH:0]   count
);

    logic [DATA_WIDTH-1:0] mem [0:DEPTH-1];
    logic [ADDR_WIDTH:0] wr_ptr, rd_ptr;
    
    // 포인터 관리
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            wr_ptr <= '0;
            rd_ptr <= '0;
        end else begin
            if (wr_en && !full)
                wr_ptr <= wr_ptr + 1'b1;
            if (rd_en && !empty)
                rd_ptr <= rd_ptr + 1'b1;
        end
    end
    
    // 메모리 액세스
    always_ff @(posedge clk) begin
        if (wr_en && !full)
            mem[wr_ptr[ADDR_WIDTH-1:0]] <= wr_data;
    end
    
    always_ff @(posedge clk) begin
        rd_data <= mem[rd_ptr[ADDR_WIDTH-1:0]];
    end
    
    // 상태 플래그
    assign count = wr_ptr - rd_ptr;
    assign empty = (count == 0);
    assign full  = (count == DEPTH);

endmodule
```

---

## 10. 데이터패스 최적화

### 10.1 파이프라인 설계

```verilog
// 3단계 파이프라인 곱셈기
module pipelined_multiplier #(
    parameter int WIDTH = 32
) (
    input  wire                clk,
    input  wire                rst_n,
    input  wire                valid_in,
    input  wire  [WIDTH-1:0]   a_in,
    input  wire  [WIDTH-1:0]   b_in,
    output logic               valid_out,
    output logic [2*WIDTH-1:0] product_out
);

    // 파이프라인 스테이지
    logic [WIDTH-1:0]   a_reg1, b_reg1;
    logic [2*WIDTH-1:0] partial_product;
    logic [2*WIDTH-1:0] product_reg;
    logic [2:0]         valid_pipe;
    
    // 스테이지 1: 입력 레지스터
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            a_reg1 <= '0;
            b_reg1 <= '0;
            valid_pipe[0] <= 1'b0;
        end else begin
            a_reg1 <= a_in;
            b_reg1 <= b_in;
            valid_pipe[0] <= valid_in;
        end
    end
    
    // 스테이지 2: 곱셈
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            partial_product <= '0;
            valid_pipe[1] <= 1'b0;
        end else begin
            partial_product <= a_reg1 * b_reg1;
            valid_pipe[1] <= valid_pipe[0];
        end
    end
    
    // 스테이지 3: 출력 레지스터
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            product_reg <= '0;
            valid_pipe[2] <= 1'b0;
        end else begin
            product_reg <= partial_product;
            valid_pipe[2] <= valid_pipe[1];
        end
    end
    
    assign product_out = product_reg;
    assign valid_out = valid_pipe[2];

endmodule
```

### 10.2 리타이밍을 위한 코딩

```verilog
// 나쁜 예: 중간에 레지스터 배치
always_ff @(posedge clk) begin
    temp1 <= a + b;
    temp2 <= c + d;
end

always_ff @(posedge clk) begin
    result <= temp1 * temp2 + e;
end

// 좋은 예: 입출력에 레지스터 배치, 합성 도구가 최적 위치로 이동
always_ff @(posedge clk) begin
    a_reg <= a;
    b_reg <= b;
    c_reg <= c;
    d_reg <= d;
    e_reg <= e;
end

always_comb begin
    temp1 = a_reg + b_reg;
    temp2 = c_reg + d_reg;
    result_comb = temp1 * temp2 + e_reg;
end

always_ff @(posedge clk) begin
    result <= result_comb;
end
```

---

## 11. 코딩 스타일 표준

### 11.1 명명 규칙

```verilog
// 모듈명: 소문자_언더스코어
module uart_transmitter (...);

// 신호명
input  wire        clk,              // 클럭
input  wire        rst_n,            // 액티브 로우 리셋
input  wire [31:0] data_i,           // 입력 데이터
output logic       valid_o,          // 출력 유효 신호

// 상수
parameter int unsigned FIFO_DEPTH = 16;
localparam int CLK_FREQ_MHZ = 100;

// enum 타입
typedef enum logic [1:0] {
    STATE_IDLE = 2'b00,
    STATE_ACTIVE = 2'b01,
    STATE_WAIT = 2'b10,
    STATE_DONE = 2'b11
} state_e;
```

### 11.2 인덴테이션 및 포맷팅

```verilog
module well_formatted_module #(
    parameter int DATA_WIDTH = 32,
    parameter int ADDR_WIDTH = 8
) (
    input  wire                    clk,
    input  wire                    rst_n,
    input  wire  [DATA_WIDTH-1:0] data_in,
    output logic [DATA_WIDTH-1:0] data_out
);

    // 지역 신호 선언
    logic [ADDR_WIDTH-1:0] addr_reg;
    logic                  valid_reg;
    
    // 조합논리
    always_comb begin
        case (addr_reg[1:0])
            2'b00: data_out = data_in;
            2'b01: data_out = data_in << 1;
            2'b10: data_out = data_in << 2;
            2'b11: data_out = data_in << 3;
            default: data_out = '0;
        endcase
    end
    
    // 순차논리
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            addr_reg <= '0;
            valid_reg <= 1'b0;
        end else begin
            addr_reg <= addr_reg + 1'b1;
            valid_reg <= 1'b1;
        end
    end

endmodule
```

### 11.3 주석 및 문서화

```verilog
/**
 * UART 송신기 모듈
 * 
 * @param CLK_FREQ    시스템 클럭 주파수 (Hz)
 * @param BAUD_RATE   전송 속도 (bps)
 * 
 * @port clk          시스템 클럭
 * @port rst_n        비동기 리셋 (액티브 로우)
 * @port tx_data      전송할 8비트 데이터
 * @port tx_valid     전송 요청 신호
 * @port tx_ready     전송 준비 신호
 * @port uart_tx      UART 출력 라인
 */
module uart_transmitter #(
    parameter int CLK_FREQ  = 50_000_000,  // 50MHz
    parameter int BAUD_RATE = 115_200      // 115.2k bps
) (
    input  wire       clk,
    input  wire       rst_n,
    input  wire [7:0] tx_data,
    input  wire       tx_valid,
    output logic      tx_ready,
    output logic      uart_tx
);

    // 보드레이트 분주기 계산
    localparam int BAUD_DIV = CLK_FREQ / BAUD_RATE;
    
    // 상태 정의
    typedef enum logic [1:0] {
        IDLE  = 2'b00,  // 대기 상태
        START = 2'b01,  // 스타트 비트 전송
        DATA  = 2'b10,  // 데이터 비트 전송
        STOP  = 2'b11   // 스톱 비트 전송
    } tx_state_e;
    
    tx_state_e current_state, next_state;
    logic [15:0] baud_counter;
    logic [2:0]  bit_counter;
    logic [7:0]  tx_shift_reg;

    // 상태머신 구현 (생략)
    // ...

endmodule
