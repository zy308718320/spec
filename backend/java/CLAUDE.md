# CLAUDE.md - Spring Boot 后端开发系统规则

> **技术栈**: Java 17+ / Spring Boot 3.x / Spring Data JPA / MySQL / Maven

---

## 🎯 常用命令

```bash
./mvnw spring-boot:run          # 启动开发服务器
./mvnw clean package            # 打包
./mvnw test                     # 运行测试
./mvnw clean package -DskipTests # 跳过测试打包
java -jar target/*.jar          # 运行生产包
```

---

## 📁 项目结构

```
src/main/java/com/example/project/
├── Application.java              # 启动类
├── config/                       # 配置类
│   ├── SecurityConfig.java
│   ├── WebConfig.java
│   └── SwaggerConfig.java
├── controller/                   # 控制器层
├── service/                      # 业务逻辑层
│   └── impl/
├── repository/                   # 数据访问层
├── entity/                       # 实体类
├── dto/                          # 数据传输对象
│   ├── request/
│   └── response/
├── mapper/                       # 对象映射（MapStruct）
├── exception/                    # 异常处理
├── security/                     # 安全相关
├── util/                         # 工具类
└── constant/                     # 常量定义

src/main/resources/
├── application.yml               # 主配置
├── application-dev.yml           # 开发环境
├── application-prod.yml          # 生产环境
└── db/migration/                 # Flyway 迁移脚本
```

---

## 🏗️ 分层架构

### Controller（控制器）
```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Tag(name = "用户管理")
public class UserController {

    private final UserService userService;

    @GetMapping
    @Operation(summary = "分页查询用户")
    public Result<PageResult<UserResponse>> list(@Valid UserQueryRequest request) {
        return Result.success(userService.list(request));
    }

    @GetMapping("/{id}")
    @Operation(summary = "根据ID查询用户")
    public Result<UserResponse> getById(@PathVariable Long id) {
        return Result.success(userService.getById(id));
    }

    @PostMapping
    @Operation(summary = "创建用户")
    public Result<UserResponse> create(@Valid @RequestBody CreateUserRequest request) {
        return Result.success(userService.create(request));
    }

    @PutMapping("/{id}")
    @Operation(summary = "更新用户")
    public Result<UserResponse> update(@PathVariable Long id, 
                                        @Valid @RequestBody UpdateUserRequest request) {
        return Result.success(userService.update(id, request));
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "删除用户")
    public Result<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return Result.success();
    }
}
```

### Service（业务层）
```java
public interface UserService {
    PageResult<UserResponse> list(UserQueryRequest request);
    UserResponse getById(Long id);
    UserResponse create(CreateUserRequest request);
    UserResponse update(Long id, UpdateUserRequest request);
    void delete(Long id);
}

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;
    private final UserMapper userMapper;
    private final PasswordEncoder passwordEncoder;

    @Override
    public PageResult<UserResponse> list(UserQueryRequest request) {
        Pageable pageable = PageRequest.of(request.getPage() - 1, request.getSize(),
                Sort.by(Sort.Direction.DESC, "createdAt"));
        
        Page<User> page = userRepository.findByCondition(request.getKeyword(), pageable);
        List<UserResponse> list = userMapper.toResponseList(page.getContent());
        
        return PageResult.of(list, page.getTotalElements(), request.getPage(), request.getSize());
    }

    @Override
    public UserResponse getById(Long id) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));
        return userMapper.toResponse(user);
    }

    @Override
    @Transactional
    public UserResponse create(CreateUserRequest request) {
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new BusinessException(ErrorCode.EMAIL_EXISTS);
        }
        
        User user = userMapper.toEntity(request);
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        user = userRepository.save(user);
        
        return userMapper.toResponse(user);
    }

    @Override
    @Transactional
    public UserResponse update(Long id, UpdateUserRequest request) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));
        
        userMapper.updateEntity(request, user);
        user = userRepository.save(user);
        
        return userMapper.toResponse(user);
    }

    @Override
    @Transactional
    public void delete(Long id) {
        if (!userRepository.existsById(id)) {
            throw new BusinessException(ErrorCode.USER_NOT_FOUND);
        }
        userRepository.deleteById(id);
    }
}
```

