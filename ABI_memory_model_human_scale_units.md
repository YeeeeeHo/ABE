# ABI Memory Model – Human-Scale Brain with Compressed Representation

> **목표**  
> 인간 뇌와 **동일한 스케일·구조·수식**을 가진 인공 뇌를 가정하고,  
> 여기에 **모든 압축 기법(희소, 이벤트 기반, 템플릿, 절차적 생성, 발달 규칙, low-bit)**을 적용했을 때  
> **실제 필요한 메모리를 체계적으로 계산**한다.
>
> 이 문서는 **“뉴런/시냅스 개수는 줄이지 않고”**,  
> **“표현 방식만 바꿔서”** 메모리를 줄인다는 전제에 기반한다.

---
## 0. 전역 단위 & 표기 규칙 (Units & Notation)

실제 구현에서 혼동을 막기 위해, **모든 변수의 단위를 명시적으로 고정**한다.

### 0.1 기본 단위

- 시간:  
  - 내부 기본 단위: `ms` (millisecond)  
  - SI 환산: `1 ms = 1e-3 s`
- 전압:  
  - 내부 기본 단위: `mV` (millivolt)  
  - SI 환산: `1 mV = 1e-3 V`
- 전류:  
  - 내부 기본 단위: `nA` (nanoampere)  
  - SI 환산: `1 nA = 1e-9 A`
- 전도도(콘덕턴스):  
  - 내부 기본 단위: `µS` (microsiemens)  
  - SI 환산: `1 µS = 1e-6 S`
- 용량(캐패시턴스):  
  - 내부 기본 단위: `nF` (nanofarad)  
  - SI 환산: `1 nF = 1e-9 F`
- 거리(좌표, 뉴런 위치):  
  - 내부 기본 단위: `µm` (micrometer)  
  - SI 환산: `1 µm = 1e-6 m`
- 메모리:  
  - 1 KB = 10³ B  
  - 1 MB = 10⁶ B  
  - 1 GB = 10⁹ B  
  - 1 TB = 10¹² B  
  - 1 PB = 10¹⁵ B

### 0.2 코드 상수 예시

```python
# Time / Voltage / Current / Conductance / Capacitance units
UNIT_TIME = "ms"   # internal time unit
UNIT_VOLT = "mV"   # membrane potential, reversal potentials
UNIT_CURR = "nA"   # synaptic / external currents
UNIT_COND = "uS"   # conductances (synapse, ion channels)
UNIT_CAP  = "nF"   # membrane capacitance

# Spatial units
UNIT_DIST = "um"   # neuron position in cortex space

# Memory units are handled at model-level (B, KB, MB, GB...)
```

### 0.3 전역 심볼 정의

- $$\ N_{\text{neuron}} \$$: 뉴런의 총 개수  
- $$\ N_{\text{syn}} \$$: 시냅스의 총 개수  
- $$\ S_{\text{neuron}} \$$: 뉴런 1개당 메모리 (bytes)  
- $$\ S_{\text{syn}} \$$: 시냅스 1개당 메모리 (bytes)  
- $$\ M_{\text{neuron}} \$$: 전체 뉴런 메모리 (bytes)  
- $$\ M_{\text{syn}} \$$: 전체 시냅스 메모리 (bytes)  
- $$\ M_{\text{raw}} \$$: 뉴런+시냅스 합, 압축 전 메모리 (bytes)  
- $$\ C_{\text{struct}} \$$: 구조/표현 압축 배수 (단위 없음)  
- $$\ C_{\text{total}} \$$: 구조+low-bit 포함 총 압축 배수 (단위 없음)  
- $$\ M_{\text{final}} \$$: 압축 후 최종 메모리 (bytes)

---

## 1. 뉴런 메모리 모델 (Neuron Memory Model)

### 1.1 뉴런 스케일 (Human-Scale)

- 인간 뇌 기준 **총 뉴런 수**:

  $$\ N_{\text{neuron}} = 8.6 \times 10^{10} \quad \text{(개)}\$$

### 1.2 뉴런당 상태 변수 & 단위

