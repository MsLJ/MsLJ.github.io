---
title: "RHEL 7.6 재부팅 후 게이트웨이 ping 불가 트러블슈팅 - 서버는 멀쩡했고 범인은 GBIC"
date: 2026-11-11
categories: [Linux, Network, Troubleshooting]
tags: [RHEL, Network, ping, GBIC, ethtool, ARP, L2, 트러블슈팅]
---

# 재부팅 후 게이트웨이로 ping이 안 나가던 사례

## 들어가며

RHEL 7.6 서버를 점검을 위해 껐다 켰더니 **게이트웨이로 ping조차 나가지 않는** 현상을 만났습니다. 서버 내부에서 확인할 수 있는 거의 모든 걸 점검했지만 이상이 없었고, 최종적으로는 **스위치 쪽 GBIC 모듈 + UTP 케이블을 다시 꽂아서** 해결했습니다.

"서버 안에서 다 정상인데 통신이 안 된다"는 상황이 왜 발생하는지, 그리고 다음에 비슷한 일이 생겼을 때 어디서부터 어떻게 좁혀나갈지 제가 실제로 확인한 순서 그대로 정리합니다.

**환경**
- OS: RHEL 7.6
- 증상: 서버 재부팅 후 게이트웨이 ping 불가 (서버 → 게이트웨이 방향으로 통신 자체가 안 됨)
- 물리 구성: 서버 NIC --(UTP 케이블)--(GBIC 모듈)-- 스위치

---

## Part 1: 문제 상황

서버를 shutdown 후 다시 부팅했는데 외부 통신은 물론이고 **동일망 게이트웨이로 ping이 나가지 않는 상태**였습니다.

이런 때 대부분은 아래 중 하나입니다.

1. 인터페이스 자체가 안 올라옴 (L1 문제)
2. IP/게이트웨이 설정이 꼬임 (L3 설정 문제)
3. 링크는 올라와 있는데 L2(스위치/케이블/모듈)에서 프레임이 죽음

순서대로 좁혀나갔습니다.

---

## Part 2: 서버 쪽 점검 순서 (실제로 확인한 것들)

### 2.1 인터페이스/링크 상태 확인

```bash
ip a            # 인터페이스에 IP가 정상적으로 할당되어 있는지
ethtool eth0    # 링크 상태 확인
```

`ip a`에서는 인터페이스에 IP가 정상적으로 붙어 있었고, `ethtool`에서도 `Link detected: yes` — **모두 up 상태**였습니다.

> 💡 여기서 함정이 하나 있습니다. `Link detected: yes`는 "물리적 링크가 형성됐다"는 뜻일 뿐 **통신이 된다는 보장이 아닙니다.** 링크가 올라와 있어도 스위치 포트나 모듈 쪽에서 프레임이 유실되면 통신은 안 됩니다.

### 2.2 네트워크 관리 주체 확인 (NetworkManager vs network)

RHEL 7은 `NetworkManager`와 `network` 서비스가 공존하는데, 둘 다 설정을 건드리면 충돌로 IP가 의도와 다르게 잡힐 수 있습니다. 어떤 서비스가 네트워크를 관리 중인지 확인했습니다.

```bash
systemctl status NetworkManager   # 결과: disabled
systemctl status network          # 결과: active (enabled)
```

이 서버는 `network` 서비스로만 관리되고 있었고, `ifcfg-*`에 `NM_CONTROLLED` 설정 충돌 같은 것도 없었습니다.

> 💡 RHEL 7에서는 `nmcli device status`로 어떤 인터페이스가 NetworkManager 관리인지(unmanaged 여부)도 함께 확인할 수 있습니다.

### 2.3 설정 파일 점검

```bash
cat /etc/sysconfig/network-scripts/ifcfg-*
cat /etc/sysconfig/network-scripts/route-* 2>/dev/null
```

- IPADDR, NETMASK, GATEWAY 값이 실제 할당받아야 할 값과 일치하는지
- ifcfg 파일이 중복으로 있거나(예: ifcfg-eth0와 ifcfg-eth0.bak) 잘못된 값이 들어가 있는지

