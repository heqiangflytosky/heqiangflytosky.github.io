---
title: Android热修复之AndFix源码分析
categories: Android开源项目
comments: true
tags: [Android, 热修复, AndFix]
date: 2016-12-16 10:00:00
---
## AndFix基本原理
AndFix实现热修复的原理就是用补丁文件的方法来替换原来有bug的方法。这一步骤是在Native层使用指针替换的方式来实现的。
基本原理：
<img src="https://github.com/alibaba/AndFix/raw/master/images/principle.png"/>
实现热修复整体流程：
<img src="https://github.com/alibaba/AndFix/raw/master/images/process.png"/>
AndFix通过Java的自定义注解来判断一个方法是否应该被替换，如果可以就会hook该方法并进行替换。AndFix在ART架构上的Native方法是`art_replaceMethod`、在X86架构上的Native方法是`dalvik_replaceMethod`。他们的实现方式是不同的。对于Dalvik，它将改变目标方法的类型为Native同时hook方法的实现至AndFix自己的Native方法，这个方法称为`dalvik_dispatcher`,这个方法将会唤醒已经注册的回调，这就是我们通常说的hooked（挂钩）。对于ART来说，我们仅仅改变目标方法的属性来替代它。

## 源码分析
### Java源码
我们根据AndFix的步骤来解析。
#### 初始化
##### 创建一个PatchManager实例：
```java
mPatchManager = new PatchManager(this);
```
PatchManager源码：
```java
public PatchManager(Context context) {
	mContext = context;
	mAndFixManager = new AndFixManager(mContext);//初始化AndFixManager
	mPatchDir = new File(mContext.getFilesDir(), DIR);//初始化存放patch补丁文件的文件夹
	mPatchs = new ConcurrentSkipListSet<Patch>();//初始化存在Patch类的集合,即需要修复类的集合,此类适合大并发
	mLoaders = new ConcurrentHashMap<String, ClassLoader>();//初始化存放类对应的类加载器集合
}
```
1. 初始化AndFixManager：
```java
public AndFixManager(Context context) {
	mContext = context;
	mSupport = Compat.isSupport();//判断Android机型是否适支持AndFix
	if (mSupport) {
		mSecurityChecker = new SecurityChecker(mContext);//初始化签名判断类
		mOptDir = new File(mContext.getFilesDir(), DIR);
		if (!mOptDir.exists() && !mOptDir.mkdirs()) {// make directory fail
			mSupport = false;
			Log.e(TAG, "opt dir create error.");
		} else if (!mOptDir.isDirectory()) {// not directory
			mOptDir.delete();
			mSupport = false;
		}
	}
}
```
1.1 判断是否支持AndFix：
```java
public static synchronized boolean isSupport() {
	if (isChecked)
		return isSupport;

	isChecked = true;
	// not support alibaba's YunOs
	//不支持YunOS系统；AndFix.setup判断是Dalvik还是Art虚拟机，来注册Native方法；isSupportSDKVersion版本SDK判断是否支持，支持android 2.3到android 7.0
	if (!isYunOS() && AndFix.setup() && isSupportSDKVersion()) {
		isSupport = true;
	}

	if (inBlackList()) {
		isSupport = false;
	}

	return isSupport;
}

public static boolean setup() {
	try {
		final String vmVersion = System.getProperty("java.vm.version");
		boolean isArt = vmVersion != null && vmVersion.startsWith("2");//判断是否是art虚拟机
		int apilevel = Build.VERSION.SDK_INT;//API版本判断
		return setup(isArt, apilevel); //这里用到native方法setup，后面再介绍
	} catch (Exception e) {
		Log.e(TAG, "setup", e);
		return false;
	}
}

// from android 2.3 to android 7.0
private static boolean isSupportSDKVersion() {
	if (android.os.Build.VERSION.SDK_INT >= 8
			&& android.os.Build.VERSION.SDK_INT <= 24) {
		return true;
	}
	return false;
	}
```
1.2 签名机制的初始化过程：
```java
public SecurityChecker(Context context) {
	mContext = context;
	init(mContext);
}

// initialize,and check debuggable
private void init(Context context) {
	try {
		PackageManager pm = context.getPackageManager();
		String packageName = context.getPackageName();

		//获得包含签名的包信息
		PackageInfo packageInfo = pm.getPackageInfo(packageName,
				PackageManager.GET_SIGNATURES);
		//证书工厂类，这个类实现了出厂合格证算法的功能
		CertificateFactory certFactory = CertificateFactory
				.getInstance("X.509");
		//将签名转换为字节数组流
		ByteArrayInputStream stream = new ByteArrayInputStream(
				packageInfo.signatures[0].toByteArray());
		//X509证书，X.509是一种非常通用的证书格式
		X509Certificate cert = (X509Certificate) certFactory
				.generateCertificate(stream);
		//以X500Principal的形式返回证书的主体（主体标识名）值，来判断是否是debug的key
		mDebuggable = cert.getSubjectX500Principal().equals(DEBUG_DN);
		//从此证书中获取公钥。 
		mPublicKey = cert.getPublicKey();
	} catch (NameNotFoundException e) {
		Log.e(TAG, "init", e);
	} catch (CertificateException e) {
		Log.e(TAG, "init", e);
	}
}
```
##### 版本的初始化
```java
mPatchManager.init(version);
```
init方法源码：
```java
public void init(String appVersion) {
	if (!mPatchDir.exists() && !mPatchDir.mkdirs()) {// make directory fail
		Log.e(TAG, "patch dir create error.");
		return;
	} else if (!mPatchDir.isDirectory()) {// not directory
		mPatchDir.delete();
		return;
	}
	//从SharedPreferences文件_andfix_.xml中获取patch文件的信息
	SharedPreferences sp = mContext.getSharedPreferences(SP_NAME,
			Context.MODE_PRIVATE);
	String ver = sp.getString(SP_VERSION, null);
	if (ver == null || !ver.equalsIgnoreCase(appVersion)) {
		cleanPatch();//删除本地patch文件，也就是/data/data/***/file/apatch下面的patch文件
		sp.edit().putString(SP_VERSION, appVersion).commit();//并把传入的版本号保存
	} else {
		initPatchs();//初始化patch列表，把本地的patch文件加载到内存
	}
}

private void cleanPatch() {
	File[] files = mPatchDir.listFiles();
	for (File file : files) {
		mAndFixManager.removeOptFile(file);///data/data/***/file/apatch_opt下面的patch文件
		if (!FileUtil.deleteFile(file)) {
			Log.e(TAG, file.getName() + " delete error.");
		}
	}
}

private void initPatchs() {
	File[] files = mPatchDir.listFiles();
	for (File file : files) {
		addPatch(file);//加载Patch文件
	}
}

private Patch addPatch(File file) {
	Patch patch = null;
	if (file.getName().endsWith(SUFFIX)) { //判断是否是.apatch后缀的文件
		try {
			patch = new Patch(file);
			mPatchs.add(patch);
		} catch (IOException e) {
			Log.e(TAG, "addPatch", e);
		}
	}
	return patch;
}
```
Patch文件：
```java
public Patch(File file) throws IOException {
	mFile = file;
	init();
}

@SuppressWarnings("deprecation")
private void init() throws IOException {
	JarFile jarFile = null;
	InputStream inputStream = null;
	try {
		jarFile = new JarFile(mFile);//使用JarFile读取Patch文件
		JarEntry entry = jarFile.getJarEntry(ENTRY_NAME);//获取META-INF/PATCH.MF文件
		inputStream = jarFile.getInputStream(entry);
		Manifest manifest = new Manifest(inputStream);
		Attributes main = manifest.getMainAttributes();
		mName = main.getValue(PATCH_NAME);//获取PATCH.MF属性Patch-Name
		mTime = new Date(main.getValue(CREATED_TIME));//获取PATCH.MF属性Created-Time

		mClassesMap = new HashMap<String, List<String>>();
		Attributes.Name attrName;
		String name;
		List<String> strings;
		for (Iterator<?> it = main.keySet().iterator(); it.hasNext();) {
			attrName = (Attributes.Name) it.next();
			name = attrName.toString();
			//判断name的后缀是否是-Classes，并把name对应的值加入到集合中，对应的值就是class类名的列表
			if (name.endsWith(CLASSES)) {
				strings = Arrays.asList(main.getValue(attrName).split(","));
				if (name.equalsIgnoreCase(PATCH_CLASSES)) {
					mClassesMap.put(mName, strings);
				} else {
					mClassesMap.put(
							name.trim().substring(0, name.length() - 8),// remove
																		// "-Classes"
							strings);
				}
			}
		}
	} finally {
		if (jarFile != null) {
			jarFile.close();
		}
		if (inputStream != null) {
			inputStream.close();
		}
	}

}
```

