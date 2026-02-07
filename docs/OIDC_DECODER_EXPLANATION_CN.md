# OIDC ID Token Decoder 代码变更详解

## 问题背景

为什么代码从简单的几行变成了复杂的一大段？每一行都在做什么？

## 旧代码（简单版本）

```java
var oidcIdTokenDecodeFactory = new ReactiveOidcIdTokenDecoderFactory();
oidcIdTokenDecodeFactory.setJwsAlgorithmResolver(clientRegistration -> {
    var configurationMetadata = clientRegistration.getProviderDetails()
        .getConfigurationMetadata();
    try {
        var supportedJwsAlgorithms = JSONObjectUtils.getStringList(
            new JSONObject(configurationMetadata),
            "id_token_signing_alg_values_supported"
        );
        // we choose the first one as JWS algorithm
        if (!supportedJwsAlgorithms.isEmpty()) {
            var jwsAlgorithm = supportedJwsAlgorithms.get(0);
            return SignatureAlgorithm.from(jwsAlgorithm);
        }
    } catch (ParseException e) {
        // ignore the error.
    }
    // default algorithm
    return SignatureAlgorithm.RS256;
});
oidcAuthManager.setJwtDecoderFactory(oidcIdTokenDecodeFactory);
```

### 旧代码逐行解释

1. **`var oidcIdTokenDecodeFactory = new ReactiveOidcIdTokenDecoderFactory();`**
   - 创建 Spring Security 提供的标准 OIDC ID Token 解码器工厂
   - 这个工厂负责创建用于验证 OIDC ID Token 的解码器

2. **`oidcIdTokenDecodeFactory.setJwsAlgorithmResolver(clientRegistration -> { ... });`**
   - 设置一个算法解析器（lambda 函数）
   - 这个解析器告诉工厂应该使用哪种签名算法（如 RS256, RS384 等）

3. **`var configurationMetadata = clientRegistration.getProviderDetails().getConfigurationMetadata();`**
   - 从客户端注册信息中获取配置元数据
   - 这个元数据可能包含 OIDC 提供商支持的算法列表

4. **`var supportedJwsAlgorithms = JSONObjectUtils.getStringList(...)`**
   - 尝试从元数据中读取 `id_token_signing_alg_values_supported` 字段
   - 这个字段列出了提供商支持的所有签名算法

5. **`if (!supportedJwsAlgorithms.isEmpty()) { return SignatureAlgorithm.from(jwsAlgorithm); }`**
   - 如果找到了支持的算法列表，使用第一个算法
   - 例如：如果列表是 `["RS256", "RS384"]`，就使用 `RS256`

6. **`catch (ParseException e) { // ignore }`**
   - 如果解析失败（比如没有配置元数据），忽略错误

7. **`return SignatureAlgorithm.RS256;`**
   - 默认使用 RS256 算法（最常用的 OIDC 算法）

### 旧代码的问题

**无法设置自定义 WebClient！**

虽然可以设置算法解析器，但是 `ReactiveOidcIdTokenDecoderFactory` 内部创建 JWT 解码器时，使用的是**默认的 WebClient**，这个 WebClient **不会使用我们配置的代理**。

当 OIDC 提供商需要通过代理访问时（比如 LINUX DO），JWKS（JSON Web Key Set）获取会失败，因为：
- Token 交换请求 ✅ 使用代理（我们配置了）
- 用户信息请求 ✅ 使用代理（我们配置了）
- **JWKS 获取请求 ❌ 不使用代理**（无法配置）

---

## 新代码（完整版本）

```java
// Create custom OIDC ID token decoder factory with proxy-enabled WebClient
var oidcIdTokenDecodeFactory = createOidcIdTokenDecoderFactory(webClient);
oidcAuthManager.setJwtDecoderFactory(oidcIdTokenDecodeFactory);
```

调用的方法：

