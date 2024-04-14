---
title: "Final Project A: 콘서트 예약 프로그램"
date: "2024-04-14"
author: "이어진, 정채원, 주현정, 홍성주 (✍️ 유빈)"
tags: ["프로젝트"]
categories: ["Final Project"]
---

# Final Project A: 콘서트 예약 프로그램

## 1️⃣ 프로젝트 아키텍처

---

- 외부 사용자 또는 클라이언트는 API Gateway를 통해 서비스에 액세스합니다.
- 각 서비스는 독립적으로 배포되며, EC2 인스턴스에서 실행됩니다.
- 서비스 별 데이터는 각 RDS 인스턴스에 저장되며, 프라이빗 서브넷에 배치됩니다.
- API Gateway를 통해 서비스 간 통신이 이루어지며, Gateway는 각 서비스의 IP 및 포트에 대한 정보를 갖고 있습니다.
- JWT 토큰을 사용하여 인증 및 권한을 부여하고, 서브넷 및 보안 그룹으로 네트워크 보안을 강화했습니다.
- 유레카 서버는 서비스의 등록 및 검색을 관리하며, 서비스 Discovery를 지원합니다.

![Untitled](/final_a/Untitled.png)

## 2️⃣ 프로젝트 설명

---

**MSA 아키텍처**를 사용 : 서비스를 마이크로 단위로 분리 

![스크린샷 2024-04-14 오후 12.16.53.png](/final_a/1.png)
- User Service : 유저 관련 서비스 담당
- Concert Service : 콘서트 관련 서비스 담당
- Reserve Service : 선착순 예약 서비스 담당
- Eureka Service : 디스커버리 관련 서비스 담당
- Gateway Service : 라우팅 관련 서비스 담당

## 3️⃣ 주요 서비스 기능 설명

---

### Reserve Service

핵심인 선착순 콘서트 예약 = **`Redis`** 로 구현

***🔎  Redis를 사용하는 이유?*  🔍**

> ***보통 콘서트 예약과 같은 `실시간 처리가 중요한` 서비스의 경우,***
특정 시간에 트래픽이 몰려 서버가 다운되거나 원활하지 않은 이벤트 처리가 되는 경우가 많습니다.
> 

1.  Redis에서 제공하는 자료구조 Sorted Set 활용
2. 모든 요청이 DB로 가지 않아 부하를 덜 수 있고, 차례대로 일정 범위만큼 처리 할 수 있다는 장점

<aside>
📌 `**Redis Sorted Set**`은 !!!

- 한 Key에 대해 여러 value와 score를 가지고 있으며,
- **중복되지 않는 value를 통해 score순으로 데이터를 정렬하는 자료구조 입니다.**
</aside>

![Untitled](/final_a/Untitled%201.png)

<aside>
💡 ***우리 프로젝트 적용 방법***

✔️ Sorted Set의 Key는 Event로 설정

✔️ Value : 사용자명

✔️ Score : 이벤트에 참여한 시간을 유닉스타임(m/s) 값으로 설정

→ 참여한 사람들을 순서대로 정렬

</aside>

![스크린샷 2024-04-14 오후 12.01.35.png](/final_a/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_12.01.35.png)

**코드 실행 흐름**

> 가정 : 콘서트 티켓 30장 / 참여 유저 100명
> 
1. 100명의 유저가 콘서트 티켓 예약 요청 → 100명의 유저는 대기열에 쌓임
2. 1초마다 동기화
    
    → 예약 성공, 실패 로직 수행
    
    - **성공 시**, 100명의 유저의 접속 순서대로 10명씩 티켓을 발급한다.
    - **실패 시**, 다음 대기열로 돌아가면서 남은 대기열 순번을 표출한다.
3. 해당 과정을 반복하면서 콘서트 티켓 판매(30개 발급 완료)를 종료한다.

<aside>
💡 이때 **10개씩 발급하는 이유**는 ❓