| 심볼 | 설명 | 단위(내부) | 타입(예시) |
|------|------|-----------|------------|
| $$\ V_m \$$ | 막전위 (membrane potential) | `mV` | float32 |
| $$\ m, h, n \$$ | Na/K 게이팅 변수 (0~1) | 무차원 (`1`) | float32 |
| $$\ C_m \$$ | 막 용량 | `nF` | float32 |
| $$\ \bar{g}_{Na} \$$ | Na 최대 전도도 | `µS` | float32 |
| $$\ \bar{g}_{K} \$$ | K 최대 전도도 | `µS` | float32 |
| $$\ \bar{g}_{L} \$$ | 누설 전도도 | `µS` | float32 |
| $$\ E_{Na}, E_K, E_L \$$ | 역전위 (Na, K, Leak) | `mV` | float32 |
| $$\ I_{\text{ext}} \$$ | 외부 입력 전류 | `nA` | float32 |
| $$\ I_{\text{syn}} \$$ | 시냅스 전류 합 | `nA` | float32 |
| `spike_time_last` | 마지막 스파이크 발생 시점 | `ms` | float32 |
| `spike_count` | 누적 스파이크 횟수 | 무차원 (`count`) | int32 |

### 1.3 뉴런 방정식 (단위 포함)

#### 1.3.1 막전위 방정식

- 단위 체크:
  - $$\ C_m \$$: `nF`
  - $$\ V_m \$$: `mV`
  - $$\ I_* \$$: `nA`
  - 시간 $$\ t \$$: `ms`

**연속형 표현:**

$$\ C_m \frac{dV_m}{dt} = -\bar{g}_{Na} m^3 h (V_m - E_{Na}) -  \bar{g}_{K} n^4 (V_m - E_{K}) - \bar{g}_{L} (V_m - E_{L}) + I_{\text{syn}} + I_{\text{ext}} \$$

- 좌변: `nF * mV/ms = nA` (전류 단위와 일치)
- 우변 각 항: `µS * mV = nA` (단위 맞음)

#### 1.3.2 게이팅 변수 방정식

- $$\ x \in \{m, h, n\} \$$$, 무차원
- $$\ \alpha_x(V_m), \beta_x(V_m) \$$: 단위 `1/ms`

$$\
\frac{dx}{dt} = \alpha_x(V_m) (1 - x) - \beta_x(V_m) x
\$$

- 우변 전체 단위: `1/ms` → x는 무차원이므로 OK.

### 1.4 뉴런당 메모리 추정

실수형(float32) 4 bytes 기준:

- 상태 변수(\(V_m, m, h, n, I_{\text{syn}}, I_{\text{ext}}\)): 약 6 × 4B = 24B  
- 파라미터(\(C_m, \bar{g}_{Na}, \bar{g}_{K}, \bar{g}_{L}, E_*\) 등 10개): 10 × 4B = 40B  
- 스파이크 관련(`spike_time_last`, `spike_count` 등): ~8~16B  
- 여유/패딩/메타: ~16B  

합산해서 **뉴런 1개당 약 100 bytes**로 모델링한다.

$$\
S_{\text{neuron}} \approx 100 \text{ B / neuron}
\$$

### 1.5 전체 뉴런 메모리 (압축 전)

$$\
M_{\text{neuron}} = N_{\text{neuron}} \times S_{\text{neuron}}
= 8.6 \times 10^{10} \times 100 \text{ B}
= 8.6 \times 10^{12} \text{ B} \approx 8.6 \text{ TB}
\$$

---

## 2. 시냅스 메모리 모델 (Synapse Memory Model)

### 2.1 시냅스 스케일 (Human-Scale)

- 인간 뇌 기준 **총 시냅스 수** (범위):
  - 낮은 추정:  
    $$\ N_{\text{syn}}^{\text{low}} = 1 \times 10^{14} \$$
    
  - 높은 추정:  
    $$\ N_{\text{syn}}^{\text{high}} = 1 \times 10^{15} \$$

### 2.2 시냅스당 상태 변수 & 단위

| 심볼 | 설명 | 단위(내부) | 타입(예시) |
|------|------|-----------|------------|
| $$\ w \$$ | 가중치 (전도도 스케일) | 무차원 (`1`) | float16/int8/int4/binary |
| $$\ g(t) \$$ | 순간 전도도 | `µS` | float32/float16 |
| $$\ E_{\text{syn}} \$$ | 시냅스 역전위 | `mV` | float32 |
| $$\ I_{\text{syn}} \$$ | 개별 시냅스 전류 | `nA` | float32 |
| $$\ x_{\text{pre}}, x_{\text{post}} \$$ | STDP trace | 무차원 (`1`) | float16/int8 |
| $$\ d \$$ | 지연 (delay) | `ms` | float16/int8 |
| `type_id` | 시냅스 타입 (AMPA, NMDA, GABA...) | 무차원(정수) | uint8/uint16 |
| `pre_id` | pre-synaptic 뉴런 인덱스 | 무차원(정수) | int32/int64 |
| `post_id` | post-synaptic 뉴런 인덱스 | 무차원(정수) | int32/int64 |

