# 02. Шаблоны проектов и скрипты сборки (Buildscripts)

## 1. Файл `gradle.properties` (Стандарт 1.21.4)

```properties
# Параметры JVM и кэширования
org.gradle.jvmargs=-Xmx2G
org.gradle.caching=true
org.gradle.parallel=true

# Версии Minecraft и зависимостей
minecraft_version=1.21.4
yarn_mappings=1.21.4+build.7
loader_version=0.16.14
fabric_version=0.119.4+1.21.4

# Метаданные мода (строго X.X или X.Y.Z без 'v')
mod_id=examplemod
mod_version=1.0.0
maven_group=com.example.mod
archives_base_name=ExampleMod-1.21.4-byMr712
```

---

## 2. Скрипт сборщика `build.gradle` (Fabric Loom для 1.21.4)

```groovy
plugins {
    id 'fabric-loom' version "1.10.1"
    id 'maven-publish'
}

version = project.mod_version
group = project.maven_group

base {
    archivesName = project.archives_base_name
}

repositories {
    mavenCentral()
    exclusiveContent {
        forRepository {
            maven {
                name = "Modrinth"
                url = "https://api.modrinth.com/maven"
            }
        }
        filter {
            includeGroup "maven.modrinth"
        }
    }
    maven { url = "https://maven.shedaniel.me/" }
    maven { url = "https://maven.terraformersmc.com/releases/" }
}

def gitCommitHash = { ->
    try {
        ExecOutput result = providers.exec {
            commandLine = ['git', 'rev-parse', '--short', 'HEAD']
            ignoreExitValue = true
        }
        return result.getResult().get().getExitValue() != 0 ? "<UnknownCommit>" : result.standardOutput.asText.get().strip()
    } catch (Exception e) {
        return "<UnknownCommit>"
    }
}

processResources {
    def hash = gitCommitHash()
    def version = project.version

    inputs.properties("version": version, "commit": hash)
    filesMatching("fabric.mod.json") {
        expand "version": version, "commit": hash
    }
}

dependencies {
    minecraft "com.mojang:minecraft:${project.minecraft_version}"
    mappings "net.fabricmc:yarn:${project.yarn_mappings}:v2"
    modImplementation "net.fabricmc:fabric-loader:${project.loader_version}"
    modImplementation "net.fabricmc.fabric-api:fabric-api:${project.fabric_version}"
}

def targetJavaVersion = 21
tasks.withType(JavaCompile).configureEach {
    it.options.encoding = "UTF-8"
    if (targetJavaVersion >= 10 || JavaVersion.current().isJava10Compatible()) {
        it.options.release = targetJavaVersion
    }
}

java {
    def javaVersion = JavaVersion.toVersion(targetJavaVersion)
    if (JavaVersion.current() < javaVersion) {
        toolchain.languageVersion = JavaLanguageVersion.of(targetJavaVersion)
    }
}

jar {
    from("LICENSE") {
        rename { "${it}_${project.archives_base_name}" }
    }
}

remapJar {
    archiveFileName = "${project.archives_base_name}.jar"
}

// Отключение генерации лишних jar-файлов
tasks.matching { it.name == 'sourcesJar' || it.name == 'remapSourcesJar' || it.name == 'javadocJar' }.configureEach {
    enabled = false
}
```

---

## 3. Скрипт быстрого запуска сборки `build.bat`

Размещается в корне проекта:
```bat
@echo off
call gradlew.bat build
pause
```

---

## 4. Файл `.gitignore`

```gitignore
.gradle/
build/
bin/
run/
out/
.idea/
*.iml
*.ipr
*.iws
.project
.classpath
.settings/
.vscode/
*.DS_Store
```