- DB 부하 문제
- 실제 서비스 : 100000개의 요청 중 1000개의 티켓을 발급하는 상황이라 가정, 
1초마다 50개씩 순차적으로 발급하는 방법으로 DB 부하를 줄일 수 있을 것 같습니다.
</aside>

> ***주요 코드***
> 

**EventScheduler.java**

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class EventScheduler {   
    private final TicketService ticketService;

    **@Scheduled(fixedDelay = 1000) // 1초마다 도는 대기열 동기화 스케줄러 구성**
    private void concertEventScheduler(){
        **if(ticketService.validEnd())**{
            log.info("===== 선착순 티켓팅이 종료되었습니다. =====");
            return;
        }
        **ticketService.publish(Event.TICKET);
        ticketService.getOrder(Event.TICKET);**
    }
}

```

- 1초마다 도는 대기열 동기화 스케줄러 구성
- **(예약실패)** 콘서트 티켓 30개가 모두 발급되면 선착순 콘서트 예매 종료
- **(예약성공)** 예매가 종료되지 않은 경우, 티켓을 발급하고 남은 대기열에 순번 표출

**TicketService.java**

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class TicketService {
    private final RedisTemplate<String,Object> redisTemplate;
    private static final long FIRST_ELEMENT = 0;
    private static final long LAST_ELEMENT = -1;
    private static final long PUBLISH_SIZE = 10;
    private static final long LAST_INDEX = 1;
    private EventCount eventCount;

    public void setEventCount(Event event, int queue){
        this.eventCount = new EventCount(event, queue);
    }
    public void addQueue(Event event){
        final String people = Thread.currentThread().getName();
        final long now = System.currentTimeMillis();

        redisTemplate.opsForZSet().add(event.toString(), people, (int) now);
        log.info("대기열에 추가 - {} ({}초)", people, now);
    }

    public void getOrder(Event event){
        final long start = FIRST_ELEMENT;
        final long end = LAST_ELEMENT;

        Set<Object> queue = redisTemplate.opsForZSet().range(event.toString(), start, end);

        for (Object people : queue) {
            Long rank = redisTemplate.opsForZSet().rank(event.toString(), people);
            log.info("'{}'님의 현재 대기열은 {}명 남았습니다.", people, rank);
        }
    }

    public void publish(Event event){
        final long start = FIRST_ELEMENT;
        final long end = PUBLISH_SIZE - LAST_INDEX;

        Set<Object> queue = redisTemplate.opsForZSet().range(event.toString(), start, end);
        for (Object people : queue) {
            final Ticket ticket = new Ticket(event);
            log.info("'{}'님의 {} 티켓이 발급되었습니다 ({})",people, ticket.getEvent().getName(), ticket.getCode());
            redisTemplate.opsForZSet().remove(event.toString(), people);
            this.eventCount.decrease();
        }
    }
    public boolean validEnd(){
        return this.eventCount != null
                ? this.eventCount.end()
                : false;
    }

    public long getSize(Event event){
        return redisTemplate.opsForZSet().size(event.toString());
    }
}

```

- `addQueue()` : 이벤트 요청한 유저들을 추가
    - value : 유저의 고유 값
    - score : 현재 시간(m/s)으로 대기열에 추가
- `getOrder()` : 차례대로 들어온 유저들의 요청을 기반으로 대기열 순번 표출
- `publish()` : 1초마다 콘서트 예매에 참여하는 사람수(10명)씩 콘서트 티켓을 발급 후 대기열에서 제거

**TicketServiceTest.java**

