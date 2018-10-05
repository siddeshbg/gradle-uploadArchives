# Gradle project to demonstrate uploading artifacts to Maven Repo

## Introducing the sample java application used for this demo
This is an advanced HelloWorld java application and it's original code is picked from 
[https://guides.gradle.org/consuming-jvm-libraries/](https://guides.gradle.org/consuming-jvm-libraries/)

This is a greeter application that produces an ASCII-art type greeting.

### Building this application

```
./gradlew -b legacy.gradle jar

or

./gradlew -b legacy.gradle build
```

Since this project contains multiple gradle files, we are using the simple legacy.gradle file for this ex.

### Testing this application manually
```aidl
./gradlew -b legacy.gradle installDist
```

The Application plugin provides `installDist` task to install your application into the build/install folder for validation purposes

#### Run the application
```aidl
./build/install/greeterApp/bin/greeterApp Gradle
```

## Publishing the produced artifact to local maven repo using legacy method
* This is the legacy publishing mechanism, should not be used in newer builds
* This uses the `'maven'` plugin
  * The Maven plugin adds support for deploying artifacts to Maven repositories
* We need to declare the outgoing artifacts of a project
* Artifacts are grouped by configurations
  * For each configuration in your project, Gradle provides the tasks uploadConfigurationName and buildConfigurationName 
  when the base plugin is applied
  * Execution of these tasks will build or upload the artifacts belonging to the respective configuration
  * The **archives** configuration is the standard configuration to assign your artifacts to.
  * The *Java plugin* automatically assigns the default jar to this **archives** configuration

### Using the Maven plugin
```legacy.gradle
plugins {
    id "maven"
}
```

### Declaring artifacts
```legacy.gradle
def someFile = file('build/scripts/greeterApp.bat')

artifacts {
    archives someFile
}
```
The custom artifact(someFile) created above is not automatically assigned to any configuration. 
We need to explicitly do this assignment (`archives someFile`)

### Publishing artifacts
* configure the upload task and define where to publish the artifacts to.
```legacy.gradle
uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: "file://localhost/tmp/myRepo/")
        }
    }
}
```
* Calling the uploadArchives task will generate the POM and deploys the artifact and the POM to the specified repository.
```aidl
./gradlew -b legacy.gradle uploadArchives
```

## Publishing the produced artifact to local maven repo using 'maven-publish' plugin
In Gradle, the publishing process looks like this:
1. Define what to publish
  * Gradle needs to know what files and information to publish
  * This is typically a combination of artifacts and metadata that Gradle calls a **publication**
2. Define where to publish it to
  * Gradle needs to know where to publish artifacts
  * This is done via **repositories**, provide the type of the repository and its location
3. Do the publishing
  * Gradle automatically generates publishing tasks for all possible combinations of publication and repository, 
  allowing you to publish any artifact to any repository
  * If you’re publishing to a Maven repository, the tasks are of type **PublishToMavenRepository**

### Apply the 'maven-publish' plugin
```mvn-pub.gradle
plugins {
    id 'maven-publish'
}
```

#### Example 1: publish the project’s production JAR file — the one produced by the jar task — to a custom, Maven repository.
```mvn-pub.gradle
group = 'org.temp'
version = '2.0'

publishing {
    publications {
        myLibrary(MavenPublication) {
            from components.java
        }
    }

    repositories {
        maven {
            name = 'myRepo'
            url = "file://${buildDir}/repo"
        }
    }
}
```
* This defines a publication called "myLibrary" that can be published to a Maven repository
* This publication consists of just the production JAR artifact (greeterApp-*.jar) and its metadata which combined are 
represented by the java component of the project
  * Components are the standard way of defining a publication. 
  * They are provided by plugins. 
  * For example, the Java Plugin defines the components.java SoftwareComponent
* The example also defines a file-based Maven repository with the name "myRepo".
  * but real-world builds typically work with HTTPS-based repository servers
* In combination with the project’s group and version, the publication and repository definitions provide everything 
that Gradle needs to publish the project’s production JAR
* Gradle will then create a dedicated publishMyLibraryPublicationToMyRepoRepository task that does just that
* You can either execute the individual publishing tasks directly, or you can execute publish, which will run all the 
available publishing tasks
* In this example, publish will just run publishMyLibraryPublicationToMavenRepository.

```aidl
./gradlew -b mvn-pub.gradle clean publish
```
This will publish artifact under `build/repo/org/temp/consuming-jvm-libraries/2.0/`

#### Example 2: Adding custom artifacts to a publication
* This example is to publish additional artifacts like "-sources" and "-javadoc" JARs
* Refer mvn-publish-custom.gradle
```mvn-publish-custom.gradle
task sourcesJar(type: Jar) {
    classifier = 'sources'
    from sourceSets.main.allJava
}

task javadocJar(type: Jar) {
    classifier = 'javadoc'
    from javadoc.destinationDir
}

publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java

            artifact sourcesJar
            artifact javadocJar
        }
    }

    repositories {
        maven {
            name = 'myRepo'
            url = "file://${buildDir}/repo"
        }
    }
}

```
* The artifact() method accepts archive tasks as an argument — like sourcesJar in the sample — as well as any type of 
argument accepted by Project.file(java.lang.Object), such as a File instance or string file path.
* Custom artifacts need to be distinct within a publication, typically via a unique combination of classifier and extension
* When you’re attaching extra artifacts to a publication, remember that they are secondary artifacts that support a primary artifact. 
* The metadata that a publication defines — such as dependency information — is associated with that primary artifact only.

```aidl
./gradlew -b mvn-pub-custom.gradle publish
```

#### Example 3: Publishing a custom primary artifact (no component)
* If your build produces a primary artifact that isn’t supported by a predefined component, then you will need to configure a custom artifact
* Refer mvn-publish-custom2.gradle

```mvn-publish-custom2.gradle
def initScript = file("$buildDir/scripts/consuming-jvm-libraries.bat")

def scriptArtifact = artifacts.add('archives', initScript) {
    type 'bat'
}

publishing {
    publications {

        mavenJava(MavenPublication) {
            from components.java

            artifact sourcesJar
            artifact javadocJar
            artifact scriptArtifact
        }
    }

    repositories {
        maven {
            name = 'myRepo'
            url = "file://${buildDir}/repo"
        }
    }
}
```
* The artifacts.add() method returns an artifact object of type PublishArtifact that can then be used in defining a publication.
* Now you can publish the sh as well as depend on it from another project using the project(path: ':my-project', configuration: 'archives') syntax.

```aidl
./gradlew -b mvn-pub-custom2.gradle installDist publish
```

#### Example 4: Publishing project’s production JAR file and custom artifacts to a Nexus
* Refer build.gradle
* Modify Nexus url, credentials before running this build.gradle
```
publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java

            artifact sourcesJar
            artifact javadocJar
        }
    }

    repositories {
        maven {
            url 'https://nexus.myorg.com/repository/third-party-lib/'
            credentials {
                username 'userid'
                password '*******'
            }
        }
    }
}
```

and to publish to Nexus
```aidl
./gradlew publish
```




