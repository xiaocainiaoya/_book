

#### SpringBoot的jar包如何启动

@[toc]
#### 一、简介

​	使用过`SprongBoot`打过`jar`包的都应该知道，目标文件一般都会生成两个文件，一个是以`.jar`的包，一个是`.jar.original`文件。那么使用`SpringBoot`会打出两个包，而`.jar.original`的作用是什么呢？还有就是`java -jar`是如何将一个`SpringBoot`项目启动，之间都进行了那些的操作？

​	其实`.jar.original`是`maven`在`SpringBoot`重新打包之前的原始`jar`包，内部只包含了项目的用户类，不包含其他的依赖`jar`包，生成之后，`SpringBoot`重新打包之后，最后生成`.jar`包，内部包含了原始`jar`包以及其他的引用依赖。以下提及的`jar`包都是`SpringBoot`二次加工打的包。

#### 二、jar包的内部结构

> `SpringBoot`打出的`jar`包，可以直接通过解压的方式查看内部的构造。一般情况下有三个目录。

+ `BOOT-INF`：这个文件夹下有两个文件夹`classes`用来存放用户类，也就是原始`jar.original`里的类；还有一个是`lib`，就是这个原始`jar.original`引用的依赖。
+ `META-INF`：这里是通过`java -jar`启动的入口信息，记录了入口类的位置等信息。
+ `org`:`Springboot loader`的代码，通过它来启动。

**这里主要介绍一下`/BOOT-INF/MANIFEST.MF`文件**

```properties
Manifest-Version: 1.0
Implementation-Title: springboot-server
Implementation-Version: 0.0.1-SNAPSHOT
Archiver-Version: Plexus Archiver
Built-By: Administrator
Implementation-Vendor-Id: cn.com.springboot
Spring-Boot-Version: 1.5.13.RELEASE
Implementation-Vendor: Pivotal Software, Inc.
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: cn.com.springboot.center.AuthEenterBootstrap
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Created-By: Apache Maven 3.6.1
Build-Jdk: 1.8.0_241
Implementation-URL: http://projects.spring.io/spring-boot/auth-server/
```

**`Main-Class`**：记录了`java -jar`的启动入口，当使用该命令启动时就会调用这个入口类的`main`方法，显然可以看出，`Springboot`转移了启动的入口，不是用户编写的`xxx.xxx.BootStrap`的那个入口类。

**`Start-Class`**：记录了用户编写的`xxx.xxx.BootStrap`的那个入口类，当内嵌的`jar`包加载完成之后，会使用`LaunchedURLClassLoader`线程加载类来加载这个用户编写的入口类。

#### 三、加载过程

##### 1.使用到的一些类

**3.1.1 Archive**

​	归档文件接口，实现迭代器接口，它有两个子类，一个是`JarFileArchive`对`jar`包文件使用，提供了返回这个`jar`文件对应的`url`、或者这个`jar`文件的`MANIFEST`文件数据信息等操作。是`ExplodedArchive`是文件目录的使用也有获取这个目录`url`的方法，以及获取这个目录下的所有`Archive`文件方法。

