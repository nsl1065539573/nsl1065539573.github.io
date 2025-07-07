+++
date = '2025-07-03T15:05:31+08:00'
draft = false
title = 'Spring Security 入门'
+++

## 什么是Spring Security
Spring Security 是 Spring 生态中功能强大的**认证（Authentication）和授权（Authorization）**框架，在 Spring Boot 中集成后可以快速实现安全控制。

### 认证
验证用户身份（如用户名/密码、OAuth2、LDAP等）

### 授权
控制用户访问权限，通过配置或者在接口上添加`@@PreAuthorize`来控制接口访问权限。

### 核心概念
Spring security本质上是一个过滤器链，通过对接口执行前增加过滤链的形式实现权限控制以及认证等。
```graph
graph LR
A[请求] --> B[SecurityFilterChain]
B --> C[认证过滤器]
B --> D[授权过滤器]
B --> E[异常处理过滤器]
B --> F[...其他过滤器]
F --> G[资源访问]
```
而以下核心组件就是用来贯穿责任链的
- UserDetails：存储用户密码、用户名、权限列表、是否可用等，根据此接口的密码方法以及权限列表等来实现认证以及授权
- UserDetailsService: 根据username获取UserDetails
- SecurityContextHolder：security上下文信息，可以存放用户的基本信息以及权限列表等
- PasswordEncoder：密码加密组件，比对密码时使用此方法对前端密码进行加密并且校验是否与UserDetails中的密码一致。

## springboot集成步骤
#### 1. 添加依赖
springboot提供了security的starter依赖，只需要依赖start即可：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
#### 2. 基础配置
通过@Configuration的形式声明密码加密组件以及过滤链bean。
```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
  @Resource
  private JwtAuthenticationFilter jwtAuthenticationFilter;

  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity http, RestAuthEntryPoint restAuthEntryPoint, CustomAccessDeniedHandler customAccessDeniedHandler) throws Exception {
    http
        .csrf(AbstractHttpConfigurer::disable)
        .sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/auth/**").permitAll()
            .anyRequest().authenticated())
        .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
        .exceptionHandling(e -> {
          e.authenticationEntryPoint(restAuthEntryPoint);
          e.accessDeniedHandler(customAccessDeniedHandler);
        });

    return http.build();
  }

  @Bean
  public AuthenticationManager authenticationManager(
      AuthenticationConfiguration config) throws Exception {
    return config.getAuthenticationManager();
  }

  @Bean
  public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();  // 密码加密器
  }
}
```
如上，配置了过滤链，新增了我们自己的异常处理类以及jwt校验的节点。

#### 3. 自定义用户服务
spring security根据`UserDetailsService`来获取用户详情，使用`UserDetails`来承载用户信息，这两个都是spring提供的接口，我们可以实现他们来实现我们自己的用户详情信息。
```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
  private final LoadingCache<String, UserDetails> cache =
      Caffeine.newBuilder()
          .expireAfterWrite(Duration.ofMinutes(10))
          .maximumSize(10_000)
          .build(this::loadUserDetails);

  @Resource
  private UserModel userModel;
  @Resource
  private PermissionService permissionService;

  @Override
  public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    try {
      return cache.get(username);
    } catch (RuntimeException e) {
      throw (UsernameNotFoundException) e.getCause();
    }
  }

  private UserDetails loadUserDetails(String username) {
    UserRecord user = userModel.getByUserName(username);
    System.out.println("查询user" + username);
    if (user == null) {
      throw new UsernameNotFoundException("User not found");
    }
    List<String> perms = permissionService.fetchPermissionsByUserId(user.getId());
    return CustomUserDetails.builder()
        .userId(user.getId())
        .username(username)
        .password(user.getPassword())
        .enabled(user.getDeleted() == (byte)0)
        .permissions(perms == null ? List.of() : perms)
        .build();
  }
}
```