#### 运行时加载Patch文件
```java
mPatchManager.loadPatch();
```
loadPatch源码：
```java
public void loadPatch() {
	mLoaders.put("*", mContext.getClassLoader());// wildcard
	Set<String> patchNames;
	List<String> classes;
	for (Patch patch : mPatchs) {
		patchNames = patch.getPatchNames();
		for (String patchName : patchNames) {
			classes = patch.getClasses(patchName);
			mAndFixManager.fix(patch.getFile(), mContext.getClassLoader(),
					classes);
		}
	}
}
```
重头戏，Fix bug：
```java
public synchronized void fix(File file, ClassLoader classLoader,
		List<String> classes) {
	if (!mSupport) {
		return;
	}

	//校验签名
	if (!mSecurityChecker.verifyApk(file)) {// security check fail
		return;
	}

	try {
		//校验apatch_opt下面的同名patch文件，防止漏洞攻击
		File optfile = new File(mOptDir, file.getName());
		boolean saveFingerprint = true;
		if (optfile.exists()) {
			// need to verify fingerprint when the optimize file exist,
			// prevent someone attack on jailbreak device with
			// Vulnerability-Parasyte.
			// btw:exaggerated android Vulnerability-Parasyte
			// http://secauo.com/Exaggerated-Android-Vulnerability-Parasyte.html
			if (mSecurityChecker.verifyOpt(optfile)) {
				saveFingerprint = false;
			} else if (!optfile.delete()) {
				return;
			}
		}

		//加载patch文件中的dex
		final DexFile dexFile = DexFile.loadDex(file.getAbsolutePath(),
				optfile.getAbsolutePath(), Context.MODE_PRIVATE);

		if (saveFingerprint) {
			//保存文件md5
			mSecurityChecker.saveOptSig(optfile);
		}

		ClassLoader patchClassLoader = new ClassLoader(classLoader) {
			//重写ClasLoader的findClass方法
			@Override
			protected Class<?> findClass(String className)
					throws ClassNotFoundException {
				Class<?> clazz = dexFile.loadClass(className, this);
				if (clazz == null
						&& className.startsWith("com.alipay.euler.andfix")) {
					return Class.forName(className);// annotation’s class
														// not found
				}
				if (clazz == null) {
					throw new ClassNotFoundException(className);
				}
				return clazz;
			}
		};
		Enumeration<String> entrys = dexFile.entries();
		Class<?> clazz = null;
		while (entrys.hasMoreElements()) {
			String entry = entrys.nextElement();
			if (classes != null && !classes.contains(entry)) {
				continue;// skip, not need fix
			}
			clazz = dexFile.loadClass(entry, patchClassLoader);//获取有bug的类文件
			if (clazz != null) {
				fixClass(clazz, classLoader);
			}
		}
	} catch (IOException e) {
		Log.e(TAG, "pacth", e);
	}
}

private void fixClass(Class<?> clazz, ClassLoader classLoader) {
	Method[] methods = clazz.getDeclaredMethods();
	MethodReplace methodReplace;
	String clz;
	String meth;
	for (Method method : methods) {
		////获取此方法的注解，有bug的方法在生成的patch的类中的方法都是有注解的
		methodReplace = method.getAnnotation(MethodReplace.class);
		if (methodReplace == null)
			continue;
		clz = methodReplace.clazz();//获取注解中clazz的值
		meth = methodReplace.method();//获取注解中method的值
		if (!isEmpty(clz) && !isEmpty(meth)) {
			replaceMethod(classLoader, clz, meth, method);//方法替换
		}
	}
}

private void replaceMethod(ClassLoader classLoader, String clz,
		String meth, Method method) {
	try {
		String key = clz + "@" + classLoader.toString();
		Class<?> clazz = mFixedClass.get(key);//判断此类是否被fix
		if (clazz == null) {// class not load
			Class<?> clzz = classLoader.loadClass(clz);
			// initialize target class
			clazz = AndFix.initTargetClass(clzz);//初始化class
		}
		if (clazz != null) {// initialize class OK
			mFixedClass.put(key, clazz);
			Method src = clazz.getDeclaredMethod(meth,
					method.getParameterTypes());//根据反射获取到有bug的类的方法(有bug的apk)
			AndFix.addReplaceMethod(src, method);//src是有bug的方法，method是补丁方法
		}
	} catch (Exception e) {
		Log.e(TAG, "replaceMethod", e);
	}
}
```