```java
@SpringBootTest
class TicketServiceTest {
    @Autowired
    private TicketService ticketService;

    @Test
    void 선착순_100명에게_티켓_30개_제공() throws InterruptedException {
        final Event ticketEvent = Event.TICKET;
        final int people = 100;
        final int limitCount = 30;
        final CountDownLatch countDownLatch = new CountDownLatch(people);

        ticketService.setEventCount(ticketEvent, limitCount);

        List<Thread> workers = Stream
                .generate(() -> new Thread(new AddQueueWorker(countDownLatch, ticketEvent)))
                .limit(people)
                .toList();
        workers.forEach(Thread::start);
        countDownLatch.await();
        **Thread.sleep(5000); //발급 스케줄러 작업 시간**

        final long failEventPeople = ticketService.getSize(ticketEvent);
        assertEquals(people - limitCount, failEventPeople); // output : 70 = 100 - 30
    }

    private class **AddQueueWorker** implements Runnable{
        private final CountDownLatch countDownLatch;
        private final Event event;

        public AddQueueWorker(CountDownLatch countDownLatch, Event event) {
            this.countDownLatch = countDownLatch;
            this.event = event;
        }

        @Override
        public void run() {
            ticketService.addQueue(event);
            countDownLatch.countDown();
        }
    }
}

```

- 대기열에 사람들을 추가하는 `AddQueueWorker` 생성
- 1초마다 도는 티켓 발급, 남은 대기열 동기화 스케줄러의 작업을 위해 `Thread.sleep(5000)` 설정

> **♦️ 결과 ♦️**
100명의 사용자 요청, 30개의 콘서트 티켓 발급을 제외하면 70명이 콘서트 티켓을 받지 못한 것을 확인할 수 있다.
> 

🎥 ***Reserve Service 시연영상***

🔎 

### Concert Service

- 콘서트 생성 및 조회
- RDS, EC2 를 사용하였다.
- EC2 서버에서 콘서트 생성 및 조회하면 EC2 내부의 concert RDS로 접근하여 C, R 구현

![프로젝트 구조 ](/final_a/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_11.41.01.png)

프로젝트 구조 

> ***주요 코드***
> 

**ConcertRepository.java**

```java
@Repository
public interface ConcertRepository extends JpaRepository<Concert, Long> {
    Optional<Concert> findByTitle(String title);
}
```

- 제목으로 콘서트를 조회하는 Jpa 코드입니다.

**ConcertService.java**

```java
@Service
public class ConcertService {

    private final ConcertRepository concertRepository;

    public ConcertService(ConcertRepository concertRepository) {
        this.concertRepository = concertRepository;
    }

    // R) 제목으로 콘서트 가져오기
    public ConcertResponse getConcertByTitle(String title) {
        Optional<Concert> concert = concertRepository.findByTitle(title);
        ConcertResponse concertResponse = null;
        if (concert.isPresent()) {
            concertResponse = ConcertResponse.of(concert.get());
        }
        return concertResponse;
    }

    // C) 콘서트 생성
    public Concert createConcert(ConcertDto concertDto) throws Exception {
        try {
            Concert concert = Concert.builder()
                    .title(concertDto.getTitle())
                    .genre(concertDto.getGenre())
                    .startDate(concertDto.getStartDate())
                    .endDate(concertDto.getEndDate())
                    .price(concertDto.getPrice())
                    .description(concertDto.getDescription())
                    .runningTime(concertDto.getRunningTime())
                    .build();
            return concertRepository.save(concert);
        } catch(Exception e) {
            System.out.println(e.getMessage());
            throw new Exception("잘못된 요청입니다.");
        }
    }

}
```

- 콘서트 생성, 조회( C, R ) 를 구현한 서비스 코드입니다

<aside>
➕ ***참고사항***

아래 모든 시연 화면들은 EC2 서버를 중단한 상태이므로 local로 실행한 테스트입니다. 

</aside>

📍 ***콘서트 생성*** 

1. 콘서트 정보를 RequestBody로 기입 후 `/concert/create` POST 요청
2. DB(RDS)에 콘서트가 생성

![스크린샷 2024-04-14 11.33.47.png](/final_a/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_11.33.47.png)

📍 ***콘서트 제목으로 조회*** 

