# Инструкция о том, как внедрить AspectJ в Android

## Шаг 1
Добавте в <i>build.gradle(APP NAME)</i> этот код

```gradle 
buildscript {
    ..
    dependencies {
        classpath 'org.aspectj:aspectjtools:1.9.4'
        classpath 'org.aspectj:aspectjweaver:1.9.7'
    }
}
```

## Шаг 2
Добавте в <i>build.gradle(:app)</i> этот код

```gradle
apply plugin: 'com.android.application'

/* other code */

dependencies {
    ..
    implementation 'org.aspectj:aspectjrt:1.9.4'
}

repositories {
    mavenCentral()
}
```

Также добавте в <i>build.gradle(:app)</i> это
P.S.: Да, это java-код, но не волнуйтесь (если вы волновались вообще) - просто вставте его 

```java 
import org.aspectj.bridge.IMessage
import org.aspectj.bridge.MessageHandler
import org.aspectj.tools.ajc.Main

/* other code */

android.applicationVariants.all { variant ->
    JavaCompile javaCompile = variant.javaCompile
    javaCompile.doLast {
        String[] args = ["-showWeaveInfo",
                         "-1.5",
                         "-inpath", javaCompile.destinationDir.toString(),
                         "-aspectpath", javaCompile.classpath.asPath,
                         "-d", javaCompile.destinationDir.toString(),
                         "-classpath", javaCompile.classpath.asPath,
                         "-bootclasspath", project.android.bootClasspath.join(
                File.pathSeparator)]

        MessageHandler handler = new MessageHandler(true);
        new Main().run(args, handler)

        def log = project.logger
        for (IMessage message : handler.getMessages(null, true)) {
            switch (message.getKind()) {
                case IMessage.ABORT:
                case IMessage.ERROR:
                case IMessage.FAIL:
                    log.error message.message, message.thrown
                    break;
                case IMessage.WARNING:
                case IMessage.INFO:
                    log.info message.message, message.thrown
                    break;
                case IMessage.DEBUG:
                    log.debug message.message, message.thrown
                    break;
            }
        }
    }
}
```

____
Всё готово! Можете теперь проверить на примере:

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(value = ElementType.METHOD)
@Retention(value = RetentionPolicy.RUNTIME)
public @interface ThreadSQLAnnotation {
}
```

```java
@ThreadSQLAnnotation
public void doSomething() {
        // Тело функции
}
```

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;

@Aspect
public class AspectThreading {
    @Pointcut("@annotation(ThreadSQLAnnotation)")
    public void putThread() {}

    @Around(value = "putThread()")
    public void putAround(final ProceedingJoinPoint joinPoint){
        new Thread(() -> {
            try {
                joinPoint.proceed();
            } catch (Throwable throwable) {
                throwable.printStackTrace();
            }
        }).start();
    }
}
```

В итоге благодаря этому коду и Aspect наши функции могу выполнятся в другом потоке, если на них просто повесить аннотацию.

Здесь более подробно расписано про Pointcut --> https://www.baeldung.com/spring-aop-pointcut-tutorial

____
Ссылка откуда брал --> https://github.com/Archinamon/android-gradle-aspectj
