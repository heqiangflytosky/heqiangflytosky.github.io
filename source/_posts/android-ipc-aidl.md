---
title: Android IPC 之 AIDL
categories: Android
comments: true
tags: [Android, IPC, AIDL]
description: 介绍编写AIDL文件的方式实现进程间通信
date: 2015-2-1 10:00:00
---
## 概述

AIDL是Android Interface Definition Language的简称，也就是Android接口定义语言，通过它我们可以定义进程间的通信接口。

## 实例

我们现在通过一个实例来介绍 AIDL 的使用方法。实例中服务端维护了一个`Student`列表，客户端可以通过跨进程调用来向列表添加和查询数据。

### 创建AIDL接口
AIDL文件支持的数据类型：

 - 基本数据类型（int, long, boolean, float, double）；
 - String和CharSequence；
 - List或Map类型：List或Map中的所有元素必须是AIDL支持的类型之一，或者是一个其他AIDL生成的接口，或者是定义的`parcelable`。

如果我们需要使用非默认支持的数据类型，就要我们自己定义一个`parcelable`对象。

AIDL文件的创建方法如图所示：

![效果图](/images/android-ipc-aidl/add-aidl.png)

创建完AIDL文件，工程的目录会变成下图所示的样子，多了一个aidl包，和java在同一个层级之下。并且aidl文件的包和java文件的包是一样的。

![效果图](/images/android-ipc-aidl/add-aidl-structure.png)

在这里有两个AIDL文件：

 - Student.aidl：用来定义进程间需要传输的数据对象的parcelable类，以供其他AIDL文件使用AIDL中非默认支持的数据类型的。
 - StudentManager.aidl：用来定义一些来完成跨进程通信的接口的类。

下面来看这两个接口的实现：

Student.aidl
```java
package com.android.hq.aidldemo;

parcelable Student;

```
StudentManager.aidl
```java
package com.android.hq.aidldemo;

import com.android.hq.aidldemo.Student;

interface StudentManager {
    List<Student>  getStudents();
    //传参时除了Java基本类型以及String，CharSequence之外的类型，都需要在前面指定定向tag，具体加什么根据自己需要
    void addStudent(inout Student student);
}

```

这里，虽然Studet.java和StudentManager.aidl在同一个包下面，但`Studet`不是系统默认支持的数据类型，使用的时候必须添加`import`进来。

Student.java
`Student`是用于跨进程传输数据使用的，因此必须进行序列化，选择的序列化方式是实现 `Parcelable` 接口。

```java
public class Student implements Parcelable {
    private String name;
    private int age;
    private int grade;

    public Student(){

    }

    protected Student(Parcel in) {
        // 注意读取的顺序要和写入顺序一致
        name = in.readString();
        age = in.readInt();
        grade = in.readInt();
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public int getGrade() {
        return grade;
    }

    public void setGrade(int grade) {
        this.grade = grade;
    }

    public static final Creator<Student> CREATOR = new Creator<Student>() {
        @Override
        public Student createFromParcel(Parcel in) {
            return new Student(in);
        }

        @Override
        public Student[] newArray(int size) {
            return new Student[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        // 注意写入的顺序要和读取顺序一致
        dest.writeString(name);
        dest.writeInt(age);
        dest.writeInt(grade);
    }

    // 因为Student在StudentManager.aidl中被申明为inout类型，这里必须实现这个方法
    public void readFromParcel(Parcel dest) {
        // 注意读取的顺序要和写入顺序一致
        name = dest.readString();
        age = dest.readInt();
        grade = dest.readInt();
    }

    @Override
    public String toString() {
        return "name = "+name+", age = "+age+", grade = "+grade;
    }
}
```

在这里注意一下，我们知道如果是要在不同的应用中进行进程通信，两个应用必须要相同的 aidl 文件（包名也要一样）。或者是 Client 端 BookManager.aidl 定义的接口在 Service 端必须存在，即 Service 端的接口大于或等于 Client 端的接口。
这里为了方便移植，我们把进程通信需要的文件都放在 aidl 目录下面，当然也包括了 Student.java 这个java文件，但是Android Studio默认是只在`src/main/java`找源文件的，因此需要进行下面的配置，在 build.gradle 文件的`android{}`里面加上下面代码：