### 2.3 시냅스 전류 식 (단위 포함)

$$\
I_{\text{syn}} = g(t) (V_m - E_{\text{syn}})
\$$

- $$\ g(t) \$$: `µS`
- $$\ V_m, E_{\text{syn}} \$$: `mV`
- 결과: `µS * mV = nA` → 전류 단위와 일치.

전도도는:

$$\ g(t) = w \cdot s(t) \$$

- \( w \): 무차원 스케일 (low-bit로 표현 가능)  
- \( s(t) \): spike 이후 exponential decay (무차원 또는 `µS` 기준 스케일링)

### 2.4 STDP 가소성 (단위 포함)

- 스파이크 시간:
  - \( t_{\text{pre}}, t_{\text{post}} \): `ms`
  - \( \Delta t = t_{\text{post}} - t_{\text{pre}} \): `ms`

전형적인 STDP 규칙:

\[
\Delta w=
\begin{cases}
A_+ \exp(-\Delta t / \tau_+) & \Delta t > 0\\
-A_- \exp(\Delta t / \tau_-) & \Delta t < 0
\end{cases}
\]

- \( A_+, A_- \): 무차원 (learning rate)  
- \( \tau_+, \tau_- \): `ms`

결과적으로 \( \Delta w \)는 무차원 → weight의 단위와 일치.

### 2.5 시냅스당 메모리 (데이터 타입별 모델링)

| 타입 | weight 표현 | 시냅스당 바이트 수 가정 \( S_{\text{syn}} \) | 비고 |
|------|-------------|-------------------------|------|
| float32 | 32bit | **24 B / synapse** | baseline |
| float16 | 16bit | **20 B / synapse** | weight만 절반 |
| int8    | 8bit  | **16 B / synapse** | weight/trace 축소 |
| int4    | 4bit  | **8 B / synapse**  | aggressive quant |
| binary  | 1bit  | **4 B / synapse**  | extreme Hebbian |

이 수치는:

- weight + trace + delay + type id + (필요 시 인덱스/압축 인코딩)을 모두 포함하는 **모델링 값**이다.

---

## 3. 압축 전(raw) 메모리 계산

### 3.1 시냅스 메모리 (압축 전)

계산식:

\[
M_{\text{syn}} = N_{\text{syn}} \times S_{\text{syn}}
\]

#### 3.1.1 낮은 추정: \(N_{\text{syn}}^{\text{low}} = 10^{14}\)

| 타입 | 시냅스당 B | 총 B | TB 단위 | PB 단위 |
|------|------------|------|---------|---------|
| float32 | 24 | \(2.4 \times 10^{15}\) | 2400 TB | 2.4 PB |
| float16 | 20 | \(2.0 \times 10^{15}\) | 2000 TB | 2.0 PB |
| int8    | 16 | \(1.6 \times 10^{15}\) | 1600 TB | 1.6 PB |
| int4    | 8  | \(0.8 \times 10^{15}\) | 800 TB  | 0.8 PB |
| binary  | 4  | \(0.4 \times 10^{15}\) | 400 TB  | 0.4 PB |

#### 3.1.2 높은 추정: \(N_{\text{syn}}^{\text{high}} = 10^{15}\)

| 타입 | 시냅스당 B | 총 B | TB 단위 | PB 단위 |
|------|------------|------|---------|---------|
| float32 | 24 | \(2.4 \times 10^{16}\) | 24000 TB | 24 PB |
| float16 | 20 | \(2.0 \times 10^{16}\) | 20000 TB | 20 PB |
| int8    | 16 | \(1.6 \times 10^{16}\) | 16000 TB | 16 PB |
| int4    | 8  | \(0.8 \times 10^{16}\) | 8000 TB  | 8 PB  |
| binary  | 4  | \(0.4 \times 10^{16}\) | 4000 TB  | 4 PB  |