1. RequestParam에 조회하고자 하는 콘서트이름을 넣어 `/concert/title` GET 요청
2. 해당하는 콘서트 정보가 반환 

![스크린샷 2024-04-14 11.34.45.png](/final_a/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_11.34.45.png)

### User Service

- 회원 정보 서비스
- 로그인, 회원가입, 로그아웃 JWT 로 구현
- EC2, RDS, Redis 사용

![구조](/final_a/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_11.41.50.png)

구조

> ***주요 코드***
> 

**UserService.java**

```java
@Service
@Transactional
@RequiredArgsConstructor
public class UserService {

    private final UserRepository memberRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtProvider jwtProvider;
    private final RedisTemplate redisTemplate;
    private final RefreshTokenRepository refreshTokenRepository;

		// 로그인 
    public SignResponse login(SignRequest request) throws Exception {
        User member = memberRepository.findByAccount(request.getAccount()).orElseThrow(() ->
                new BadCredentialsException("잘못된 계정정보입니다."));

        if (!passwordEncoder.matches(request.getPassword(), member.getPassword())) {
            throw new BadCredentialsException("잘못된 계정정보입니다.");
        }

        return SignResponse.builder()
                .id(member.getId())
                .account(member.getAccount())
                .name(member.getName())
                .email(member.getEmail())
                .nickname(member.getNickname())
                .roles(member.getRoles())
                .token(jwtProvider.createToken(member.getAccount(), member.getRoles()))
                .build();

    }

    //로그아웃
    @Transactional
    public void logout(String refreshToken, String accessToken){

        Optional<Long> findUserId = jwtProvider.getUserIdToToken(accessToken);

        //액세스 토큰 남은 유효시간
        Long expiration = jwtProvider.getExpiration(accessToken);

        //리프레시 토큰 남은 유효시간
        Long refreshExpiration = jwtProvider.getExpiration(refreshToken);

        // 액세스 토큰 유효시간이 남았을 경우에만 로그아웃 수행
        if (expiration > 0) {
            // 액세스 토큰을 만료시킴
            // Redis Cache에 저장
            redisTemplate.opsForValue().set(accessToken, "logout", expiration, TimeUnit.MILLISECONDS);
        }

        // 리프레시 토큰이 유효할 경우에만 삭제
        if (refreshExpiration > 0) {
            // 리프레시 토큰 삭제
             refreshTokenRepository.deleteByUserId(findUserId.get());
        }
    }

// 회원가입 
    public boolean register(SignRequest request) throws Exception {
        try {
            User member = User.builder()
                    .account(request.getAccount())
                    .password(passwordEncoder.encode(request.getPassword()))
                    .name(request.getName())
                    .nickname(request.getNickname())
                    .email(request.getEmail())
                    .build();

            member.setRoles(Collections.singletonList(Authority.builder().name("ROLE_USER").build()));

            memberRepository.save(member);
        } catch (Exception e) {
            System.out.println(e.getMessage());
            throw new Exception("잘못된 요청입니다.");
        }
        return true;
    }

// 회원 조회 
    public SignResponse getMember(String account) throws Exception {
        User member = memberRepository.findByAccount(account)
                .orElseThrow(() -> new Exception("계정을 찾을 수 없습니다."));
        return new SignResponse(member);
    }

}
```

- 회원가입, 로그인, 로그아웃, 회원 조회가 이루어지는 서비스 코드입니다.

**SecurityConfig.java**

