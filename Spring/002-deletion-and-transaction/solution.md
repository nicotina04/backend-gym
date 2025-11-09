# 002 - deletion-and-transaction 해설

## Prerequisite

JVM에서는 제공하는 `synchronized(item)` 키워드가 있습니다. `synchronized(item)`는 `item`을 모니터로 내부의 코드 블럭을 Lock으로 묶어 동기화를 제공하게 됩니다. 

### 예시
```kotlin
synchronized(item) {
    // Mutual Exclusion Start
    item.foo()
    item.bar()
    // Mutual Exclusion End
}
```

## Solution

Thread-unsafe한 로직을 배제하고 `synchronized`를 사용하여 삭제 연산에서 발생하는 레이스 컨디션을 방지합니다.  
추가로 파일을 삭제할 때 파일이 없으면 예외를 던지도록 하여 안정성을 보강합니다.  
최종적으로 `synchronized`를 적절히 사용하여 모든 문제를 해결할 수 있습니다.

샘플 코드
```kotlin
fun deleteVersion(projectKey: String, clientId: String, version: String): Boolean {
    val client = projectMap[projectKey]?.getClient(clientId) ?: return false

    synchronized(client) {
        val versionRecord = client.findVersionRecord(version) ?: return false

        val path = Paths.get(contentRootPath, versionRecord.fileName.trimStart('/')).normalize()
        val clientDir = Paths.get(contentRootPath, projectKey,clientId).toFile()

        if (Files.notExists(path)) {
            throw IllegalStateException("File $path does not exists.")
        }

        Files.delete(path)

        if (!client.removeVersionRecord(version)) {
            return false
        }

        val officialVersionInfo = File(clientDir, "versions.json")
        objectMapper.writerWithDefaultPrettyPrinter().writeValue(officialVersionInfo, client.versions)

        if (client.versions.isEmpty()) {
            removeClientDirectory(projectKey, clientId)
            if (projectMap[projectKey]!!.clients.isEmpty()) {
                removeProjectDirectory(projectKey)
            }
            return true
        }

        return true
    }
}
```
