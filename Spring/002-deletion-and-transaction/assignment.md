# 문제

클라이언트 버전 관리 시스템에서 관리하는 스토리지는 다음과 같은 디렉터리 구조로 구성됩니다.  

```text
storage
|-  ProjectA
|   |- Client1
|      |- Version1
|      |- Version2
|
|- ProjectB
   |- Client2
   |  |- Version1
   |- Client3
      |- Version1
      |- Version2
```

다음 코드는 클라이언트 버전 관리 시스템에서 특정 버전을 삭제하는 메서드입니다.  

```kotlin
fun deleteVersion(projectKey: String, clientId: String, version: String): Boolean {
    if (!hasVersion(projectKey, clientId, version)) {
        return false
    }

    val client = projectMap[projectKey]!!.clients[clientId]!!
    val versionRecord = client.findVersionRecord(version)!!

    val path = Paths.get(contentRootPath, versionRecord.fileName.trimStart('/')).normalize()
    if (!client.removeVersionRecord(version)) {
        return false
    }

    val clientDir = Paths.get(contentRootPath, projectKey,clientId).toFile()
    val officialVersionInfo = File(clientDir, "versions.json")
    objectMapper.writerWithDefaultPrettyPrinter().writeValue(officialVersionInfo, client.versions)

    val status = Files.deleteIfExists(path)

    if (!status) {
        return false
    }

    if (client.versions.isEmpty()) {
        removeClientDirectory(projectKey, clientId)
        if (projectMap[projectKey]!!.clients.isEmpty()) {
            removeProjectDirectory(projectKey)
        }
        return true
    }

    return true
}
```

이 코드는 다음과 같은 문제가 있습니다.

## 1. Check-Then-Act Race Condition
- Thread A: hasVersion() → true ✓
- Thread B: hasVersion() → true ✓ (거의 동시)
- Thread A: removeVersionRecord() → 성공
- Thread B: removeVersionRecord() → 실패 (이미 없음)
- Thread A: Files.deleteIfExists(path) → 성공
- Thread B: Files.deleteIfExists(path) → false 반환 (이미 삭제됨)  

## 2. 메모리 ↔ 파일 시스템 불일치
현재 순서:  
a) client.removeVersionRecord() → 메모리에서 제거  
b) versions.json 저장 → 디스크에 기록  
c) Files.deleteIfExists() → 실제 zip 파일 삭제  

문제:
- (b)와 (c) 사이에 크래시 발생 시: versions.json에는 없는데 zip 파일은 남아있음 (고아 파일)
- (c)가 실패해도 (b)는 이미 완료됨: 메타데이터는 삭제됐는데 파일은 남아있음

## 3. versions.json 동시 쓰기
- 두 스레드가 동시에 다른 버전을 삭제하면 versions.json 손상 가능

**동시 삭제를 방지하고, 트랜잭션 순서를 개선하세요.**  

[풀이](solution.md)