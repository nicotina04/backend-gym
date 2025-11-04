# 001 - Check-Then-Act 001

## 문제
다음 메서드는 스토리지에 디렉터리를 생성하고 `projectMap`에 등록하는 기능을 하며, Check-Then-Act 패턴으로 인한 race condition이 있습니다.

```kotlin
fun createProject(projectKey: String): Boolean {
    if (projectMap.containsKey(projectKey)) {
        return false
    }

    val projectDir = File(contentRootPath, projectKey)
    if (!projectDir.exists()) {
        if (!projectDir.mkdirs()) {
            throw IOException("Failed to create directory ${projectDir.path}")
        }
    }

    projectMap[projectKey] = Project(name = projectKey, path = projectDir.absolutePath, clients = mutableMapOf<String, Client>())
    return true
}
```

시나리오:
- Thread A: containsKey(projectKey) → false ✓
- Thread B: containsKey(projectKey) → false ✓ (거의 동시)
- Thread A: mkdirs() → 성공
- Thread B: mkdirs() → 성공 (이미 존재해도 true 반환)
- Thread A: projectMap[projectKey] = Project(...) → 성공
- Thread B: projectMap[projectKey] = Project(...) → 덮어쓰기! ❌

결과:
- 두 스레드가 서로 다른 project 객체를 생성
- 나중에 실행된 스레드의 객체로 덮어써짐
- 먼저 생성된 객체에 추가된 Client 정보 손실 가능

**Race Condition이 발생하지 않도록 `createProject()`를 수정하세요.**

[풀이](./solution.md)