```java
@Getter
@ToString(exclude = "password")
@AllArgsConstructor
public class CustomUserDetails implements UserDetails {
  @Serial
  private static final long serialVersionUID = 1L;

  private final Long userId;
  private final String username;

  @JsonIgnore
  private final String password;

  private final boolean enabled;
  private final List<String> permissions;

  // 缓存好的权限集合
  private final List<? extends GrantedAuthority> authorities;

  @Builder
  public CustomUserDetails(Long userId,
                           String username,
                           String password,
                           Boolean enabled,
                           List<String> permissions) {
    this.userId = userId;
    this.username = username;
    this.password = password;
    this.enabled = Boolean.TRUE.equals(enabled);
    this.permissions = permissions == null ? List.of() : List.copyOf(permissions);
    this.authorities = this.permissions.stream()
        .map(SimpleGrantedAuthority::new)
        .toList();
  }

  @Override
  public Collection<? extends GrantedAuthority> getAuthorities() {
    return permissions.stream()
        .map(SimpleGrantedAuthority::new)
        .collect(Collectors.toList());
  }

  @Override
  public String getPassword() {
    return this.password;
  }

  @Override
  public String getUsername() {
    return this.username;
  }

  @Override
  public boolean isAccountNonExpired() {
    return true;
  }

  @Override
  public boolean isAccountNonLocked() {
    return true;
  }

  @Override
  public boolean isCredentialsNonExpired() {
    return true;
  }

  @Override
  public boolean isEnabled() {
    return this.enabled;
  }
}
```

#### 4. jwt认证流程
前后端采用jwt来存储认证凭证，后端保留refreshToken来在jwt失效之后进行刷新操作。在jwt中存储用户名以及权限等。
在我们实现自己的自定义的过滤链节点时，需要在认证之后将认证信息放入SecurityContextHolder中，用来给后续节点获取登录信息。

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

  private static final String AUTH_HEADER = "Authorization";
  private static final String BEARER_PREFIX = "Bearer ";

  @Resource private JwtTokenUtil jwtUtil;
  @Resource private CustomUserDetailsService uds;

  /** 白名单路径可通过配置注入 */
  private static final AntPathMatcher matcher = new AntPathMatcher();
  private static final List<String> WHITELIST = List.of("/api/auth/**");

  @Override
  protected boolean shouldNotFilter(HttpServletRequest req) {
    return WHITELIST.stream().anyMatch(p -> matcher.match(p, req.getServletPath()));
  }

  @Override
  protected void doFilterInternal(HttpServletRequest req,
                                  HttpServletResponse res,
                                  FilterChain chain) throws ServletException, IOException {
    String auth = req.getHeader(AUTH_HEADER);
    if (auth == null || !auth.startsWith(BEARER_PREFIX)) {
      chain.doFilter(req, res);
      return;
    }

    String token = auth.substring(BEARER_PREFIX.length());
    try {
      Claims claims = jwtUtil.parseToken(token);          // 只 parse 一次
      String username = claims.getSubject();

      if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
        UserDetails ud = uds.loadUserByUsername(username);

        if (jwtUtil.isValid(claims, ud)) {               // 利用一次解析后的 claims
          JwtAuthenticationToken authentication =
              new JwtAuthenticationToken(ud, claims, ud.getAuthorities());
          authentication.setDetails(
              new WebAuthenticationDetailsSource().buildDetails(req));
          SecurityContextHolder.getContext().setAuthentication(authentication);
        }
      }
    } catch (ExpiredJwtException e) {          // token 过期
      throw new TokenExpiredException("token expire", e);
    } catch (JwtException e) {                 // 伪造 / 签名错误
      throw new InvalidTokenException("token invalid", e);
    }
    chain.doFilter(req, res);
  }
}
```

#### 5. 权限接口
```java
@RestController
public class TestController {
  @GetMapping("/getPermissions")
  @PreAuthorize("hasAnyAuthority('asdasd')")
  public Set<String> getPermissions() {
    throw new RuntimeException("测试一下异常");
  }
}
```