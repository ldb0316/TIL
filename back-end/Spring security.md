### 토큰 생성 흐름

Controller 에서 loginProcess 호출

loginProcess에서 SpringSecurity의 AuthenticationManager를 통해 authenticate 호출.
인자로는 UserNamePasswordAuthenticationToken(아이디,비밀번호) 전달

ProviderManager가 각각의 AuthenticationProvider 에게 순서대로 authenticate 요청

DaoAuthenticationProvider 실행되고 authentication 을 받을 수 있다.
이후 authentication에서 정상 여부 확인을 하고 비정상일 경우 Exception 처리.
정상이라면 UserDetails 를 뽑아 Jwts라이브러리와 유저정보를 이용해서 jwtToken을 만들고 response로 전달한다.

### 사용자 정보 커스텀이나 비밀번호, 아이디가 아닌 다른 로그인 방식을 사용하기 위한 방법

#### 1. 커스텀 UserDetails를 만든다.

UserDetails인터페이스를 구현한다.

```java
@Getter
@Builder
@AllArgsConstructor
public class CustomUserDetails implements UserDetails {

	public static CustomUserDetails create(UserInfo userInfo) {
	return CustomUserDetails.builder().
			...
			.build();
}
```

#### 2. CustomUserDetailsService 구현.

UserDetailsService인터페이스를 구현한다.

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

  private final UserRepository userRepository;

  @Transactional(readOnly = true)
  @Override
  public UserDetails loadUserByUsername(String loginId) throws UsernameNotFoundException {
    Optional<UserInfo> optUserInfo = userRepository.findByLoginId(loginId); // DB에서 원하는 사용자 정보 조합을 가져오는 로직 작성

    return optUserInfo.map(CustomUserDetails::create).orElseThrow(() -> new UsernameNotFoundException("로그인 아이디 " + lgnId + "가 존재하지 않습니다."));
  }

}
```

#### 3. SecurityConfig에 설정

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

	...
	private final CustomUserDetailsService customUserDetailsService;
	...

  @Bean
  public AuthenticationProvider authenticationProvider() {
    DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
    authProvider.setUserDetailsService(customUserDetailsService);// UserDetailsService 커스텀설정
    authProvider.setPasswordEncoder(passwordEncoder());
    return authProvider;
  }

	...
}
```

### 이후 처리과정

1. SecurityContext에 인증 정보 저장

```java
SecurityContextHolder.getContext().setAuthentication(authentication);
```

2. Jwts builder 를 이용하여 authentication의 UserDetails를 토큰화 후 response로 전달

3. 클라이언트단에서 새로운 request 요청이 발생하면 securityFilterChain 동작.<br>기타 필터 내용은 기술안함.

```java
// SecurityConfig.java
  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.csrf(csrf -> csrf.disable())
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authenticationProvider(authenticationProvider())
        .exceptionHandling(exceptionHandling -> exceptionHandling.authenticationEntryPoint(customSecurityExceptionHandler) // AuthenticationEntryPoint는 인증되지
                                                                                                                     // 않은 사용자가 보안된 리소스에 접근하려 할 때 호출
																													 // customSecurityExceptionHandler 구현 후 Component로 등록하여 spring bean으로 만들어 전달함
            .accessDeniedHandler(customSecurityExceptionHandler)) // AccessDeniedHandler는 인증된 사용자가 권한이 없는 리소스에 접근하려 할 때 호출
        .addFilterBefore(new GlobalExceptionTranslationFilter(securityExceptionHandler), UsernamePasswordAuthenticationFilter.class) // GlobalExceptionTranslationFilter를 구현 후, UsernamePasswordAuthenticationFilter보다 앞에서 동작하게 하여 예외를 JSON응답으로 돌려보낸다.
        .addFilterAfter(new JwtAuthenticationFilter(tokenProvider, customUserDetailsService, isSingleActiveToken),
            GlobalExceptionTranslationFilter.class) // GlobalExceptionTranslationFilter 이후 Jwt 유효성검증 등 수행하는 필터 JwtAuthenticationFilter 수행하도록 지정. JwtAuthenticationFilter에서 401 return
        .addFilterAfter(new DynamicAuthorizationFilter(menuService, aopLogService, maxRequests), JwtAuthenticationFilter.class) // JwtAuthenticationFilter이후에 인가 처리
        .anonymous()
        .disable(); // 익명 인증 비활성화. 로그인하지않은사용자는 SecurityContext에 Authentication 전달안하도록.

    return http.build();
  }

  ...

  @Bean // 특정 도메인에서의 API 접근 허용
  public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();
    configuration.setAllowedOrigins(allowedOrigins);
    configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    configuration.setAllowedHeaders(Arrays.asList("Origin", "X-Requested-With", "Content-Type", "Accept", "Authorization", "X-CSRF-TOKEN", "Cache",
        "X-Forwarded-For", "X-Client-IP", "X-Mbtl-Cert-Key-No", "X-Mbtl-Ci-No", "X-Mbtl-User-Nm", "X-Mbtl-Num"));
    configuration.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
  }
```