확인 결과 **중복도, 잘못된 값도 없었습니다.**

### 2.4 라우팅 테이블 확인

```bash
netstat -rn      # 또는 ip route show
```

기본 게이트웨이(0.0.0.0 → 게이트웨이 IP)가 라우팅 테이블에 정상적으로 잡혀 있었고, 소속 서브넷 라우트도 이상 없었습니다.

### 2.5 서버 쪽 정상 → 스위치 의심

여기까지 확인한 결과:

| 점검 항목 | 결과 |
|-----------|------|
| 인터페이스 상태 (`ip a`) | ✅ 정상 (IP 할당됨) |
| 링크 상태 (`ethtool`) | ✅ Link detected: yes |
| 네트워크 관리 서비스 | ✅ network 단일 관리, 충돌 없음 |
| ifcfg 설정 | ✅ 이상 없음 |
| 라우팅 테이블 | ✅ 게이트웨이 정상 |

서버 NIC에 불도 다 들어오고 인터페이스도 전부 up인데 통신이 안 된다 → **서버 밖, 즉 스위치 쪽 문제로 판단**했습니다.

---

## Part 3: 스위치 쪽 조치와 해결

스위치 포트는 **GBIC 모듈 + UTP 케이블** 구조였습니다. GBIC 모듈과 UTP 케이블을 **뺐다가 다시 꽂으니** 게이트웨이로 ping이 정상적으로 나갔습니다.

### 왜 이게 고쳐졌을까?

L1/L2 구간의 물리적 접촉 문제는 "완전히 죽지 않고 반쯤 죽는" 경우가 많습니다.

- **접촉 불량**: GBIC 모듈 커넥터가 살짝 헐겁거나 단자가 산화되어 링크는 형성된 것처럼 보이지만 신호 품질이 나빠 프레임이 유실
- **서버 재부팅을 계기로 한 재협상 실패**: 서버가 껐다 켜지면서 NIC가 재초기화되고 오토네고시에이션을 다시 하는데, 이때 모듈/포트 쪽 상태가 깨끗하지 않으면 협상이 어중간하게 끝남 (링크 up이지만 실제로는 통신 불가)
- 재장착하면 모듈과 포트가 **재인식되면서 협상이 강제로 다시 트리거**되고, 이 과정에서 문제 상태가 리셋됨

즉, "서버를 재부팅했다"는 게 원인이 아니라 **원래 접촉 상태가 경계선에 있던 물리 구간에 재부팅 시 재협상이 얹히면서 증상이 표면화**된 것으로 보는 게 자연스럽습니다.

---

## Part 4: 다음에 같은 일이 생기면 - 추가로 알아두면 좋은 진단

이번에는 서버 내부 점검 → 스위치 재장착으로 해결했지만, 그 사이에 **"L2 문제다"라고 확신을 더 빨리 줄 수 있는 진단법**들이 있습니다. 다음에 비슷한 일이 생기면 아래를 함께 확인하려고 합니다.

### 4.1 ping 에러 메시지 구분하기

게이트웨이로 ping을 보낼 때 메시지가 다릅니다.

```bash
ping 192.168.10.1
# "Destination Host Unreachable" → ARP가 안 풀림 = L2 문제 (가까운 구간 단절)
# 그냥 timeout (응답 없음) → 패킷은 나가는데 응답이 안 옴
```

**"Destination Host Unreachable"이 서버 자신에게서 뜬다면, 게이트웨이의 MAC 주소(ARP)를 못 찾았다는 뜻이라 L2 단절을 강력히 시사합니다.** 이번 사례가 정확히 이 케이스였을 겁니다.

### 4.2 ARP 테이블 확인

```bash
ip neigh        # 또는 arp -n
# 192.168.10.1 dev eth0 lladdr xx:xx:xx:xx:xx:xx REACHABLE   ← 정상
# 192.168.10.1 dev eth0  INCOMPLETE                          ← ARP 실패 = L2 문제
```