```
    sourceSets {
        main {
            java.srcDirs = ['src/main/java', 'src/main/aidl']
        }
    }
```

添加之后编译，工程变成下图现实，aidl也就显示在源文件下面了：

![效果图](/images/android-ipc-aidl/add-source-structure.png)


编译过之后，会生成下面的文件：

```
app/build/generated/source/aidl/debug/com/android/hq/aidldemo/StudentManager.java

```

StudentManager.aidl 这里`void addStudent(inout Student student);`的参数类型必须指定定向tag，否则编译会报下面的错误，这里我们指定的是`inout`类型。一旦申明为`inout`这个类型，`Student`类就必须实现`readFromParcel()`方法。具体`inout`类型介绍可以参考我的下一篇博客。

```
17:18:23.963 [ERROR] [org.gradle.api.Task] aidl E  8954  8954 type_namespace.cpp:130] In file /home/heqiang/AndroidStudioProjects/AIDLDemo/app/src/main/aidl/com/android/hq/aidldemo/StudentManager.aidl line 8 parameter student (argument 1):
aidl E  8954  8954 type_namespace.cpp:130]     'Student' can be an out type, so you must declare it as in, out or inout.

17:18:23.967 [DEBUG] [org.gradle.api.internal.tasks.execution.ExecuteAtMostOnceTaskExecuter] Finished executing task ':app:compileDebugAidl'
17:18:23.967 [LIFECYCLE] [class org.gradle.TaskExecutionLogger] :app:compileDebugAidl FAILED
17:18:23.968 [INFO] [org.gradle.execution.taskgraph.AbstractTaskPlanExecutor] :app:compileDebugAidl (Thread[Daemon worker,5,main]) completed. Took 0.202 secs.
17:18:23.968 [DEBUG] [org.gradle.execution.taskgraph.AbstractTaskPlanExecutor] Task worker [Thread[Daemon worker,5,main]] finished, busy: 0.578 secs, idle: 0.01 secs
17:18:23.974 [ERROR] [org.gradle.BuildExceptionReporter] 
17:18:23.975 [ERROR] [org.gradle.BuildExceptionReporter] FAILURE: Build failed with an exception.
17:18:23.975 [ERROR] [org.gradle.BuildExceptionReporter] 
17:18:23.975 [ERROR] [org.gradle.BuildExceptionReporter] * What went wrong:
17:18:23.975 [ERROR] [org.gradle.BuildExceptionReporter] Execution failed for task ':app:compileDebugAidl'.
17:18:23.975 [ERROR] [org.gradle.BuildExceptionReporter] > java.lang.RuntimeException: com.android.ide.common.process.ProcessException: Error while executing '/home/heqiang/install/android-studio/android-sdk-linux/build-tools/25.0.2/aidl' with arguments {-p/home/heqiang/install/android-studio/android-sdk-linux/platforms/android-25/framework.aidl -o/home/heqiang/AndroidStudioProjects/AIDLDemo/app/build/generated/source/aidl/debug -I/home/heqiang/AndroidStudioProjects/AIDLDemo/app/src/main/aidl -I/home/heqiang/AndroidStudioProjects/AIDLDemo/app/src/debug/aidl -I/home/heqiang/AndroidStudioProjects/AIDLDemo/app/build/intermediates/exploded-aar/com.android.support/appcompat-v7/25.2.0/aidl -I/home/heqiang/AndroidStudioProjects/AIDLDemo/app/build/intermediates/exploded-aar/com.android.support/support-v4/25.2.0/aidl -I/home/heqiang/AndroidStudioProjects/AIDLDemo/app/build/intermediates/exploded-aar/com.android.support/support-fragment/25.2.0/aidl -I/home/heqiang/AndroidStudioProjects/AIDLDemo/app/build/intermediates/exploded-aar/com.android.support/support-media-compat/25.2.0/aidl -I/home/heqiang/AndroidStudioProjects/AIDLDemo/app/build/intermediates/exploded-aar/com.android.support/support-core-ui/25.2.0/aidl -I/home/heqiang/AndroidStudioProjects/AIDLDemo/app/build/intermediates/exploded-aar/com.android.support/support-core-utils/25.2.0/aidl -I/home/heqiang/AndroidStudioProjects/AIDLDemo/app/build/intermediates/exploded-aar/com.android.support/animated-vector-drawable/25.2.0/aidl -I/home/heqiang/AndroidStudioProjects/AIDLDemo/app/build/intermediates/exploded-aar/com.android.support/support-vector-drawable/25.2.0/aidl -I/home/heqiang/AndroidStudioProjects/AIDLDemo/app/build/intermediates/exploded-aar/com.android.support/support-compat/25.2.0/aidl -d/tmp/aidl1017216491494308629.d /home/heqiang/AndroidStudioProjects/AIDLDemo/app/src/main/aidl/com/android/hq/aidldemo/StudentManager.aidl}
17:18:23.975 [ERROR] [org.gradle.BuildExceptionReporter] 
17:18:23.976 [ERROR] [org.gradle.BuildExceptionReporter] * Try:
17:18:23.976 [ERROR] [org.gradle.BuildExceptionReporter] Run with --stacktrace option to get the stack trace. 
17:18:23.977 [LIFECYCLE] [org.gradle.BuildResultLogger] 
17:18:23.977 [LIFECYCLE] [org.gradle.BuildResultLogger] BUILD FAILED
17:18:23.977 [LIFECYCLE] [org.gradle.BuildResultLogger] 
```