```java
private ReactiveJwtDecoderFactory<ClientRegistration> createOidcIdTokenDecoderFactory(
    WebClient webClient) {
    
    return new ReactiveJwtDecoderFactory<ClientRegistration>() {
        @Override
        public ReactiveJwtDecoder createDecoder(ClientRegistration clientRegistration) {
            // Determine the JWS algorithm from provider metadata
            SignatureAlgorithm jwsAlgorithm = resolveJwsAlgorithm(clientRegistration);
            
            String jwkSetUri = clientRegistration.getProviderDetails().getJwkSetUri();
            if (jwkSetUri == null) {
                OAuth2Error oauth2Error = new OAuth2Error(
                    "missing_signature_verifier",
                    "Failed to find a Signature Verifier for Client Registration: '"
                        + clientRegistration.getRegistrationId()
                        + "'. Check to ensure you have configured the JWK Set URI.",
                    null
                );
                throw new OAuth2AuthenticationException(oauth2Error, oauth2Error.toString());
            }
            
            // Build decoder with custom WebClient for JWKS retrieval
            NimbusReactiveJwtDecoder decoder = NimbusReactiveJwtDecoder
                .withJwkSetUri(jwkSetUri)
                .jwsAlgorithm(jwsAlgorithm)
                .webClient(webClient)  // 🔑 关键：注入我们的 WebClient（带代理配置）
                .build();
            
            // Apply default OIDC claim type converters
            decoder.setClaimSetConverter(
                MappedJwtClaimSetConverter.withDefaults(
                    ReactiveOidcIdTokenDecoderFactory.createDefaultClaimTypeConverters()
                )
            );
            
            return decoder;
        }
        
        private SignatureAlgorithm resolveJwsAlgorithm(ClientRegistration clientRegistration) {
            var configurationMetadata = clientRegistration.getProviderDetails()
                .getConfigurationMetadata();
            try {
                var supportedJwsAlgorithms = JSONObjectUtils.getStringList(
                    new JSONObject(configurationMetadata),
                    "id_token_signing_alg_values_supported"
                );
                // we choose the first one as JWS algorithm
                if (!supportedJwsAlgorithms.isEmpty()) {
                    var jwsAlgorithm = supportedJwsAlgorithms.get(0);
                    return SignatureAlgorithm.from(jwsAlgorithm);
                }
            } catch (ParseException e) {
                // Ignore the error if metadata is missing or malformed and fall back to default RS256 algorithm
            }
            // default algorithm
            return SignatureAlgorithm.RS256;
        }
    };
}
```

### 新代码逐行解释

#### 主方法签名
```java
private ReactiveJwtDecoderFactory<ClientRegistration> createOidcIdTokenDecoderFactory(
    WebClient webClient) {
```
- **参数 `webClient`**：这是我们配置好的、包含代理设置的 WebClient
- **返回值**：返回一个 JWT 解码器工厂

#### 创建匿名内部类
```java
return new ReactiveJwtDecoderFactory<ClientRegistration>() {
```
- 创建 `ReactiveJwtDecoderFactory` 接口的匿名实现类
- 这个接口只有一个方法：`createDecoder(ClientRegistration)`

#### createDecoder 方法

**第1步：解析算法**
```java
SignatureAlgorithm jwsAlgorithm = resolveJwsAlgorithm(clientRegistration);
```
- 调用内部方法获取应该使用的签名算法
- 这部分逻辑和旧代码一样，只是提取成了独立方法

**第2步：检查 JWKS URI**
```java
String jwkSetUri = clientRegistration.getProviderDetails().getJwkSetUri();
if (jwkSetUri == null) {
    OAuth2Error oauth2Error = new OAuth2Error(
        "missing_signature_verifier",
        "Failed to find a Signature Verifier for Client Registration: '"
            + clientRegistration.getRegistrationId()
            + "'. Check to ensure you have configured the JWK Set URI.",
        null
    );
    throw new OAuth2AuthenticationException(oauth2Error, oauth2Error.toString());
}
```
- 获取 JWKS URI（OIDC 提供商的公钥地址）
- 如果没有配置，抛出清晰的错误信息
- **这是旧代码没有的**：提供了更好的错误处理

**第3步：构建解码器（核心！）**
```java
NimbusReactiveJwtDecoder decoder = NimbusReactiveJwtDecoder
    .withJwkSetUri(jwkSetUri)           // 设置 JWKS URI
    .jwsAlgorithm(jwsAlgorithm)         // 设置签名算法
    .webClient(webClient)               // 🔑 关键：使用我们的 WebClient（带代理）
    .build();
```
- **`.withJwkSetUri(jwkSetUri)`**：告诉解码器从哪里获取公钥
- **`.jwsAlgorithm(jwsAlgorithm)`**：告诉解码器使用什么算法验证签名
- **`.webClient(webClient)`**：**这是最重要的！** 使用我们传入的、配置了代理的 WebClient
  - 这样，JWKS 获取请求也会通过代理
  - 这是旧代码无法做到的
- **`.build()`**：构建最终的解码器

