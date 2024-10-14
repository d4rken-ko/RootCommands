# 더 이상 활발하게 개발되지 않음

**RootCommands는 더 이상 활발하게 개발되지 않습니다. 유지 관리를 맡고 싶으시다면 간단히 포크해서 수정 사항을 구현하세요. 저는 풀 리퀘스트와 릴리스 병합과 같은 기본적인 유지 관리만 할 것입니다.**

# 루트 명령

이는 Android OS에서 루트 명령의 사용을 단순화하는 라이브러리입니다. 네이티브를 둘러싼 Java 래퍼입니다. 모든 Android OS와 함께 제공되는 바이너리이지만, 사용자 고유의 네이티브 바이너리를 패키징하고 실행하는 데에도 사용할 수 있습니다.

## 라이브러리를 Gradle 종속성으로 사용(Android 라이브러리 프로젝트)

1. ``libraries/RootCommands``를 프로젝트에 복사하고 ``settings.gradle``에 포함합니다(https://github.com/dschuermann/root-commands/blob/master/settings.gradle 참조)
2. 프로젝트 ``build.gradle``에 종속성 ``compile project(':libraries:RootCommands')``를 추가합니다. (https://github.com/dschuermann/root-commands/blob/master/ExampleApp/build.gradle 참조)

# 예시

RootCommands가 실제로 어떻게 작동하는지 보려면 ``RootCommands Demo`` 프로젝트를 컴파일하고 logcat 출력을 살펴보세요.

## 디버그 모드

다음 줄을 사용하여 RootCommands에서 디버그 로깅을 활성화할 수 있습니다.
```java
RootCommands.DEBUG = true;
```

## 루트 접근 확인

This tries to find the su binary, opens a root shell and checks for root uid.

```java
if (RootCommands.rootAccessGiven()) {
    // your code
}
```

## 간단한 명령

실행하려는 셸 명령으로 SimpleCommands를 인스턴스화할 수 있습니다. 이는 셸에서 무언가를 실행하는 매우 기본적인 접근 방식입니다.

```java
// 루트 셸 시작
Shell shell = Shell.startRootShell();

// 간단한 명령
SimpleCommand command0 = new SimpleCommand("echo this is a command",
        "echo this is another command");
SimpleCommand command1 = new SimpleCommand("toolbox ls");
SimpleCommand command2 = new SimpleCommand("ls -la /system/etc/hosts");

shell.add(command0).waitForFinish();
shell.add(command1).waitForFinish();
shell.add(command2).waitForFinish();

Log.d(TAG, "Output of command2: " + command2.getOutput());
Log.d(TAG, "Exit code of command2: " + command2.getExitCode());

// ROOT 셸 닫기
shell.close();
```

## 자신의 명령을 정의하세요

더 복잡한 명령의 경우 셸이 명령을 실행하는 동안 출력을 구문 분석하도록 Command 클래스를 확장할 수 있습니다.

```java
private class MyCommand extends Command {
    private static final String LINE = "hosts";
    boolean found = false;

    public MyCommand() {
        super("ls -la /system/etc/");
    }

    public boolean isFound() {
        return found;
    }

    @Override
    public void output(int id, String line) {
        if (line.contains(LINE)) {
            Log.d(TAG, "Found it!");
            found = true;
        }
    }

    @Override
    public void afterExecution(int id, int exitCode) {
    }

}
```

```java
// ROOT 셸 시작
Shell shell = Shell.startRootShell();

// 사용자 정의 명령 클래스:
MyCommand myCommand = new MyCommand();
shell.add(myCommand).waitForFinish();

Log.d(TAG, "myCommand.isFound(): " + myCommand.isFound());

// ROOT 셸을 닫습니다
shell.close();
```

## 도구 상자

Toolbox는 BusyBox와 비슷하지만 일반적으로 모든 Android OS에 탑재되어 있습니다. 도구 상자 명령은 다음에서 찾을 수 있습니다 https://github.com/CyanogenMod/android_system_core/tree/ics/toolbox . 즉, 이러한 명령은 모든 Android OS에서 작동하도록 설계되었습니다. _작동하는_ 도구상자 바이너리가 있습니다. BusyBox가 필요하지 않습니다!

Toolbox 클래스는 이 도구 상자 실행 파일을 기반으로 하며 다음과 같은 Java 메서드로 몇 가지 유용한 명령을 제공합니다.

* isRootAccessGiven()
* killAll(String processName)
* isProcessRunning(String processName)
* getFilePermissions(String file)
* setFilePermissions(String file, String permissions)
* getSymlink(String file)
* copyFile(String source, String destination, boolean remountAsRw, boolean preservePermissions)
* reboot(int action)
* withWritePermissions(String file, WithPermissions withWritePermission)
* setSystemClock(long millis)
* remount(String file, String mountType)
* ...

```java
Shell shell = Shell.startRootShell();

Toolbox tb = new Toolbox(shell);

if (tb.isRootAccessGiven()) {
    Log.d(TAG, "Root access given!");
} else {
    Log.d(TAG, "No root access!");
}

Log.d(TAG, tb.getFilePermissions("/system/etc/hosts"));

shell.close();
```

## 실행 파일

Android APK는 일반적으로 네이티브 실행 파일을 포함하도록 설계되지 않았습니다. 그러나 이들은 다양한 아키텍처에 대한 네이티브 라이브러리를 포함하도록 설계되었습니다. 앱이 장치에 설치될 때 배포됩니다. 안드로이드 메커니즘은 기기의 아키텍처에 따라 적절한 네이티브 라이브러리를 배포합니다.
이 방법은 프로젝트의 libs 폴더에 포함된 ``lib*.so``와 같은 이름의 파일만 배포합니다.

우리는 네이티브 실행 파일을 배포하기 위한 Android 라이브러리 방법이 부족합니다. 컴파일 후 이름을 바꾸면, APK에 포함되고 아키텍처에 따라 배포됩니다.

참고: 배포된 파일의 권한 및 소유자: ``-rwxr-xr-x system   system      38092 2012-09-24 19:51 libhello_world_exec.so``

1. 네이티브 실행 파일의 소스를 jni 폴더에 넣으세요 https://github.com/dschuermann/root-commands/tree/master/ExampleApp/jni
2. Android.mk와 Application.mk를 직접 작성하세요
3. 이름 변경 프로세스를 자동화하기 위해 Gradle 작업을 제안합니다 https://github.com/dschuermann/root-commands/blob/master/ExampleApp/build.gradle. 이렇게 하면 파일 이름이 ``*``에서 ``lib*_bin.so``로 변경됩니다.
4. ``ndk-build``를 실행하여 실행 파일을 빌드합니다.
5. ``gradle renameExecutables``를 실행합니다.
6. ``gradle build``를 실행합니다.

이제 실행 파일이 묶여 있으므로 다음 예와 같이 ``SimpleExecutableCommand``를 사용할 수 있습니다.

```java
SimpleExecutableCommand execCommand = new SimpleExecutableCommand(this, "hello_world", "");

// ROOT 없이 일반 셸로 시작하지만 ROOT에서 실행 파일을 시작할 수도 있습니다.
// 더 많은 권한이 필요하면 셸을 사용하세요!
Shell shell = Shell.startShell();

shell.add(execCommand).waitForFinish();

Toolbox tb = new Toolbox(shell);
if (tb.killAllBinary("hello_world")) {
    Log.d(TAG, "Hello World 데몬이 죽었습니다!");
} else {
    Log.d(TAG, "죽이는 데 실패했습니다!");
}

shell.close();
```

# 기여하다

RootCommands를 포크하고 풀 리퀘스트를 하세요. 저는 여러분의 변경 사항을 메인 프로젝트에 다시 병합할 것입니다.

# 기타 문서
* http://su.chainfire.eu/

# 기타 루트 라이브러리
* https://github.com/Chainfire/libsuperuser
* http://code.google.com/p/roottools/
* https://github.com/SpazeDog/rootfw

# 저자
RootCommands는 다른 여러 오픈 소스 프로젝트를 기반으로 하며 여기에는 여러 작성자가 참여했습니다.

* Dominik Schürmann(루트 명령)
* Michael Elsdörfer(Android 자동 시작)
* Stephen Erickson, Chris Ravenscroft, Adam Shanks, Jeremy Lakeman(RootTools)
