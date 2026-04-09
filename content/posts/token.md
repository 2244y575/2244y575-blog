+++
date = '2026-03-27T21:13:13+08:00'
draft = false
title = '浅谈Token登录方案'
+++

## 一、传统Token方法

### 1. Token的颁发与解析
当下Token常使用 **JWT** 方案，是一种无状态的、自包含的令牌标准。

传统方案设置 `secretKey` 密钥，颁发Token和对应存储信息。Spring Boot项目中需要先引入 `jjwt` 依赖。
<!--more-->

**颁发令牌：**
```java
public String generateToken(String secretKey, Long ttlMillis, Map<String, Object> claims, Long userId) {
        return Jwts.builder()
                .subject(userId.toString())
                .claims(claims)
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + ttlMillis))
                .signWith(getSigningKey(secretKey))
                .compact();
    }
```

**解析令牌：**
```java
public Claims parseToken(String secretKey, String jwt){
        return Jwts.parser()
                .verifyWith(getSigningKey(secretKey))
                .build()
                .parseSignedClaims(jwt)
                .getPayload();
    }
```

### 2. Token的使用
有了令牌的创建解析工具类后，我们需要去使用它。
- **前端**：接收后存放Token，发送请求时携带 `Authorization` 头。
- **后端**：设计过滤器拦截请求，解析Token并放入上下文。

**前端拦截器 (Axios)：**
```typescript
request.interceptors.request.use(
  (config) => {
    const token = useUserStore().getAccessToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
)
```

**后端过滤器 (Spring Security)：**
这里引入了 Spring Security，主要逻辑如下：
```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {

        // 1. 从 Header 获取 JWT
        String header = request.getHeader(jwtProperties.getAccessTokenName());

        if (header == null || !header.startsWith(jwtProperties.getAccessTokenHeaderPrefix())) {
            chain.doFilter(request, response);     // 没 Token 先放过，后面 Security 会报 401
            return;
        }
        String token = header.substring(jwtProperties.getAccessTokenHeaderPrefix().length());

        try {
            // 2. 验签 + 检查过期
            Claims claims = jwtUtils.parseToken(jwtProperties.getAccessSecretKey(), token);
            String username = claims.get(JwtClaimsConstant.USERNAME, String.class);

            // 3. 实时检验用户状态
            UserDetails user = userDetailsService.loadUserByUsername(username);

            // 4. 封装 Authentication 对象
            List<SimpleGrantedAuthority> authorities =
                    user.getAuthorities()
                            .stream()
                            .map(grantedAuthority -> new SimpleGrantedAuthority(grantedAuthority.getAuthority()))
                            .collect(Collectors.toList());

            // 这里存放 userDetails 对象，会自动存放到 context 上下文中
            UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(user, token, authorities);
            authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            SecurityContextHolder.getContext().setAuthentication(authentication); // 放入上下文
            
        }  catch (ExpiredJwtException e) {
            customAuthenticationEntryPoint.commence(request, response, new SecurityTokenExpiredException());
            return;
        } catch (JwtException | IllegalArgumentException e) {
            customAuthenticationEntryPoint.commence(request, response, new SecurityTokenInvalidException());
            return;
        }

        chain.doFilter(request, response);
    }
}
```

### 3. 主动失效Token
JWT 是无状态的，这意味着一旦颁发，服务端很难控制它的失效。为了解决“用户下线”或“封禁”问题，我们通常引入 **Redis**。

常见的方案有两种：

#### 3.1 黑名单机制
- **原理**：Token 正常颁发。当需要失效时，将 Token 存入 Redis。
- **校验**：请求进来时，查询 Redis 中是否存在该 Token。存在则拦截。
- **适用**：大多数场景。

#### 3.2 白名单机制
- **原理**：颁发 Token 后，必须存入 Redis 才有效。
- **校验**：请求进来时，Redis 中必须有该 Token。
- **缺点**：存储数据量大，对 Redis 压力大。

> **注意：** 引入 Redis 虽然解决了失效问题，但在一定程度上破坏了 JWT **无状态**的优势，实际开发中需要根据业务场景进行取舍。

---

## 二、双Token机制

### 1. 单Token的痛点
- **有效期难权衡**：太长不安全，太短影响体验。
- **被盗风险**：一旦泄露，在过期前黑客都能用。

### 2. 双Token机制原理
登录时颁发两个 Token：
- **Access Token (AT)**：用于业务请求，**短效**（如 15 分钟）。
- **Refresh Token (RT)**：用于刷新 Access Token，**长效**（如 7 天）。

当 AT 过期时，前端利用 RT 去后端换取一对新的 Token。这样即使 AT 被盗，有效期也很短；而 RT 不常携带，降低了风险。

### 3. 具体实现
双 Token 基于单 Token 方案改造，后端需增加刷新接口。

