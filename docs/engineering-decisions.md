# Engineering decisions

## 1. 승차역 발차 확인

iOS 앱과 서버는 승차역의 실제 발차 또는 같은 운행일 시각표상 승차역 이후 진행을 확인하고 운행 중 표시로 바꿉니다. 발차를 확인한 뒤에는 그 상태를 유지합니다. 예정 시각 경과, 역 접근·도착, 데이터 누락은 발차 증거로 쓰지 않습니다. 판정 코드와 데모 입력별 결과는 [README](../README.md#판단-근거가-보이는-코드)에 있습니다.

## 2. Prevent stale asynchronous commits

역을 빠르게 바꾸거나 토큰 등록이 겹치면 먼저 시작한 요청이 나중에 끝날 수 있습니다. 단순히 마지막으로 완료된 결과를 저장하면 UI와 구독이 과거 상태로 되돌아갑니다.

각 작업 시작 시 증가하는 generation/revision을 발급하고, 완료 시 현재 값과 일치하는 작업만 commit합니다.

## 3. Coalesce in-flight requests, not only cached values

동일한 열차를 앱 요청과 서버 폴러가 동시에 조회할 수 있습니다. TTL 값만 캐시하면 cache miss 순간에는 같은 upstream 요청이 여러 번 발생합니다.

서버 캐시는 완성된 값과 별도로 진행 중인 요청을 보관합니다. 같은 키의 호출자는 한 조회를 공유하고, 성공 여부와 무관하게 작업이 끝나면 그 항목을 지웁니다. 조회 오류는 캐시하지 않아 일시 장애가 ‘빈 위치’로 굳지 않습니다. 서버 QA의 캐시 경계 테스트가 중복 조회 병합과 오류 비캐시를 검사합니다.

## 4. Keep confirmed and inferred location separate

좌표로 계산한 최근접역은 유용하지만 실제 운행 상태와 다를 수 있습니다. 이를 확정 현재역처럼 표시하면 정확도 문제보다 신뢰 문제가 더 커집니다.

응답에서 `currentStation`/`nextStation`과 `nearestStation`/거리 값을 분리하고, 확정값이 없을 때만 UI가 추정값을 보조 정보로 표시하도록 했습니다.

## 5. Treat restore and token rotation as normal state transitions

Live Activity는 앱 프로세스보다 오래 살 수 있습니다. 앱 시작 시 단순히 새 항목을 만들지 않고 OS 상태, 로컬 저장과 서버 구독을 대조합니다.

등록 요청에는 revision을 포함해 늦게 도착한 이전 등록이 새 토큰을 덮지 않게 했습니다. 푸시 실패도 즉시 구독 삭제로 연결하지 않고 재시도와 명시적 무효 토큰을 구분했습니다.

## 6. Separate the public contract from private integrations

클라이언트가 데이터 제공자를 직접 호출하면 제공자 변경, 인증, rate limit과 장애 대응이 앱 릴리스에 묶입니다. 앱은 서버의 정규화된 응답만 소비하고 제공자별 처리는 서버 뒤에 둡니다.

포트폴리오 저장소도 같은 경계를 따릅니다. 판단 로직 발췌와 합성 여정은 공개하지만, 실제 provider adapter와 운영 push engine은 포함하지 않습니다.
