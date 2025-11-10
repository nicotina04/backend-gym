# 003 - TOCTOU 해설

## Prerequisite

파일 시스템에서 어떤 파일 `a`가 조작 중임을 알릴 때는 `.a.lock`과 같이 lock 파일을 생성하여 `a`가 어떤 쓰레드에 의해 조작 중임을 인지할 수 있게 할 수 있습니다.  

또한 `java.io.File.createNewFile()`은 리턴형이 `Boolean`이며 atomic한 연산입니다.

## Solution

먼저 `getOrPut`사용 문제를 해결합시다. 이는 [001-check-then-act-001](/Spring/001-check-then-act-001/solution.md) 에도 언급되었듯이 `computeIfAbsent`를 사용한 atomic 연산으로 바꿔주면 됩니다.

```kotlin
val project = projectMap.computeIfAbsent(projectKey) {
    Project(name = projectKey, path = projectDir.absolutePath, clients = hashMapOf())
}
val client = project.clients.computeIfAbsent(clientId) {
    Client(name = clientId, path = clientDir.absolutePath, versions = mutableListOf())
}
```

다음은 파일의 Race Condition 문제를 해결해봅시다. 이 문제를 해결하는 것은 간단합니다. `java.io.File.createNewFile()`을 통해 lock 파일의 생성을 실패하면 이미 다른 쓰레드가 점유하고 있으므로 예외를 던지고, 성공한다면 기존의 하려던 작업을 이어하면 됩니다.  

**작업이 끝난 후 lock 파일을 삭제하는 것을 잊지 마세요.**

### Example

```kotlin
val lockFile = File(dest, ".$version.lock")

try {
    if (!lockFile.createNewFile()) throw IllgalStateException()
    / *
    기존 로직 진행...
    * /
} catch (e: Exception) {
    throw e
} finally {
    lockFile.dlete()
}
```