### 3.2 뉴런 + 시냅스 합 (압축 전)

뉴런 메모리 \(M_{\text{neuron}} \approx 8.6 \text{ TB}\)를 더해서,

예: float16, 높은 추정 (10¹⁵ 시냅스):

- 뉴런: 8.6 TB  
- 시냅스: 20 PB = 20000 TB  
- 합계:

  \[
  M_{\text{raw}} \approx 20008.6 \text{ TB} \approx 20 \text{ PB}
  \]

---

## 4. 구조/표현 압축 (뉴런/시냅스 개수 유지)

여기서는 **뉴런/시냅스 개수를 줄이지 않고**,  
**표현 방식만 뇌와 비슷하게 바꿔서** 메모리를 줄이는 압축 기법들을 정의한다.

### 4.1 구조적/발달적 압축 배수 \(C_{\text{struct}}\)

보수적인(현실적인) 압축 배수 가정:

| 기법 | 압축 배수 | 설명 |
|------|----------|------|
| Sparse (희소 표현) | ×10 | 비활성/불필요 연결 제거 |
| Event-driven | ×10 | 스파이크 있을 때만 활성/저장 |
| Column/Kernel sharing | ×10 | minicolumn 템플릿 공유 |
| Procedural synapse generation | ×10 | 테이블 대신 생성 규칙 |
| Developmental rules | ×100 | DNA 스타일 connectome 생성 |

전체 구조 압축 배수:

\[
C_{\text{struct}} = 10 \times 10 \times 10 \times 10 \times 100
= 1 \times 10^{6}
\]

→ **구조/발달/표현만으로 약 1,000,000배(10⁶배) 압축**.

### 4.2 데이터 타입별 추가 압축 (Low-Bit)

구조 압축은 float32 기준으로 잡았다고 보고,  
데이터 타입 변경에 따른 비율은:

\[
C_{\text{type}}(\text{type}) = \frac{24}{S_{\text{syn}}(\text{type})}
\]

| 타입 | 시냅스당 B | \(C_{\text{type}}\) |
|------|-----------|------------------------|
| float32 | 24 | 1.0 |
| float16 | 20 | 1.2 |
| int8    | 16 | 1.5 |
| int4    | 8  | 3.0 |
| binary  | 4  | 6.0 |

최종 압축 배수:

\[
C_{\text{total}}(\text{type}) = C_{\text{struct}} \times C_{\text{type}}(\text{type})
\]

| 타입 | \(C_{\text{total}}\) |
|------|----------------------|
| float32 | \(1.0 \times 10^{6}\) |
| float16 | \(1.2 \times 10^{6}\) |
| int8    | \(1.5 \times 10^{6}\) |
| int4    | \(3.0 \times 10^{6}\) |
| binary  | \(6.0 \times 10^{6}\) |

---

## 5. 압축 후 최종 메모리 계산

최종 메모리:

\[
M_{\text{final}}(\text{type}) =
  \frac{M_{\text{raw}}(\text{type})}{C_{\text{total}}(\text{type})}
\]

### 5.1 낮은 추정 (\(N_{\text{syn}}^{\text{low}} = 10^{14}\))

#### 5.1.1 raw 합계 (뉴런 + 시냅스)

| 타입 | 시냅스 raw (TB) | 뉴런 (TB) | 합계 raw (TB) | 합계 raw (B) |
|------|-----------------|-----------|----------------|--------------|
| float32 | 2400 | 8.6 | 2408.6 | \(2.4086 \times 10^{15}\) |
| float16 | 2000 | 8.6 | 2008.6 | \(2.0086 \times 10^{15}\) |
| int8    | 1600 | 8.6 | 1608.6 | \(1.6086 \times 10^{15}\) |
| int4    | 800  | 8.6 | 808.6  | \(8.086 \times 10^{14}\) |
| binary  | 400  | 8.6 | 408.6  | \(4.086 \times 10^{14}\) |

#### 5.1.2 압축 후 (GB 단위)

계산 예시 (float16):

\[
M_{\text{final}}(\text{float16}) =
\frac{2.0086 \times 10^{15}}{1.2 \times 10^{6}}
= 1.6738 \times 10^{9} \text{ B}
\approx 1.67 \text{ GB}
\]

전체 표:

