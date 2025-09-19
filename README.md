# blog
tech blog

쇼핑몰/콘서트 예약 서비스 트래픽 대응 최적화
※ 개발환경 : SpringBoot, Java, JPA, Mysql (8.0), JUnit5
1) 쇼핑몰 웹 솔루션 백엔드 개발
- 포인트 충전·조회, 상품 주문·결제, 선착순 쿠폰, 콘서트 예약 API 설계
- 대기열 토큰 예약시스템: UUID 토큰 기반 순번 관리, 최대 100명 동시 활성화 제한, 토큰 순차 활성화
- 좌석 상태 관리: RESERVE 상태 3단계 전환, 5분 임시예약 후 결제 미완료시 자동 해제
- TDD 기반 테스트 작성 및 클린 아키텍처 적용 및 JUnit5+AssertJ 기반 단위/통합 테스트
2) 성능 최적화 및 동시성 처리
- Redis 캐싱 성능 최적화: 좌석 배치도 Cache-Aside 패턴(2시간 TTL) 으로 DB 부하감소
- Redis 분산락 동시성 제어: Redisson 기반 동일좌석예약시 분산락 적용, 재고상품주문, 쿠폰발급 경쟁조건 동시성제어
- 다중 인스턴스 환경에서 재고 차감과 쿠폰 발급 정확성 보장
3) 이벤트 처리 및 안정성
- Kafka 활용 이벤트 메시지 송수신 구현
- 부하 테스트 및 장애 시나리오 대응을 통한 안정성 확보
- DB 성능 분석 및 최적화: 복합 인덱스, 테이블 파티셔닝으로 좌석 조회속도 개선, 동시성테스트로 장애 시나리오 사전 검증
- Testcontainers 격리 테스트 : MySQL 8.0 컨테이너 기반환경에서 데드락, 락 충돌, 타임아웃 시나리오 테스트로 운영 환경 안정성 검증

========================================



========================================
  // 1. DTO 객체 생성 (메모리)
  SeatLayoutDto dto = new SeatLayoutDto(...);
  // 2. RedisTemplate이 Redis 서버(localhost:6379)에 저장
  redisTemplate.opsForValue().set("key", dto, Duration.ofHours(2));
  // → dto를 JSON으로 변환 → TCP/IP로 Redis 서버 전송 → Redis 메모리에 저장
  // 3. Redis 서버에서 가져오기
  Object cached = redisTemplate.opsForValue().get("key");
  // → Redis 서버에서 JSON 조회 → DTO 객체로 변환 → 반환
  RedisTemplate = Redis 서버와 통신하는 도구
  1. 첫 요청:
     클라이언트 → Spring → Redis(없음) → DB 조회 → DTO 변환 → Redis 저장 → 응답
  2. 두 번째 요청 (2시간 내):
     클라이언트 → Spring → Redis(있음) → 바로 응답 (DB 접근 X)

  // SeatLayoutDto 구조 (SeatCacheService.java:126-169):
  {
    "seatId": 456,
    "seatNumber": 12,
    "seatGrade": "VIP",
    "price": 150000,
    "rowNumber": 3,
    "columnNumber": 4,
    "isAvailable": true
  }
 이 JSON 데이터가 Redis 메모리에 2시간 동안 저장되어, DB 부하를 줄이고 응답 속도를 높입니다.
 캐시 무효화는 예약/결제 시 seatCacheService.invalidateSeatLayout()로 삭제합니다.

 // RedisTemplate 설정 (RedisConfig.java:30-57):
  @Bean
  public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
      RedisTemplate<String, Object> template = new RedisTemplate<>();
      template.setConnectionFactory(connectionFactory);  // Redis 서버 연결
      // JSON 직렬화 설정
      GenericJackson2JsonRedisSerializer jackson2JsonRedisSerializer =
          new GenericJackson2JsonRedisSerializer(objectMapper);
      template.setKeySerializer(new StringRedisSerializer());
      template.setValueSerializer(jackson2JsonRedisSerializer);  // 객체→JSON 변환
      return template;

============================
 // 분산락 함수  (RedisDistributedLock.java:32-56):
  public <T> T executeWithLock(String lockKey, long waitTime, long leaseTime, Supplier<T> supplier) {
      RLock lock = redissonClient.getLock(lockKey);  // Redis에 락 생성
      boolean isLockAcquired = lock.tryLock(waitTime, leaseTime, TimeUnit.SECONDS);
      // waitTime: 락 획득 대기 시간
      // leaseTime: 락 보유 시간
      if (!isLockAcquired) {
          throw new IllegalStateException("Could not acquire lock");
      }
      return supplier.get();  // 비즈니스 로직 실행
      // finally 블록에서 자동으로 lock.unlock()

  // 실제 사용 - 좌석 예약 (ReservationUseCase.java:44-93):
  public ReservationResult reserveSeat(ReserveSeatCommand command) {
      String lockKey = "seat:reserve:" + command.getSeatId();  // 좌석별 락
      return distributedLock.executeWithLock(lockKey, 3, 10, () -> {    //####################### 
          // 이 블록은 한번에 한 스레드만 실행 가능
          // 1. 좌석 조회
          Seat seat = seatRepository.findById(command.getSeatId());
          // 2. 예약 가능 확인
          if (!seat.isAvailable()) {
              throw new IllegalStateException("Seat is not available");
          }
          // 3. 좌석 예약 처리
          seat.reserve(command.getUserId(), 5);
          seatRepository.save(seat);
          return new ReservationResult(...);
      });
  }

  // 분산락 동작 시나리오:
  사용자 A, B가 동시에 좌석 #12 예약 시도:
  1. A 요청 → "seat:reserve:12" 락 획득 ✓ → 예약 진행
  2. B 요청 → "seat:reserve:12" 락 대기 (3초까지)
  3. A 완료 → 락 해제
  4. B 진입 → 좌석 이미 예약됨 확인 → 예외 발생
  결제 처리 분산락 (ReservationUseCase.java:96-155):
  String lockKey = "payment:" + command.getReservationId();  // 예약건별 락
  distributedLock.executeWithLock(lockKey, 3, 10, () -> {    //###################### 
      // 중복 결제 방지
      userBalanceService.deductBalance(...);  // 잔액 차감
      paymentService.processPayment(...);     // 결제 처리
      reservation.confirm();                   // 예약 확정
  });
  핵심: 동일한 좌석/예약에 대한 동시 요청을 순차 처리하여 데이터 정합성 보장

  

      





  

