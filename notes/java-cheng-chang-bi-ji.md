# java 成长笔记

## 目的

C 可以说是老手啦，但是java用的其实不多，但是非常适合偷懒，所以把日常的一些技巧和疑惑放在这里做一个汇总。

## Here we go

### 数组转字符串

```java
		int [] tap = {1,2,3,4}

		StringBuilder sb = new StringBuilder();
		for(int i : tap) {
			sb.append(i);
			sb.append(",");
		}
		sb.deleteCharAt(sb.length()-1);
		System.out.println(sb.toString());
		
		
```

### 字符串转数组

```java
        String delay = "1,2,3,4";
        String [] delays = delay.split(","); 
        int [] tapDelay;
        tapDelay = new int[delays.length];
        for( String tmp : delays ) {
			tapDelay[i++] = Integer.parseInt(tmp);
		}
		
		
```

### 重定向输入输出

```java
		int length = 256;
		PrintStream out = System.out;
		OutputStream tmpStream = new ByteArrayOutputStream(length);
		System.setOut(new PrintStream(tmpStream));
		System.out.println("something");	
        System.setOut(out);
        System.out.println(tmpStream.toString());
```

### 获取系统属性和环境变量

```java
        Map<String, String> map = System.getenv();
        for(Iterator<String> itr = map.keySet().iterator();itr.hasNext();){
            String key = itr.next();
            System.out.println(key + "=" + map.get(key));
        } 
        
        String tmpdir = System.getProperty("java.io.tmpdir");
        
        更多请看手册哦~
```

### 动态库的问题

```java
System.loadLibrary() 加载的动态库必须在 Path环境变量的路径下 
String libdir = System.getProperty("java.library.path");
System.load() 可以解决这个问题，但是得给出绝对路径

最后是 dll 的生成必须符合 JNI 规范，自己找资料0.0
编译dll得使用java提供的特定头文件

但是JNA很好的解决了这个问题，但是JNA的问题是
JNA加载DLL的路径会改变，因为它绕过了JVM
和JVM是同等地位，看WINIO那篇文章可以知道
```

### 获取jar包的文件

```java
jar包内所有的文件都属于class，需要采用非正常方法加载，参考的是 Jinteltype的源码
  void fromJarToFs(String jarPath, String filePath) throws IOException {
      InputStream is = null;
      OutputStream os = null;
      try {
         File file = new File(filePath);
         if (file.exists()) {
            boolean success = file.delete();
            if (!success) {
               throw new IOException("Could not delete file: " + filePath);
            }
         }

         is = ClassLoader.getSystemClassLoader().getResourceAsStream(jarPath);
         os = new FileOutputStream(filePath);
         byte[] buffer = new byte[8192];
         int bytesRead;
         while ((bytesRead = is.read(buffer)) != -1) {
            os.write(buffer, 0, bytesRead);
         }
      } catch (Exception ex) {
         throw new IOException("FromJarToFileSystem failed " + jarPath, ex);
      } finally {
         if (is != null) {
            is.close();
         }
         if (os != null) {
            os.close();
         }
      }
```