**第4步：设置声明转换器**
```java
decoder.setClaimSetConverter(
    MappedJwtClaimSetConverter.withDefaults(
        ReactiveOidcIdTokenDecoderFactory.createDefaultClaimTypeConverters()
    )
);
```
- 设置 OIDC 标准的声明类型转换器
- 例如：将 `exp`（过期时间）从数字转换为日期对象
- `ReactiveOidcIdTokenDecoderFactory.createDefaultClaimTypeConverters()` 提供了 OIDC 标准转换规则
- **这确保了行为和 Spring Security 默认行为一致**

**第5步：返回解码器**
```java
return decoder;
```

#### resolveJwsAlgorithm 方法

```java
private SignatureAlgorithm resolveJwsAlgorithm(ClientRegistration clientRegistration) {
    var configurationMetadata = clientRegistration.getProviderDetails()
        .getConfigurationMetadata();
    try {
        var supportedJwsAlgorithms = JSONObjectUtils.getStringList(
            new JSONObject(configurationMetadata),
            "id_token_signing_alg_values_supported"
        );
        // we choose the first one as JWS algorithm
        if (!supportedJwsAlgorithms.isEmpty()) {
            var jwsAlgorithm = supportedJwsAlgorithms.get(0);
            return SignatureAlgorithm.from(jwsAlgorithm);
        }
    } catch (ParseException e) {
        // Ignore the error if metadata is missing or malformed and fall back to default RS256 algorithm
    }
    // default algorithm
    return SignatureAlgorithm.RS256;
}
```

这部分逻辑**和旧代码完全一样**，只是提取成了独立方法：
1. 获取配置元数据
2. 尝试读取支持的算法列表
3. 使用第一个算法
4. 如果失败，默认使用 RS256

---

## 对比总结

| 特性 | 旧代码 | 新代码 |
|------|--------|--------|
| **代理支持** | ❌ JWKS 获取不使用代理 | ✅ JWKS 获取使用代理 |
| **算法解析** | ✅ 支持从元数据读取 | ✅ 支持从元数据读取（逻辑相同） |
| **错误处理** | ⚠️ 简单 | ✅ 详细的错误信息 |
| **声明转换** | ✅ 使用默认转换器 | ✅ 明确设置 OIDC 转换器 |
| **代码复杂度** | 简单（~20 行） | 复杂（~60 行） |

---

## 为什么要这样改？

### 核心原因：无法注入自定义 WebClient

Spring Security 的 `ReactiveOidcIdTokenDecoderFactory` 类是 `final` 的：
```java
public final class ReactiveOidcIdTokenDecoderFactory { ... }
```

这意味着：
1. ❌ 不能继承这个类
2. ❌ 不能覆盖它的方法
3. ❌ 它没有提供 `setWebClient()` 方法

所以，旧代码虽然可以设置算法解析器，但是**无法控制内部使用的 WebClient**。

### 解决方案：实现接口而不是使用现成的类

新代码通过实现 `ReactiveJwtDecoderFactory<ClientRegistration>` 接口：
1. ✅ 完全控制解码器的创建过程
2. ✅ 可以注入自定义的 WebClient
3. ✅ 保持了算法解析的灵活性
4. ✅ 使用 OIDC 标准的声明转换器

---

## 实际影响

假设你的服务器需要通过代理访问 LINUX DO：

### 旧代码的行为
```
1. Token 交换  → https://connect.linux.do/oauth2/token  ✅ 通过代理
2. 用户信息    → https://connect.linux.do/api/user      ✅ 通过代理
3. JWKS 获取   → https://connect.linux.do/.well-known/jwks.json  ❌ 直连失败！
```

### 新代码的行为
```
1. Token 交换  → https://connect.linux.do/oauth2/token  ✅ 通过代理
2. 用户信息    → https://connect.linux.do/api/user      ✅ 通过代理
3. JWKS 获取   → https://connect.linux.do/.well-known/jwks.json  ✅ 通过代理
```

---

## 总结

虽然新代码看起来更复杂，但是：

1. **功能完整性**：三种请求都使用代理，保证一致性
2. **必要性**：这是绕过 Spring Security 限制的唯一方法
3. **逻辑清晰**：
   - `resolveJwsAlgorithm()` 方法：处理算法解析（和旧代码相同）
   - `createDecoder()` 方法：构建解码器并注入 WebClient（新增功能）
4. **维护性**：代码结构更清晰，每个方法职责单一

**简单来说**：为了让 OIDC 的 JWKS 获取也能使用代理配置，我们必须手动实现 JWT 解码器工厂，而不能使用 Spring Security 的默认实现。虽然代码变长了，但是解决了一个关键问题。