![](https://img-blog.csdnimg.cn/img_convert/1faa8d1fb220b5b1b466cd0b27d62e30.png)

**3.1.2 Launcher**

​	启动程序的基类，这边最后是通过`JarLauncher#main()`来启动。`ExecutableArchiveLauncher`是抽象类，提供了获取`Start-Class`类路径的方法，以及是否还有内嵌对应文件的判断方法和获取到内嵌对应文件集合的后置处理方法的抽象，由子类`JarLauncher`和`WarLauncher`自行实现。

![](https://img-blog.csdnimg.cn/img_convert/d5ef4fccc5146602dccccc0c53b4e074.png)

**3.1.3 Spring.loader下的JarFile和JarEntry**

​	`jarFile`继承于`jar.util.jar.JarFile`，`JarEntry`继承于`java.util.jar.JarEntry`，对原始的一些方法进行重写覆盖。每一个`JarFileArchive`都拥有一个`JarFile`方法，用于存储这个`jar`包对应的文件，而每一个`JarFile`都有一个`JarFileEntries`,`JarFileEntries`是一个迭代器。总的来说，在解析`jar`包时，会将`jar`包内的文件封装成`JarEntry`对象后由`JarFile`对象保存文件列表的迭代器。所以`JarFileArchive`和`JarFileEntries`之间是通过`JarFile`连接，二者都可以获取到`JarFile`对象。

##### 2.过程分析

从`MANIFEST.MF`文件中的`Main-class`指向入口开始。

创建`JarLauncher`并且通过它的`launch()`方法开始加载`jar`包内部信息。

```java
public static void main(String[] args) throws Exception {
  new JarLauncher().launch(args);
}
```

`JarLauncher`的空构造方法时一个空实现，刚开始看的时候还懵了一下，以为是在后续的操作中去加载的文件，其实不然，在创建时由父类`ExecutableArchiveLauncher`的构造方法去加载的文件。

```java
public ExecutableArchiveLauncher() {
  try {
    // 加载为归档文件对象
    this.archive = createArchive();
  }
  catch (Exception ex) {
    throw new IllegalStateException(ex);
  }
}


/**
	 * 具体的加载方法
	 */
protected final Archive createArchive() throws Exception {
  ProtectionDomain protectionDomain = getClass().getProtectionDomain();
  CodeSource codeSource = protectionDomain.getCodeSource();
  URI location = (codeSource == null ? null : codeSource.getLocation().toURI());
  String path = (location == null ? null : location.getSchemeSpecificPart());
  if (path == null) {
    throw new IllegalStateException("Unable to determine code source archive");
  }
  File root = new File(path);
  if (!root.exists()) {
    throw new IllegalStateException(
      "Unable to determine code source archive from " + root);
  }
  // 判断路径是否是一个文件夹，是则返回ExplodedArchive对象，否则返回JarFileArchive
  return (root.isDirectory() ? new ExplodedArchive(root): new JarFileArchive(root));
}

// ============== JarFileArchive =============

public class JarFileArchive implements Archive {

  public JarFileArchive(File file) throws IOException {
    this(file, null);
  }

  public JarFileArchive(File file, URL url) throws IOException {
    // 通过这个new方法创建JarFile对象
    this(new JarFile(file));
    this.url = url;
  }
}

// ============== JarFile =============

public class JarFile extends java.util.jar.JarFile {

  public JarFile(File file) throws IOException {
    // 通过RandomAccessDataFile读取文件信息
    this(new RandomAccessDataFile(file));
  }
}
```

`jarLauncher#launch()`方法：

```java
protected void launch(String[] args) throws Exception {
  // 注册URL协议的处理器，没有指定时，默认指向org.springframework.boot.loader包路径
  JarFile.registerUrlProtocolHandler();
  // 获取类路径下的归档文件Archive并通过这些归档文件的URL，创建线程上下文类加载器LaunchedURLClassLoader
  ClassLoader classLoader = createClassLoader(getClassPathArchives());
  // 使用类加载器和用户编写的启动入口类，通过反射调用它的main方法。
  launch(args, getMainClass(), classLoader);
}
```

`JarLauncher`的`getClassPathArchives()`是在`ExecutableArchiveLauncher`中实现:

```java
@Override
protected List<Archive> getClassPathArchives() throws Exception {
  List<Archive> archives = new ArrayList<Archive>(
    // 获取归档文件中满足EntryFilterg过滤器的项，isNestedArchive()方法由具体
    // 的之类实现。
    this.archive.getNestedArchives(new EntryFilter() {

      @Override
      public boolean matches(Entry entry) {
        return isNestedArchive(entry);
      }

    }));
  // 获取到当前归档文件下的所有子归档文件之后的后置操作，是一个扩展点。在JarLauncher
  // 中是一个空实现。
  postProcessClassPathArchives(archives);
  return archives;
}

/**
 * JarLauncher的具体实现，这里通过判断是否在BOOT-INF/lib/包下返回true
 * 也就是说只会把jar包下的BOOT-INF/lib/下的文件加载为Archive对象
 */
@Override
protected boolean isNestedArchive(Archive.Entry entry) {
  if (entry.isDirectory()) {
    return entry.getName().equals(BOOT_INF_CLASSES);
  }
  return entry.getName().startsWith(BOOT_INF_LIB);
}
```

`JarFileArchive`的`getNestedArchives`方法

```java
@Override
public List<Archive> getNestedArchives(EntryFilter filter) throws IOException {
	List<Archive> nestedArchives = new ArrayList<Archive>();
		for (Entry entry : this) {
		  // 若匹配器匹配获取内嵌归档文件
			if (filter.matches(entry)) {
				nestedArchives.add(getNestedArchive(entry));
			}
		}
	return Collections.unmodifiableList(nestedArchives);
}

protected Archive getNestedArchive(Entry entry) throws IOException {
	JarEntry jarEntry = ((JarFileEntry) entry).getJarEntry();
	if (jarEntry.getComment().startsWith(UNPACK_MARKER)) {
		return getUnpackedNestedArchive(jarEntry);
	}
	try {
		// 根据具体的Entry对象，创建JarFile对象
		JarFile jarFile = this.jarFile.getNestedJarFile(jarEntry);
		// 封装成归档文件对象后返回
		return new JarFileArchive(jarFile);
	}
	catch (Exception ex) {
		throw new IllegalStateException("Failed to get nested  entry"+entry.getName(),ex);
	}
}

public synchronized JarFile getNestedJarFile(final ZipEntry entry)throws IOException {
	return getNestedJarFile((JarEntry) entry);
}

public synchronized JarFile getNestedJarFile(JarEntry entry) throws IOException {
	try {
		 // 根据具体的Entry对象，创建JarFile对象
		 return createJarFileFromEntry(entry);
	}
	catch (Exception ex) {
		throw new IOException( "Unable to open nested jar file'"+entry.getName()+"'",ex);
	}
}

private JarFile createJarFileFromEntry(JarEntry entry) throws IOException {
	if (entry.isDirectory()) {
		return createJarFileFromDirectoryEntry(entry);
	}
	return createJarFileFromFileEntry(entry);
}

private JarFile createJarFileFromFileEntry(JarEntry entry) throws IOException {
	if (entry.getMethod() != ZipEntry.STORED) {
	  throw new IllegalStateException("Unable to open nested entry '"
	  + entry.getName() + "'. It has been compressed and nested "
	  + "jar files must be stored without compression. Please check the "
	   + "mechanism used to create your executable jar file");
	}
	// 获取到参数entry对应的RandomAccessData对象
	RandomAccessData entryData = this.entries.getEntryData(entry.getName());
	// 这里根据springboot扩展的url协议，在父路径的基础上添加!/来标记子包
	return new JarFile(this.rootFile, this.pathFromRoot + "!/" + entry.getName(),
	                   entryData, JarFileType.NESTED_JAR);
}
```

到这基本上读取`jar`内部信息，加载为对应归档文件对象的大概过程已经讲完了，接下来分析一下在获取到了整个`jar`的归档文件对象后的处理。

```java

/**
 * 通过归档文件对象列表，获取对应的url信息，并通过url信息创建LaunchedURLClassLoader
 */
protected ClassLoader createClassLoader(List<Archive> archives) throws Exception {
	List<URL> urls = new ArrayList<URL>(archives.size());
	for (Archive archive : archives) {
		urls.add(archive.getUrl());
	}
	return createClassLoader(urls.toArray(new URL[urls.size()]));
}

protected ClassLoader createClassLoader(URL[] urls) throws Exception {
	return new LaunchedURLClassLoader(urls, getClass().getClassLoader());
}
```

获取到对应的`LaunchedUrlClassLoader`类加载器之后，设置线程的上下文类加载器为该加载器。

```java
protected void launch(String[] args, String mainClass, ClassLoader classLoader) throws Exception {
	Thread.currentThread().setContextClassLoader(classLoader);
	// 根据MANIFI.MF文件中的start-classs信息创建项目启动入口主类对象，并通过run方法启动
	createMainMethodRunner(mainClass, args, classLoader).run();
}

// =========== MainMethodRunner ================
public class MainMethodRunner {

	private final String mainClassName;

	private final String[] args;

	public MainMethodRunner(String mainClass, String[] args) {
		this.mainClassName = mainClass;
		this.args = (args == null ? null : args.clone());
	}

	public void run() throws Exception {
		Class<?> mainClass = Thread.currentThread().getContextClassLoader()
				.loadClass(this.mainClassName);
    // 通过反射调用启动项目启动类的main方法
		Method mainMethod = mainClass.getDeclaredMethod("main", String[].class);
		mainMethod.invoke(null, new Object[] { this.args });
	}
}
```

最后来说一下这个`LaunchedURLClassLoader`，它继承于`URLClassLoader`，并重写了`loadClass`方法

`LaunchedClassLoader`的`loadClass`方法

```java
@Override
protected Class<?> loadClass(String name, boolean resolve)throws ClassNotFoundException {
   Handler.setUseFastConnectionExceptions(true);
   try {
      try {
         definePackageIfNecessary(name);
      }
      catch (IllegalArgumentException ex) {
         if (getPackage(name) == null) {
            throw new AssertionError("Package " + name + " has already been "
                  + "defined but it could not be found");
         }
      }
     // 调用父类loadClass方法，走正常委派流程，最终会被LaunchURLClassLoader
      return super.loadClass(name, resolve);
   }
   finally {
      Handler.setUseFastConnectionExceptions(false);
   }
}

// ============URLClassLoader =====================

protected Class<?> findClass(final String name)throws ClassNotFoundException
{
  final Class<?> result;
  try {
    result = AccessController.doPrivileged(
      new PrivilegedExceptionAction<Class<?>>() {
        public Class<?> run() throws ClassNotFoundException {
          // 根据name，将路径转化为以.class结尾的/分隔的格式。
          String path = name.replace('.', '/').concat(".class");
          // 通过UrlClassPath对象根据路径获取资源类文件
          Resource res = ucp.getResource(path, false);
          if (res != null) {
            try {
              return defineClass(name, res);
            } catch (IOException e) {
              throw new ClassNotFoundException(name, e);
            }
          } else {
            return null;
          }
        }
      }, acc);
  } catch (java.security.PrivilegedActionException pae) {
    throw (ClassNotFoundException) pae.getException();
  }
  if (result == null) {
    throw new ClassNotFoundException(name);
  }
  return result;
}
```

#### 四、总结

​	`Springboot`主要实现了对`URL`加载方式进行了扩展，并且对一些对象`Archive`、`JarFile`、`Entry`等进行了抽象和扩展，最后使用`LaunchedUrlClassLoader`来进行处理。



​	




