| 타입 | \(C_{\text{total}}\) | raw (B) | 최종 메모리 (GB) |
|------|-------------------------|---------|-------------------|
| float32 | \(1.0 \times 10^{6}\) | \(2.4086 \times 10^{15}\) | \(\approx 2.41\) GB |
| float16 | \(1.2 \times 10^{6}\) | \(2.0086 \times 10^{15}\) | \(\approx 1.67\) GB |
| int8    | \(1.5 \times 10^{6}\) | \(1.6086 \times 10^{15}\) | \(\approx 1.07\) GB |
| int4    | \(3.0 \times 10^{6}\) | \(8.086 \times 10^{14}\)  | \(\approx 0.27\) GB |
| binary  | \(6.0 \times 10^{6}\) | \(4.086 \times 10^{14}\)  | \(\approx 0.07\) GB |

> **요약 (10¹⁴ 시냅스)**  
> - float16: 약 **1.7 GB**  
> - **int8: 약 1.1 GB**  
> - int4: ~0.27 GB (270 MB)  
> - binary: ~0.07 GB (70 MB)

---

### 5.2 높은 추정 (\(N_{\text{syn}}^{\text{high}} = 10^{15}\))

#### 5.2.1 raw 합계 (뉴런 + 시냅스)

| 타입 | 시냅스 raw (TB) | 뉴런 (TB) | 합계 raw (TB) | 합계 raw (B) |
|------|-----------------|-----------|----------------|--------------|
| float32 | 24000 | 8.6 | 24008.6 | \(2.40086 \times 10^{16}\) |
| float16 | 20000 | 8.6 | 20008.6 | \(2.00086 \times 10^{16}\) |
| int8    | 16000 | 8.6 | 16008.6 | \(1.60086 \times 10^{16}\) |
| int4    | 8000  | 8.6 | 8008.6  | \(8.0086 \times 10^{15}\) |
| binary  | 4000  | 8.6 | 4008.6  | \(4.0086 \times 10^{15}\) |

#### 5.2.2 압축 후 (GB 단위)

| 타입 | \(C_{\text{total}}\) | raw (B) | 최종 메모리 (GB) |
|------|-------------------------|---------|-------------------|
| float32 | \(1.0 \times 10^{6}\) | \(2.40086 \times 10^{16}\) | \(\approx 24.01\) GB |
| float16 | \(1.2 \times 10^{6}\) | \(2.00086 \times 10^{16}\) | \(\approx 16.67\) GB |
| int8    | \(1.5 \times 10^{6}\) | \(1.60086 \times 10^{16}\) | \(\approx 10.67\) GB |
| int4    | \(3.0 \times 10^{6}\) | \(8.0086 \times 10^{15}\)  | \(\approx 2.67\) GB |
| binary  | \(6.0 \times 10^{6}\) | \(4.0086 \times 10^{15}\)  | \(\approx 0.67\) GB |

> **요약 (10¹⁵ 시냅스)**  
> - float16: 약 **16.7 GB**  
> - **int8: 약 10.7 GB**  
> - int4: ~2.7 GB  
> - binary: ~0.67 GB

---

## 6. 핵심 비교 – float16 ↔ int8

당신이 가장 궁금해하는 부분만 뽑아서 정리하면:

### 6.1 10¹⁴ 시냅스 (낮은 추정)

- float16: ≈ **1.67 GB**
- int8:    ≈ **1.07 GB**

→ int8로 바꾸면 **약 36% 메모리 절감**.

### 6.2 10¹⁵ 시냅스 (높은 추정)

- float16: ≈ **16.67 GB**
- int8:    ≈ **10.67 GB**

→ int8로 바꾸면 여기서도 **약 36% 메모리 절감**.

### 6.3 한 줄 요약

> 인간 뇌 스케일(860억 뉴런, 100~1000조 시냅스)을 유지한 상태에서,  
> 희소 표현, 이벤트 기반 시뮬레이션, 템플릿/커널 공유, 절차적 생성, 발달 규칙 기반 connectome 등  
> 구조/표현 압축을 모두 적용하면,  
> **float16 기준 약 1.7~16.7GB**,  
> **int8 기준 약 1.1~10.7GB** 정도의 메모리로  
> 전체 인공 뇌를 표현할 수 있다.

이 파일은 **ABI 메모리 모델의 기준 레퍼런스**로 사용 가능하며,  
실제 코드 구현 시 각 변수의 단위/타입/용량을 이 문서와 맞춰가며 통일하면 된다.
