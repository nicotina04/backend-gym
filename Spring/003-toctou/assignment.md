# 문제

다음 메서드는 파일 관리 시스템에 `version` 태그에 해당하는 파일을 업로드하는 코드입니다.

```kotlin
fun uploadClient(projectKey: String, clientId: String, file: MultipartFile, version: String, entryFile: String, description: String) {
    val projectDir = Paths.get(contentRootPath, projectKey).toFile()
    val clientDir = Paths.get(contentRootPath, projectKey, clientId).toFile()

    var projectDirCreated = false
    var clientDirCreated = false
    var tempFile: File? = null
    var finalZip: File? = null

    try {
        if (!projectDir.exists()) {
            projectDir.mkdirs()
            projectDirCreated = true
        }
        if (!clientDir.exists()) {
            clientDir.mkdirs()
            clientDirCreated = true
        }

        tempFile = File.createTempFile("upload-", ".zip", clientDir)
        file.transferTo(tempFile)

        // Single Top Level Directory를 unwrap
        val unwrappedZip = File.createTempFile("unwrapped-", ".zip", clientDir)
        unwrapAndRepackZip(tempFile, unwrappedZip)
        tempFile.delete()
        tempFile = unwrappedZip

        // Use version and entryFile from parameters instead of manifest.json
        val versionName = version

        // Create a new zip with launch-settings.json added
        val tempZipWithSettings = File.createTempFile("with-settings-", ".zip", clientDir)
        addLaunchSettingsToZip(tempFile, tempZipWithSettings, version, entryFile)

        // Delete original temp file and use the new one
        tempFile.delete()
        tempFile = tempZipWithSettings

        finalZip = File(clientDir, "$versionName.zip")
        if (finalZip.exists()) {
            throw IllegalStateException("Version $versionName already exists. Cannot overwrite.")
        }
        if (!tempFile.renameTo(finalZip)) {
            throw IOException("Failed to rename temp file to $finalZip")
        }
        tempFile = null // 이름이 변경되었으므로 더 이상 삭제할 필요 없음

        val resultPath = Paths.get("/", projectKey, clientId, finalZip.name).toString().replace('\\', '/')

        val versionRecord = VersionRecord(
            version = versionName,
            description = description,
            fileName = resultPath,
            updatedAt = OffsetDateTime.now()
        )

        val project = projectMap.getOrPut(projectKey) {
            Project(name = projectKey, path = projectDir.absolutePath, clients = hashMapOf())
        }
        val client = project.clients.getOrPut(clientId) {
            Client(name = clientId, path = clientDir.absolutePath, versions = mutableListOf())
        }

        synchronized(client) {
            if (client.hasVersion(versionName)) {
                throw IllegalStateException("$versionName already exists.")
            }
            client.addVersionRecord(versionRecord, true)

            val officialVersionInfo = File(clientDir, "versions.json")
            objectMapper.writerWithDefaultPrettyPrinter()
                .writeValue(officialVersionInfo, client.versions)
        }

        println("Successfully uploaded and indexed content for clientId: $clientId, version: $versionName")

    } catch (e: Exception) {
        finalZip?.delete()
        if (clientDirCreated) {
            clientDir.deleteRecursively()
        } else if (projectDirCreated) {
            if (projectDir.listFiles()?.isEmpty() == true) {
                projectDir.delete()
            }
        }
        throw e
    } finally {
        tempFile?.delete()
    }
}
```

## 1. Replace `getOrPut()`

여기서 `getOrPut()`은 원자적 연산을 하지 않아 다음 부분에서 race condition이 발생할 수 있습니다.

```
val project = projectMap.getOrPut(projectKey) {
    Project(name = projectKey, path = projectDir.absolutePath, clients = hashMapOf())
}
val client = project.clients.getOrPut(clientId) {
    Client(name = clientId, path = clientDir.absolutePath, versions = mutableListOf())
}
```

## 2. TOCTOU(Time-of-Check to Time-of-Use) Race Condition

다음과 같이 파일을 생성하는 코드는 thread unsafe 입니다.

```
finalZip = File(clientDir, "$versionName.zip")
if (finalZip.exists()) {
    throw IllegalStateException("Version $versionName already exists.")
}
if (!tempFile.renameTo(finalZip)) {
    throw IOException("Failed to rename temp file to $finalZip")
}
```

동시에 여러 쓰레드가 동일한 버전 태그를 가진 파일을 업로드할 경우, 예외가 발생하지 않고 덮어씌워질 문제가 있습니다.

**여러 쓰레드가 같은 버전 태그를 업로드할 때 발생하는 Race Condition 문제를 해결하고 예외가 정상적으로 발생하도록 수정하세요.**

[풀이](solution.md)