AndFix.addReplaceMethod源码：

```java
public static void addReplaceMethod(Method src, Method dest) {
	try {
		replaceMethod(src, dest);//调用了native方法，具体见JNI方法介绍
		initFields(dest.getDeclaringClass());
	} catch (Throwable e) {
		Log.e(TAG, "addReplaceMethod", e);
	}
}

private static void initFields(Class<?> clazz) {
	Field[] srcFields = clazz.getDeclaredFields();
	for (Field srcField : srcFields) {
		Log.d(TAG, "modify " + clazz.getName() + "." + srcField.getName()
				+ " flag:");
		setFieldFlag(srcField);//调用了native方法，具体见JNI方法介绍
	}
}

private static native void replaceMethod(Method dest, Method src);

private static native void setFieldFlag(Field field);
```

#### 添加Patch

也就是将SDCard中的Patch文件copy到/data/data/***/apatch下面，并加载。

```java
        try {
            // .apatch file path
            String patchFileString = Environment.getExternalStorageDirectory()
                    .getAbsolutePath() + APATCH_PATH;
            Log.e(TAG, "path:" + patchFileString );
            mPatchManager.addPatch(patchFileString);
            Log.e(TAG, "apatch:" + patchFileString + " added.");
        } catch (IOException e) {
            Log.e(TAG, "", e);
        }
```

