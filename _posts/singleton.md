Singleton pattern
instance를 오직 한개만 제공하는 class (ex. setting info)

```java
public class Settings{
    private static Settings instance;

    private Settings() {}

    public static Settings getInstance() {
        if (instance == null) {
            instance = new Settings();
        }
        return instance;
    }
}

public class App{
    public static void main(String[] args) {
        Settings settings = Settings.getInstance();
    }
}
```
생성자를 private로 구현하여 외부에서 생성하지 못하도록 방지한다.
하지만 multi-thread 환경에서 thread safe하지 않다.

이를 해결하기 위한 방법은 여러가지다.

첫번째로 synchronized keyword를 사용하는 것이다.
```java
public class Settings{
    private static Settings instance;

    private Settings() {}

    public static synchronized Settings getInstance() {
        if (instance == null) {
            instance = new Settings();
        }
        return instance;
    }
}
```
synchronized keyword를 사용한다면 getInstance()를 한번에 하나의 thread만 접근하도록 동기화하여 여러 Settings가 생기는 문제를 방지할 수 있다. 다만 getInstance() 자체를 한번에 하나의 thread만 접근 가능하므로 성능상 손실을 볼 수 밖에 없다.

두번째로 eager initialization 방법을 사용한다.
```java
public class Settings{
    private static final Settings INSTANCE = new Settings();

    private Settings() {}

    public static Settings getInstance() {
        return INSTANCE;
    }
}
```
최초 초기화 시에 미리 Settings instance를 생성하면 thread마다 이미 만들어진 instance를 접근하게 되므로 thread safe하게 된다.
다만 미리 instance를 생성한다는 부분 자체가 단점이 될 수 있다. 만약 instance를 생성하는 과정이 비용이 큰데(시간이 오래 걸리거나, memory를 많이 차지하거나) 사용을 하지 않을 수도 있다.

위의 문제는 double checked locking을 사용하면 좀 더 효율적으로 동기화를 진행할 수 있다.
```java
public class Settings{
    private static volatile Settings instance;

    private Settings() {}

    public static Settings getInstance() {
        if (instance == null) {
            synchronized (Settings.class) {
                if (instance == null) {
                    instance = new Settings();
                }
            }
        }
        return instance;
    }
}
```
instance를 생성하는 부분만 동기화를 진행하여 실제로 생성은 한번만 되도록 막고, instance가 이미 존재할 경우엔 동기화를 진행하지 않고도 instance를 가져올 수 있도록 허용하여 getInstance() 전체를 동기화 할 때보다 성능을 높일 수 있다.

다른 방법으로는 static inner class를 사용하는 방법이 있다.
```java
public class Settings{
    private Settings() {}

    private static class SettingsHolder {
        private static final Settings INSTANCE = new Settings();
    }

    public static Settings getInstance() {
        return SettingsHolder.INSTANCE;
    }
}
```
SettingsHolder가 처음 호출 될 때 INSTANCE가 정의되어 생성되어 lazy loading이 가능하고, 이후에는 INSTANCE를 직접 접근하기 때문에 thread safe한 방법이다.

## Singleton을 깨뜨리는 방법

1. reflection.
```java
public class App{
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, InstantiationException {
        Settings settings = Settings.getInstance();

        Constructor<Settings> constructor = Settings.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        Settings settings1 = constructor.newInstance();

        System.out.println(settings == settings1); // false
    }
}
```

2. Serialize & Deserialize
```java
public class Settings implements Serializable {
    ...
}

public class App {
    public static void main(String[] args) throws IOException {
        Settings settings = Settings.getInstance();
        Settings settings1 = null;

        try (ObjectOutput out = new ObjectOutputStream(new FileOutputStream("settings.obj"))) {
            out.writeObject(settings);
        }

        try (ObjectInput in = new ObjectInputStream(new FileInputStream("settings.obj"))) {
            settings1 = (Settings) in.readObject();
        }

        System.out.println(settings == settings1); // false
    }
}
```
대응방안
```java
public class Settings implements Serializable {
    ...
    protected Object readResolve() {
        return getInstance();
    }
}
```

## reflection까지 막는 방법
```java
public enum Settings {
    INSTANCE;
}

public class App {
    public static void main(String[] args) {
        Settings settings = Settings.INSTANCE;
        ...
    }
}
```
enum을 사용할 경우 reflection까지 방지할 수 있다. 하지만 이 방법은 eager initialization 처럼 미리 만들어지기 때문에, eager initialization의 단점과 같다. 또한 상속을 사용하지 못하는 특징도 있다.