### Repository（数据访问层）
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);

    boolean existsByEmail(String email);

    @Query("SELECT u FROM User u WHERE " +
           "(:keyword IS NULL OR u.name LIKE %:keyword% OR u.email LIKE %:keyword%)")
    Page<User> findByCondition(@Param("keyword") String keyword, Pageable pageable);
}
```

---

## 📦 实体与 DTO

### Entity（实体）
```java
@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@EntityListeners(AuditingEntityListener.class)
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 50)
    private String name;

    @Column(nullable = false, unique = true, length = 100)
    private String email;

    @Column(nullable = false)
    @JsonIgnore
    private String password;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private UserRole role = UserRole.USER;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private UserStatus status = UserStatus.ACTIVE;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

public enum UserRole {
    ADMIN, USER
}

public enum UserStatus {
    ACTIVE, INACTIVE, BANNED
}
```

### DTO（数据传输对象）
```java
// Request DTO
@Data
public class CreateUserRequest {
    @NotBlank(message = "姓名不能为空")
    @Size(min = 2, max = 50, message = "姓名长度2-50个字符")
    private String name;

    @NotBlank(message = "邮箱不能为空")
    @Email(message = "邮箱格式不正确")
    private String email;

    @NotBlank(message = "密码不能为空")
    @Size(min = 8, max = 20, message = "密码长度8-20个字符")
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*\\d).+$", message = "密码需包含大写字母和数字")
    private String password;
}

@Data
public class UpdateUserRequest {
    @Size(min = 2, max = 50, message = "姓名长度2-50个字符")
    private String name;

    private UserStatus status;
}

@Data
public class UserQueryRequest extends PageRequest {
    private String keyword;
}

// Response DTO
@Data
@Builder
public class UserResponse {
    private Long id;
    private String name;
    private String email;
    private UserRole role;
    private UserStatus status;
    private LocalDateTime createdAt;
}
```

### Mapper（对象映射）
```java
@Mapper(componentModel = "spring")
public interface UserMapper {

    UserResponse toResponse(User user);

    List<UserResponse> toResponseList(List<User> users);

    User toEntity(CreateUserRequest request);

    void updateEntity(UpdateUserRequest request, @MappingTarget User user);
}
```

---

## 📡 统一响应格式

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Result<T> {
    private int code;
    private String message;
    private T data;
    private long timestamp = System.currentTimeMillis();

    public static <T> Result<T> success() {
        return new Result<>(200, "success", null, System.currentTimeMillis());
    }

    public static <T> Result<T> success(T data) {
        return new Result<>(200, "success", data, System.currentTimeMillis());
    }

    public static <T> Result<T> error(int code, String message) {
        return new Result<>(code, message, null, System.currentTimeMillis());
    }

    public static <T> Result<T> error(ErrorCode errorCode) {
        return new Result<>(errorCode.getCode(), errorCode.getMessage(), null, System.currentTimeMillis());
    }
}

@Data
@Builder
public class PageResult<T> {
    private List<T> list;
    private long total;
    private int page;
    private int size;
    private int totalPages;

    public static <T> PageResult<T> of(List<T> list, long total, int page, int size) {
        return PageResult.<T>builder()
                .list(list)
                .total(total)
                .page(page)
                .size(size)
                .totalPages((int) Math.ceil((double) total / size))
                .build();
    }
}
```

---

## 🚨 异常处理

### 错误码枚举
```java
@Getter
@AllArgsConstructor
public enum ErrorCode {
    // 通用错误 1xxx
    SUCCESS(200, "操作成功"),
    BAD_REQUEST(400, "请求参数错误"),
    UNAUTHORIZED(401, "未认证"),
    FORBIDDEN(403, "无权限"),
    NOT_FOUND(404, "资源不存在"),
    INTERNAL_ERROR(500, "服务器内部错误"),

    // 用户相关 2xxx
    USER_NOT_FOUND(2001, "用户不存在"),
    EMAIL_EXISTS(2002, "邮箱已被注册"),
    INVALID_CREDENTIALS(2003, "用户名或密码错误"),
    USER_DISABLED(2004, "用户已被禁用"),

    // 认证相关 3xxx
    TOKEN_EXPIRED(3001, "令牌已过期"),
    TOKEN_INVALID(3002, "无效的令牌");

    private final int code;
    private final String message;
}
```

### 自定义异常
```java
@Getter
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }

    public BusinessException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
}
```

### 全局异常处理
```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException e) {
        log.warn("业务异常: {}", e.getMessage());
        return Result.error(e.getErrorCode().getCode(), e.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(error -> error.getField() + ": " + error.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return Result.error(400, message);
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public Result<Void> handleConstraintViolation(ConstraintViolationException e) {
        String message = e.getConstraintViolations().stream()
                .map(ConstraintViolation::getMessage)
                .collect(Collectors.joining(", "));
        return Result.error(400, message);
    }

    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.error(500, "服务器内部错误");
    }
}
```