PatchManager.addPatch源码：

```java
public void addPatch(String path) throws IOException {
	File src = new File(path);
	File dest = new File(mPatchDir, src.getName());
	if(!src.exists()){
		throw new FileNotFoundException(path);
	}
	if (dest.exists()) {
		Log.d(TAG, "patch [" + path + "] has be loaded.");
		return;
	}
	FileUtil.copyFile(src, dest);// copy to patch's directory
	Patch patch = addPatch(dest);//同loadPatch中的addPatch一样的操作
	if (patch != null) {
		loadPatch(patch);//加载pach，同上loadPatch
	}
}
```

### JNI源码

AndFix分别针对Dalvik虚拟机和Art虚拟机，实现了不同的native方法。

```cpp
//dalvik
extern jboolean dalvik_setup(JNIEnv* env, int apilevel);
extern void dalvik_replaceMethod(JNIEnv* env, jobject src, jobject dest);
extern void dalvik_setFieldFlag(JNIEnv* env, jobject field);
//art
extern jboolean art_setup(JNIEnv* env, int apilevel);
extern void art_replaceMethod(JNIEnv* env, jobject src, jobject dest);
extern void art_setFieldFlag(JNIEnv* env, jobject field);
```

#### setUp

##### Dalvik setUp

```cpp
extern jboolean __attribute__ ((visibility ("hidden"))) dalvik_setup(
		JNIEnv* env, int apilevel) {
	void* dvm_hand = dlopen("libdvm.so", RTLD_NOW);
	if (dvm_hand) {
		//从libdvm.so中获取两个函数的指针：dvmDecodeIndirectRef和dvmThreadSelf。
		dvmDecodeIndirectRef_fnPtr = dvm_dlsym(dvm_hand,
				apilevel > 10 ?
						"_Z20dvmDecodeIndirectRefP6ThreadP8_jobject" :
						"dvmDecodeIndirectRef");
		if (!dvmDecodeIndirectRef_fnPtr) {
			return JNI_FALSE;
		}
		dvmThreadSelf_fnPtr = dvm_dlsym(dvm_hand,
				apilevel > 10 ? "_Z13dvmThreadSelfv" : "dvmThreadSelf");
		if (!dvmThreadSelf_fnPtr) {
			return JNI_FALSE;
		}
		//获得Method类的getDeclaringClass方法ID jClassMethod
		jclass clazz = env->FindClass("java/lang/reflect/Method");
		jClassMethod = env->GetMethodID(clazz, "getDeclaringClass",
						"()Ljava/lang/Class;");

		return JNI_TRUE;
	} else {
		return JNI_FALSE;
	}
}
```
dvmDecodeIndirectRef是libdvm中的方法，它可以从java对象的间接引用获得ClassObject对象。在dalvik_replaceMethod会用到这个方法。