```java
@Slf4j
@Configuration
@RequiredArgsConstructor
@EnableWebSecurity
public class SecurityConfig {

    private final JwtProvider jwtProvider;
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .csrf((csrf) -> csrf.disable())
                // CORS 설정
                .cors(c -> {
                    CorsConfigurationSource source = request -> {
                        // Cors 허용 패턴
                        CorsConfiguration config = new CorsConfiguration();
                        config.setAllowedOrigins(
                                List.of("*")
                        );
                        config.setAllowedMethods(
                                List.of("*")
                        );
                        return config;
                    };
                    c.configurationSource(source);
                })
                // 세션 관리 설정 제거 및 세션 생성 정책 설정
                .sessionManagement(session -> session
                        .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                )
                // 조건별로 요청 허용/제한 설정
                .authorizeRequests(authorize -> authorize
                        // 회원가입과 로그인은 모두 승인
                                .requestMatchers(PathRequest.toStaticResources().atCommonLocations()).permitAll()
                                .requestMatchers(new AntPathRequestMatcher("/member/register", "POST")).permitAll()
                                .requestMatchers(new AntPathRequestMatcher("/member/login", "POST")).permitAll()
                                .requestMatchers(new AntPathRequestMatcher("/member/logout", "PATCH")).permitAll()

                                // /admin으로 시작하는 요청은 ADMIN 권한이 있는 유저에게만 허용
                                .requestMatchers("/admin/**").hasRole("ADMIN")
                                // /user로 시작하는 요청은 USER 권한이 있는 유저에게만 허용
                                .requestMatchers("/user/**").hasRole("USER")
                                .anyRequest().authenticated()
                                // 나머지 요청은 모두 거부
//                                .anyRequest().denyAll()
                )
                // JWT 인증 필터 적용
                .addFilterBefore(new JwtAuthenticationFilter(jwtProvider), UsernamePasswordAuthenticationFilter.class)
                // 에러 핸들링
                .exceptionHandling(exception -> exception
                        .accessDeniedHandler((request, response, accessDeniedException) -> {
                            log.error("Access Denied: {}", accessDeniedException.getMessage());
                            // 권한 문제가 발생했을 때 이 부분을 호출한다.
                            response.setStatus(403);
                            response.setCharacterEncoding("utf-8");
                            response.setContentType("text/html; charset=UTF-8");
                            response.getWriter().write("권한이 없는 사용자입니다.");
                        })
                        .authenticationEntryPoint((request, response, authException) -> {
                            log.error("Authentication Error: {}", authException.getMessage());
                            // 인증문제가 발생했을 때 이 부분을 호출한다.
                            response.setStatus(401);
                            response.setCharacterEncoding("utf-8");
                            response.setContentType("text/html; charset=UTF-8");
                            response.getWriter().write("인증되지 않은 사용자입니다.");
                        })
                );
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```

- JWT 관련 설정을 한 코드 입니다.

**JwtAuthenticationFilter.java**

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private static final String AUTHORIZATION_HEADER = "Authorization";
    private static final String BEARER_TYPE = "Bearer";

    private final JwtProvider jwtProvider;
    private ObjectMapper objectMapper;
    private UserService userService;
    private RedisTemplate<String, String> redisTemplate;
    private AuthenticationManager authenticationManager;

    public JwtAuthenticationFilter(JwtProvider jwtProvider) {
        this.jwtProvider = jwtProvider;
    }

    public JwtAuthenticationFilter(JwtProvider jwtProvider, ObjectMapper objectMapper,
                                   UserService userService, RedisTemplate<String, String> redisTemplate) {
        this.jwtProvider = jwtProvider;
        this.objectMapper = objectMapper;
        this.userService = userService;
        this.redisTemplate = redisTemplate;
    }

    public void setAuthenticationManager(AuthenticationManager authenticationManager) {
        this.authenticationManager = authenticationManager;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        String token = jwtProvider.resolveToken(request);

        if (token != null && jwtProvider.validateToken(token)) {
            // (추가) Redis 에 해당 accessToken logout 여부 확인
            String isLogout = (String)redisTemplate.opsForValue().get(token);
            if (ObjectUtils.isEmpty(isLogout)) {
                // 토큰이 유효할 경우 토큰에서 Authentication 객체를 가지고 와서 SecurityContext 에 저장
                Authentication authentication = jwtProvider.getAuthentication(token);
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
//            // check access token
//            token = token.split(" ")[1].trim();
//            Authentication auth = jwtProvider.getAuthentication(token);
//            SecurityContextHolder.getContext().setAuthentication(auth);
        }

        filterChain.doFilter(request, response);

    }

    // Request Header 에서 토큰 정보 추출
    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader(AUTHORIZATION_HEADER);
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith(BEARER_TYPE)) {
            return bearerToken.substring(7);
        }
        return null;
    }

}
```

- Jwt가 유효성을 검증하는 Filter 코드입니다.

**RedisConfig.java**

```java
@Configuration
public class RedisConfig {