### 代码

AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest package="com.android.hq.aidldemo"
          xmlns:android="http://schemas.android.com/apk/res/android">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>

                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        
        <service android:name=".AIDLService"
            android:process=":remote">

        </service>
    </application>

</manifest>
```

MainActivity.java

```java
public class MainActivity extends AppCompatActivity {

    private final static String TAG = "Client";
    private StudentManager mStudentManager;
    private boolean mConnected = false;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        if(!mConnected){
            bindService();
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if(mConnected){
            unbindService(mServiceConnection);
        }
    }

    private void bindService(){
        Intent intent = new Intent();
        intent.setClassName("com.android.hq.aidldemo","com.android.hq.aidldemo.AIDLService");
        bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);
    }

    public void addStudent(View view){
        if(mStudentManager == null){
            return;
        }
        Student student = new Student();
        student.setName("James");
        student.setAge(7);
        student.setGrade(2);
        Log.d(TAG, "addStudent : "+student);
        try {
            mStudentManager.addStudent(student);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        Log.d(TAG, "addStudent end");
    }

    public void getStudent(View view){
        if(mStudentManager == null){
            return;
        }
        List<Student> list = null;
        try {
            list = mStudentManager.getStudents();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        Log.d(TAG, "getStudent. student list is : "+list);
    }

    private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mStudentManager = StudentManager.Stub.asInterface(service);
            mConnected = true;
            if(mStudentManager != null){
                try {
                    List<Student> list = mStudentManager.getStudents();
                    Log.d(TAG, "onServiceConnected. student list is : "+list);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mConnected = false;
            if(mStudentManager != null){
                try {
                    List<Student> list = mStudentManager.getStudents();
                    Log.d(TAG, "onServiceDisconnected. student list is : "+list);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        }
    };
}
```

AIDLService.java

```java
public class AIDLService extends Service {
    private final static String TAG = "AIDLService";

    private final MyStudentManager mStudentManager = new MyStudentManager();
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "called onBind");
        return mStudentManager;
    }

    @Override
    public void onCreate() {
        super.onCreate();

    }

    public static class MyStudentManager extends StudentManager.Stub{
        private List<Student> mStudents = new ArrayList<>();

        public MyStudentManager() {
            Student student = new Student();
            student.setName("John");
            student.setAge(6);
            student.setGrade(1);
            mStudents.add(student);
        }

        @Override
        public List<Student> getStudents() throws RemoteException {
            Log.d(TAG, "call getStudents(), student list is :"+mStudents.toString());
            return mStudents;
        }

        @Override
        public void addStudent(Student student) throws RemoteException {
            if(!mStudents.contains(student)){
                mStudents.add(student);
            }
            Log.d(TAG, "call addStudent(), student list is :"+mStudents.toString());
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.d(TAG, "sleep end !");
        }
    }
}
```

## 结果

如图所示，Clent和Server是在两个不同的进程中的：

![效果图](/images/android-ipc-aidl/process.png)

启动后分别点击添加和获取列表按钮，打印如下：
```
06-08 19:30:10.047 10005 10005 D AIDLService: called onBind
06-08 19:30:10.072 10005 10017 D AIDLService: call getStudents(), student list is :[name = John, age = 6, grade = 1]
06-08 19:30:10.073  9991  9991 D Client  : onServiceConnected. student list is : [name = John, age = 6, grade = 1]

06-08 19:32:51.814  9991  9991 D Client  : addStudent : name = James, age = 7, grade = 2
06-08 19:32:51.815 10005 10016 D AIDLService: call addStudent(), student list is :[name = John, age = 6, grade = 1, name = James, age = 7, grade = 2]

06-08 19:33:09.875 10005 10017 D AIDLService: call getStudents(), student list is :[name = John, age = 6, grade = 1, name = James, age = 7, grade = 2]
06-08 19:33:09.876  9991  9991 D Client  : getStudent. student list is : [name = John, age = 6, grade = 1, name = James, age = 7, grade = 2]

```

成功的进行了进程间通信。

## 问题

### Android进程间通信是异步的还是同步的？

我们可以在程序中验证这个问题。
我们在`AIDLService`类的`mStudentManager`实现的方法`addStudent()`中添加一个延时：

```java
        @Override
        public void addStudent(Student student) throws RemoteException {
            if(!mStudents.contains(student)){
                mStudents.add(student);
            }
            Log.d(TAG, "call addStudent(), student list is :"+mStudents.toString());
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.d(TAG, "sleep end !");
        }
```

我们在客户端调用`addStudent()`方法的结尾添加打印，这样可以得到函数调用结束的信息：

```java
    public void addStudent(View view){
        ......
        Log.d(TAG, "addStudent : "+student);
        try {
            mStudentManager.addStudent(student);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        Log.d(TAG, "addStudent end");
    }
```

看到打印：
```
06-08 19:42:28.505 20807 20807 D Client  : addStudent : name = James, age = 7, grade = 2
06-08 19:42:28.506 20822 20833 D AIDLService: call addStudent(), student list is :[name = John, age = 6, grade = 1, name = James, age = 7, grade = 2]
06-08 19:42:30.506 20822 20833 D AIDLService: sleep end !
06-08 19:42:30.506 20807 20807 D Client  : addStudent end
```
看到客户端等服务端调用完毕才会往下走，因此我们可以得到这样的结论：
**Android进程间通信默认是同步的**，这里为什么说默认呢？难道还有其他情况，是的，也可以是异步的，为什么呢？
看 StudentManager.java 中的 `addStudent` 方法：

```java
@Override public void addStudent(com.android.hq.aidldemo.Student student) throws android.os.RemoteException
{
...
mRemote.transact(Stub.TRANSACTION_addStudent, _data, _reply, 0);
...
}
```

再来看一下 `IBinder` 中对这个方法的介绍：

```java
    /**
     * Perform a generic operation with the object.
     * 
     * @param code The action to perform.  This should
     * be a number between {@link #FIRST_CALL_TRANSACTION} and
     * {@link #LAST_CALL_TRANSACTION}.
     * @param data Marshalled data to send to the target.  Must not be null.
     * If you are not sending any data, you must create an empty Parcel
     * that is given here.
     * @param reply Marshalled data to be received from the target.  May be
     * null if you are not interested in the return value.
     * @param flags Additional operation flags.  Either 0 for a normal
     * RPC, or {@link #FLAG_ONEWAY} for a one-way RPC.
     */
    public boolean transact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException;
```
`flags` 为0时是普通的RPC调用，为 `FLAG_ONEWAY` 时是 one-way RPC，是单向调用，执行后立即返回，无需等待Server端 `transact()` 返回。这个时候就是异步执行了。