##### Art setUp

```cpp
extern jboolean __attribute__ ((visibility ("hidden"))) art_setup(JNIEnv* env,
		int level) {
	apilevel = level;
	return JNI_TRUE;
}
```
art_setup很简单，只是记录了apilevel。以便后面替换方法的时候选取不同的处理函数。

#### replaceMethod

##### Dalvik replaceMethod

```cpp
extern void __attribute__ ((visibility ("hidden"))) dalvik_replaceMethod(
		JNIEnv* env, jobject src, jobject dest) {
	jobject clazz = env->CallObjectMethod(dest, jClassMethod);
	ClassObject* clz = (ClassObject*) dvmDecodeIndirectRef_fnPtr(
			dvmThreadSelf_fnPtr(), clazz);
	clz->status = CLASS_INITIALIZED;

	Method* meth = (Method*) env->FromReflectedMethod(src);
	Method* target = (Method*) env->FromReflectedMethod(dest);
	LOGD("dalvikMethod: %s", meth->name);

//	meth->clazz = target->clazz;
	meth->accessFlags |= ACC_PUBLIC;
	meth->methodIndex = target->methodIndex;
	meth->jniArgInfo = target->jniArgInfo;
	meth->registersSize = target->registersSize;
	meth->outsSize = target->outsSize;
	meth->insSize = target->insSize;

	meth->prototype = target->prototype;
	meth->insns = target->insns;
	meth->nativeFunc = target->nativeFunc;
}
```

##### Art replaceMethod

Art针对不同的Android版本实现了不同的方法

```cpp
extern void __attribute__ ((visibility ("hidden"))) art_replaceMethod(
		JNIEnv* env, jobject src, jobject dest) {
    if (apilevel > 23) {
        replace_7_0(env, src, dest);
    } else if (apilevel > 22) {
		replace_6_0(env, src, dest);
	} else if (apilevel > 21) {
		replace_5_1(env, src, dest);
	} else if (apilevel > 19) {
		replace_5_0(env, src, dest);
    }else{
        replace_4_4(env, src, dest);
    }
}
```

以6.0为例：

```cpp
void replace_6_0(JNIEnv* env, jobject src, jobject dest) {
	art::mirror::ArtMethod* smeth =
			(art::mirror::ArtMethod*) env->FromReflectedMethod(src);

	art::mirror::ArtMethod* dmeth =
			(art::mirror::ArtMethod*) env->FromReflectedMethod(dest);

    reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->class_loader_ =
    reinterpret_cast<art::mirror::Class*>(smeth->declaring_class_)->class_loader_; //for plugin classloader
    reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->clinit_thread_id_ =
    reinterpret_cast<art::mirror::Class*>(smeth->declaring_class_)->clinit_thread_id_;
    reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->status_ = reinterpret_cast<art::mirror::Class*>(smeth->declaring_class_)->status_-1;
    //for reflection invoke
    reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->super_class_ = 0;

    smeth->declaring_class_ = dmeth->declaring_class_;
    smeth->dex_cache_resolved_methods_ = dmeth->dex_cache_resolved_methods_;
    smeth->dex_cache_resolved_types_ = dmeth->dex_cache_resolved_types_;
    smeth->access_flags_ = dmeth->access_flags_ | 0x0001;
    smeth->dex_code_item_offset_ = dmeth->dex_code_item_offset_;
    smeth->dex_method_index_ = dmeth->dex_method_index_;
    smeth->method_index_ = dmeth->method_index_;
    
    smeth->ptr_sized_fields_.entry_point_from_interpreter_ =
    dmeth->ptr_sized_fields_.entry_point_from_interpreter_;
    
    smeth->ptr_sized_fields_.entry_point_from_jni_ =
    dmeth->ptr_sized_fields_.entry_point_from_jni_;
    smeth->ptr_sized_fields_.entry_point_from_quick_compiled_code_ =
    dmeth->ptr_sized_fields_.entry_point_from_quick_compiled_code_;
    
    LOGD("replace_6_0: %d , %d",
         smeth->ptr_sized_fields_.entry_point_from_quick_compiled_code_,
         dmeth->ptr_sized_fields_.entry_point_from_quick_compiled_code_);
}
```