게이트웨이 엔트리가 `INCOMPLETE`/`FAILED`로 남아 있으면 프레임이 상대에게 아예 안 닿는다는 뜻입니다.

### 4.3 ethtool은 Speed/Duplex까지 볼 것

`Link detected: yes`만 보고 넘어가지 말고 협상 결과를 봅니다.

```bash
ethtool eth0
# Speed: 1000Mb/s, Duplex: Full  ← 정상 기대값
# Speed: 10Mb/s / Duplex: Half   ← 협상이 깨진 상태. 링크는 up이지만 통신 불량
```

링크는 올라와 있는데 협상이 어중간하게 끝난 경우(특히 Half duplex) 재부팅 후 종종 보입니다.

### 4.4 인터페이스 에러 카운터 확인

```bash
ethtool -S eth0 | egrep -i 'err|drop|crc|fifo'
ip -s link show eth0
```

`rx_crc_errors`, `rx_errors`, `drops`가 계속 증가하고 있다면 물리 구간(케이블/모듈/포트)의 신호 품질 문제를 지시합니다. **카운터를 두 번 찍어서 "증가 추세"를 보는 게 핵심**입니다.

### 4.5 링크 이벤트 로그 확인

```bash
dmesg | grep -iE 'eth|link'
journalctl -b | grep -iE 'link is|NIC'
```

부팅 시점에 링크 업/다운이 반복되었거나, NIC 드라이버 초기화 중 경고가 있었는지 볼 수 있습니다.

### 4.6 패킷이 실제로 나가는지 직접 보기 (결정타)

```bash
tcpdump -i eth0 arp or icmp -n
```

- ARP Request가 계속 나가는데 Reply가 안 들어온다 → **서버→스위치 구간에서 프레임이 죽고 있다는 확정 증거.** 이 시점부터는 서버가 아니라 스위치/케이블/모듈을 봐야 합니다.

### 4.7 스위치 쪽 확인 포인트

스위치에 접근할 수 있다면:

- 해당 포트 상태(up/down, err-disabled 아닌지)
- 포트 카운터의 CRC/input error 증가 여부
- GBIC 모듈 인식 상태(시리얼/온도 등 DDM 정보)

---

## Part 5: 정리 - 재부팅 후 통신 불가 체크리스트

| # | 확인 | 명령어 | 이상 시 의미 |
|---|------|--------|--------------|
| 1 | 인터페이스/IP | `ip a` | L1/L3 기본 |
| 2 | 링크 + 협상 결과 | `ethtool eth0` (Speed/Duplex까지) | 어중간한 협상 |
| 3 | 관리 서비스 | `systemctl status NetworkManager network` | 설정 충돌 |
| 4 | 설정 파일 | `cat /etc/sysconfig/network-scripts/ifcfg-*` | 값 오류/중복 |
| 5 | 라우팅 | `netstat -rn` | 게이트웨이 유실 |
| 6 | ping 에러 구분 | `ping <GW>` | Unreachable이면 L2 |
| 7 | ARP | `ip neigh` | INCOMPLETE면 L2 |
| 8 | 에러 카운터 | `ethtool -S`, `ip -s link` | 물리 신호 불량 |
| 9 | 패킷 덤프 | `tcpdump -i eth0 arp` | 프레임 단절 확정 |
| 10 | 물리 재장착 | 케이블/GBIC 재장착, 포트 변경 | 접촉 불량 |

## 마치며

이번 사례의 교훈을 한 줄로 요약하면:

> **링크가 up이라는 것과 통신이 된다는 것은 다른 이야기다.**

서버 안(IP, 라우팅, 설정, 서비스)이 전부 정상이라면 남는 건 L2 물리 구간입니다. ping의 에러 메시지와 ARP 상태(`ip neigh`)만 봐도 "게이트웨이 MAC을 못 찾는 L2 문제"인지는 금방 확인할 수 있으니, 다음에는 재장착 전에 이 두 가지를 먼저 찍어보고 스위치 쪽을 확신한 뒤 손대려고 합니다.