**后端颁发双Token逻辑：**
```java
// 获取用户名信息
User user = getOne(new LambdaQueryWrapper<User>()
        .eq(User::getId, userId)
        .select(User::getUsername)
);
Map<String, Object> newClaims = new HashMap<>();
newClaims.put(JwtClaimsConstant.USERNAME, user.getUsername());

// 颁发新的双Token
String accessToken = jwtUtils.generateToken(jwtProperties.getAccessSecretKey(), jwtProperties.getAccessTtl(), newClaims, userId);
String refreshToken = jwtUtils.generateToken(jwtProperties.getRefreshSecretKey(), jwtProperties.getRefreshTtl(), Collections.emptyMap(), userId);

// 过期当前刷新Token (如果是轮换策略)
cacheService.cacheJwt(refreshTokenDTO.getRefreshToken(), userId);

return RefreshTokenVO.builder()
        .accessToken(accessToken)
        .refreshToken(refreshToken)
        .expireTimestamp(jwtUtils.getExpirationTime(jwtProperties.getRefreshSecretKey(), refreshToken).getTime())
        .build();
```

### 4. 前端核心实现
这是最复杂的部分，我们需要处理 **401 拦截** 和 **请求队列**。

**核心逻辑：**
1. 接口报 401。
2. 判断是否正在刷新 Token。
    - 是：将当前请求加入队列，等待刷新完成。
    - 否：调用刷新接口，成功后重试原请求，并唤醒队列。

```typescript
// 定义扩展的请求配置类型，以支持 _retry 属性
interface CustomInternalConfig extends InternalAxiosRequestConfig {
  _retry?: boolean;
}

// 定义队列项的类型
type FailedQueueItem = {
  resolve: (token: string) => void;
  reject: (err: any) => void;
};

let isRefreshing = false;
let failedQueue: FailedQueueItem[] = [];

// 处理队列函数
const processQueue = (error: AxiosError | null, token: string | null = null) => {
  failedQueue.forEach((prom) => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token!);
    }
  });
  failedQueue = [];
};

// 响应拦截器
request.interceptors.response.use(
  (response) => {
    return response.data;
  },
  async (error) => {
    const originalRequest = error.config as CustomInternalConfig;
    const userStore = useUserStore();

    // 1. 基础校验：如果没有 response (网络断开等)，直接报错
    if (!error.response) {
      ElMessage.error("网络错误，请稍后再试");
      return Promise.reject(error);
    }

    const { status, data } = error.response;
    const msg = data.msg;

    // 2. 如果已经重试过，直接走错误处理
    if (originalRequest._retry) {
      return Promise.reject(error);
    }

    // 3. 处理 401 未授权 (核心逻辑)
    if (status === 401) {
      // 排除刷新接口本身，防止死循环
      if (originalRequest.url?.includes('auth/refresh')) {
        ElMessage.warning("登录已过期，请重新登录");
        userStore.reset();
        router.replace({ name: "Login" });
        return Promise.reject(error);
      }

      // 如果正在刷新中，将当前请求加入队列
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        })
          .then((token) => {
            // 队列唤醒：拿到新 Token，重试原请求
            if (originalRequest.headers) {
              originalRequest.headers.Authorization = `Bearer  $ {token}`;
            }
            return request(originalRequest);
          })
          .catch((err) => {
            return Promise.reject(err);
          });
      }

      // 标记开始刷新
      isRefreshing = true;
      originalRequest._retry = true;

      try {
        // 调用刷新api
        const res = await refreshToken().runAsync();
        // 更新 Store
        const newAccessToken = res.data.accessToken;
        userStore.setAccessToken(newAccessToken);
        userStore.setRefreshToken(res.data.refreshToken);
        userStore.setExpireTimestamp(res.data.expireTimestamp);

        // 唤醒队列
        processQueue(null, newAccessToken);

        // 重试原请求
        if (originalRequest.headers) {
          originalRequest.headers.Authorization = `Bearer  $ {newAccessToken}`;
        }
        return request(originalRequest);

      } catch (refreshError) {
        // 刷新失败 (RT 过期或被顶替)
        processQueue(refreshError as AxiosError, null);
        userStore.reset();
        ElMessage.warning("登录已过期，请重新登录");
        router.replace({ name: "Login" });
        return Promise.reject(refreshError);
      } finally {
        // 无论成功失败，重置锁
        isRefreshing = false;
      }
    }

    // 4. 处理其他状态码 (403, 500 等)
    if (status === 403) {
      console.warn(data);
      ElMessage.warning(msg);
    } else if (status === 500) {
      console.error(data);
      ElMessage.error(msg);
    } else {
      // 其他错误
      console.warn(data);
      ElMessage.warning(msg);
    }

    return Promise.reject(error);
  }
);
```
对于401状态码，我们调用刷新令牌的api进行重试，并设置期间的异常请求到队列中。

如果成功，则添加对应双token并携带新token进行重试

如果失败，则在拦截器识别对应请求并阻止接下来的逻辑，代码回到调用刷新api的位置，catch异常并将队列中的所有异常请求抛出。
### 4. 失效令牌
对于双 Token，**Refresh Token**的失效尤为重要。

- Access Token：因为短效，通常不需要额外处理，等它自动过期即可。
- Refresh Token：建议使用 白名单 机制。
    - 以 userId 为 Key，存储最新的 Refresh Token。
    - 刷新时校验 Redis 中的 Token 是否与传入的一致。

与单token不同，双token我们常用白名单来实现。

如果使用黑名单，平时的请求只携带了at，那就代表对于退出登录等业务用户还需要额外提供rt，相应的增大了风险复杂程度。

使用白名单，在颁发新的rt时存入（替换）redis里的对应数据，在刷新时先校验rt是否有效并对应传入的userId。

这样一旦用户修改密码或主动登出，删除 Redis 中的 Key，旧 Token 即刻失效。