    @Value("${redis.host}")
    private String redisHost;

    @Value("${redis.port}")
    private int redisPort;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(redisHost, redisPort);
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate() {
        RedisTemplate<String, String> redisTemplate = new RedisTemplate<>();
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new StringRedisSerializer());
        redisTemplate.setConnectionFactory(redisConnectionFactory());

        return redisTemplate;
    }
}

```

- Redis 설정 정보를 포함하고 있는 코드입니다.

**📍 *회원가입*** 

1. 회원 정보를 RequestBody로 `/member/register` POST
2. `200OK` 로 “true” 반환 

![스크린샷 2024-04-14 11.40.11.png](/final_a/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_11.40.11.png)

**📍 *로그인*** 

1. RequestBody로 ID/PW 를 `/member/login` POST 요청
2. 토큰이 발생함과 동시에 DB(RDS)에 회원이 등록

![스크린샷 2024-04-14 11.55.14.png](/final_a/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_11.55.14.png)

→ 여기서 반환된 토큰을 복사 

**📍 *회원 확인 (조회)*** 

1. Param으로 ID를 포함해서 토큰과 함께 `/member/user/get` GET 요청 
    1. 로그인할 때 복사한 토큰을 Authorization에 Token 부분에 기입 
2. 200OK 로 회원 정보가 반환

![성공](/final_a/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_12.05.42.png)

성공

🔻 토큰을 기입하지 않으면 401 에러 발생

![실패 ](%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%84%8C%E1%85%A6%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A9%E1%86%BC%20%E1%84%87%E1%85%A1%E1%86%AF%E1%84%91%E1%85%AD%205d8987967d46459abea5047a369b6450/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_12.05.58.png)

실패 

**📍 *로그아웃***

1. 토큰을 Header에 기입하고 `/member/logout`으로 PATCH 요청

![스크린샷 2024-04-14 12.19.54.png](/final_a/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_12.19.54.png)

1. Redis 접속해서 확인 (로컬이라서 `127.0.0.1:6379` 포트 이용)

![스크린샷 2024-04-14 12.22.55.png](/final_a/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_12.22.55.png)

→ `KEYS *` 명령어를 입력하면, request 시 보낸 access token이 Redis에 잘 저장된 것을 확인

- `KEYS` 명령어는 key-value 중 `key`만 보여준다.

1. Key 만료시간 확인 

![스크린샷 2024-04-14 12.24.13.png](/final_a/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_12.24.13.png)

- `TTL` 명령어로 만료시간을 확인 (유효 시간 : 3600sec)
- `3162`이라고 나오는 건 만료될 때까지 3162sec 남았다는 뜻

1. value 확인 

![스크린샷 2024-04-14 12.26.23.png](/final_a/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-04-14_12.26.23.png)

→ 토큰에 대한 value 로 “logout” 이 설정되었음

1. 그 후로 ElasticCache를 연동하여 로그아웃 구현을 마무리 했어야 했으나, 저희 팀 AWS 비용 이슈로 전체적인 서비스들을 종료시켜놔서 이 부분 구현은 중단 되었습니다.. 

## 4️⃣ Github

---

[](https://github.com/orgs/ACC-TEAM-A/repositories)