---

## 🔐 安全配置

### Security 配置
```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .csrf(AbstractHttpConfigurer::disable)
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/v1/auth/**", "/swagger-ui/**", "/v3/api-docs/**").permitAll()
                        .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated()
                )
                .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
                .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### JWT 工具类
```java
@Component
public class JwtUtils {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration}")
    private long expiration;

    public String generateToken(User user) {
        return Jwts.builder()
                .setSubject(user.getId().toString())
                .claim("email", user.getEmail())
                .claim("role", user.getRole().name())
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(getSigningKey(), SignatureAlgorithm.HS256)
                .compact();
    }

    public Claims parseToken(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
    }

    public boolean validateToken(String token) {
        try {
            parseToken(token);
            return true;
        } catch (JwtException e) {
            return false;
        }
    }

    private Key getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }
}
```

---

## ⚙️ 配置文件

### application.yml
```yaml
spring:
  profiles:
    active: dev
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=Asia/Shanghai
    username: ${DB_USERNAME:root}
    password: ${DB_PASSWORD:root}
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        format_sql: true
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
    default-property-inclusion: non_null

jwt:
  secret: ${JWT_SECRET:your-256-bit-secret-key-here-at-least-32-chars}
  expiration: 604800000  # 7天

logging:
  level:
    com.example: debug
    org.springframework.security: debug
```

### application-prod.yml
```yaml
spring:
  jpa:
    show-sql: false
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5

logging:
  level:
    com.example: info
    org.springframework: warn
```

---

## 🧪 测试规范

### 单元测试
```java
@ExtendWith(MockitoExtension.class)
class UserServiceImplTest {

    @Mock
    private UserRepository userRepository;
    @Mock
    private UserMapper userMapper;
    @Mock
    private PasswordEncoder passwordEncoder;

    @InjectMocks
    private UserServiceImpl userService;

    @Test
    void getById_WhenUserExists_ReturnsUser() {
        User user = User.builder().id(1L).name("Test").build();
        UserResponse response = UserResponse.builder().id(1L).name("Test").build();

        when(userRepository.findById(1L)).thenReturn(Optional.of(user));
        when(userMapper.toResponse(user)).thenReturn(response);

        UserResponse result = userService.getById(1L);

        assertThat(result.getName()).isEqualTo("Test");
    }

    @Test
    void getById_WhenUserNotExists_ThrowsException() {
        when(userRepository.findById(1L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> userService.getById(1L))
                .isInstanceOf(BusinessException.class)
                .hasFieldOrPropertyWithValue("errorCode", ErrorCode.USER_NOT_FOUND);
    }
}
```

### 集成测试
```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;
    @Autowired
    private ObjectMapper objectMapper;

    @Test
    @WithMockUser(roles = "ADMIN")
    void createUser_WithValidData_ReturnsCreated() throws Exception {
        CreateUserRequest request = new CreateUserRequest();
        request.setName("Test");
        request.setEmail("test@example.com");
        request.setPassword("Password123");

        mockMvc.perform(post("/api/v1/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.data.name").value("Test"));
    }
}
```

---

## 🎯 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 类名 | PascalCase | `UserService` |
| 方法名 | camelCase | `findByEmail` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| 包名 | 全小写 | `com.example.user` |
| 表名 | snake_case 复数 | `users`, `user_roles` |
| 字段名 | snake_case | `created_at` |
| URL | kebab-case | `/api/v1/user-profiles` |

---

## 📋 代码规范

```
✅ 使用 Lombok 减少样板代码
✅ 使用 @RequiredArgsConstructor 构造器注入
✅ Service 层使用接口 + 实现类
✅ 使用 @Transactional 管理事务（只读方法加 readOnly=true）
✅ 使用 MapStruct 进行对象映射
✅ 使用 @Valid 验证请求参数
✅ 统一使用 Result 包装响应
✅ 异常使用 ErrorCode 枚举
```

---

## 🔒 安全规范

```
🔴 密码使用 BCrypt 加密
🔴 敏感配置使用环境变量
🔴 JWT 密钥至少 256 位
🔴 SQL 使用参数化查询
🔴 接口做权限校验
🔴 敏感字段加 @JsonIgnore
🔴 生产环境关闭 Swagger
```

---

## 📋 提交前检查

```
□ 代码编译通过
□ 单元测试通过
□ 接口有参数校验
□ 异常处理完整
□ 敏感操作有权限验证
□ 无硬编码敏感信息
□ 日志级别合适
```

---
