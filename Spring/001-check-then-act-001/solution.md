# 001 - Check-Then-Act 001 해설

## Prerequisite
JVM 계열의 `ConcurrentHashMap`은 다음과 같은 atomic method를 지원합니다.  
- putIfAbsent(K, V): K가 없으면 V를 추가
- computeIfAbsent(K, Function): 없을 때만 계산해서 추가
- computeIfPresent(K, BiFuntion): 있을 때만 계산해서 업데이트
- compute(K, BiFunction): 항상 계산해서 업데이트

## Solution
`ConcurrentHashMap`의 연산이 atomic임을 보장하기 위해서는 Prerequisite에 언급된 atomic method를 활용해야 합니다.  

`createProject()`에서 race condition 문제를 해결하기 위해서는 `containsKey`를 쓰지 않고 `computeIfAbsent`를 사용하여 `Project` 생성의 원자성을 보장하고 디렉터리 생성을 콜백으로 처리하여 프로젝트가 덮어씌워지는 문제를 방지할 수 있습니다.

```kotlin
fun createProject(projectKey: String): Boolean {
    var isNewProject = false

    projectMap.computeIfAbsent(projectKey) {
        isNewProject = true
        val projectDir = File(contentRootPath, projectKey)
        if (!projectDir.exists()) {
            if (!projectDir.mkdirs()) {
                throw IOException("Failed to create directory ${projectDir.path}")
            }
        }

        Project(name = projectKey, path = projectDir.absolutePath, clients = mutableMapOf())
    }

    return isNewProject
}
```

## Note
- computeIfAbsent의 람다는 한 번만 실행됨이 보장됨
- 디렉터리 생성 실패 시 예외를 던져서 Map에 추가되지 않도록 함
- 람다 내부에서는 다른 Map 연산을 호출하면 안 됨 (deadlock 위험)
