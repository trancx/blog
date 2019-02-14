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

## 获取系统属性和环境变量

```java
        Map<String, String> map = System.getenv();
        for(Iterator<String> itr = map.keySet().iterator();itr.hasNext();){
            String key = itr.next();
            System.out.println(key + "=" + map.get(key));
        } 
        
        String tmpdir = System.getProperty("java.io.tmpdir");
        
        更多请看手册哦~
```

