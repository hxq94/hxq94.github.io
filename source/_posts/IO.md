---
title: IO 流
thumbnail:  /img/IO-modified.png
cover: /img/IO.jpg
categories: 
- java
- IO
tags: IO
toc: true
---
IO是网络世界中不可获缺的并且很重要的一部分，它是input|output的缩写，有了它，我们才能在网络中传输数据，包括输入和输出
<!--more-->
### 分类
> 输入流：主要有InputStream 和 Reader这两个字节和字符流
    
> 输出流：主要有OutputStream 和 Writer这两个字节和字符流

> 字节流：可以操作任何数据

> 字符流：转么用来操作字符的，比较方便

### 常用的流
流的种类很多，我们只需要知道常用的流，知道他们的原理就好，以后用到其他的再去查阅即可
``` java
// FileInputStream/FileReader 
// FileOutputStream/FileWriter
package com.hyf.demo.io;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;

public class FileInputStreamTest {

    public static void main(String[] args) {

        // 将一个文本写入到一个文件
        String txt = "hello java";
        File file = new File("/Users/huyunfei/Downloads/java.txt");
        try {
            FileOutputStream fileInputStream = new FileOutputStream(file, true);
            fileInputStream.write(txt.getBytes());
        } catch (IOException e) {
            e.printStackTrace();
        }

        // 读取路径中的数据  输出到控制台
        FileInputStream fileInputStream = null;
        try {
            fileInputStream = new FileInputStream(file);
            // 1. 这种方式是一个一个字节读取
            int msg;
            while ((msg = fileInputStream.read()) != -1) {
                System.out.println((char) msg);
            }
            // 2. 这种方式是1024个字节读取
            byte[] bytes = new byte[1024];
            while (fileInputStream.read(bytes) != -1) {
                System.out.println(new String(bytes));
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                fileInputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }


    }
}
```

这个是最基础的输入输出,还有专门用于读取字符的流
```java
package com.hyf.demo.io;

import java.io.*;

public class BufferReaderTest {

    public static void main(String[] args) {

        // 将一个文本写入到一个文件
        String txt = "hello buffer\n";
        File file = new File("/Users/huyunfei/Downloads/java.txt");
        try {
            BufferedOutputStream bufferedOutputStream = new BufferedOutputStream(new FileOutputStream(file));
            bufferedOutputStream.write(txt.getBytes());
            bufferedOutputStream.flush();

            // 字符buffer写入基础代码
//            BufferedWriter bufferedWriter = new BufferedWriter(new FileWriter(file));
//            bufferedWriter.write(txt);
//            bufferedWriter.newLine();
//            bufferedWriter.flush();
        } catch (IOException e) {
            e.printStackTrace();
        }


        // 读取路径中的数据  输出到控制台
        BufferedReader bufferedReader = null;
        try {
            bufferedReader = new BufferedReader(new FileReader(file));
            // 1. 这种方式是一行一行读取
            String msg;
            while ((msg = bufferedReader.readLine()) != null) {
                System.out.println(msg);
            }
            // 2. 这种方式是1024个字节读取 上面读取完毕后 流里面的数据会清空
//            char[] bytes = new char[1024];
//            while (bufferedReader.read(bytes) != -1) {
//                System.out.println(new String(bytes));
//            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                bufferedReader.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }


    }
}
```
上面是按照带buffer的流读取的基础代码
<p>

### 牵扯到的两个问题

#### 为什么要出现带缓冲区的流
#### 使用带缓冲的流包装基础流涉及到的设计模式

> 接着我们来解释第一个问题，为什么要使用带缓冲区的流?

1. 一般来说我们进行频繁数据读写的时候，很耗费性能，于是我们需要将需要读写的数据找到一个地方暂存起来，然后进行一个批量的读写，用的也是顺序写，用来缓解不同设备之间频繁的数据读写，于是有了缓冲区的出现
2. 怎么证明使用了buffer 比不使用buffer 速度快呢？

    1. 源码如下
    
    ```java
     public synchronized void write(byte b[], int off, int len) throws IOException {
        //在这判断需要写的数据长度是否已经超出容器的长度（8192byte）了,如果超出则直接写到相应的outputStream中,并清空缓冲区
        if (len >= buf.length) {
            /* If the request length exceeds the size of the output buffer,
               flush the output buffer and then write the data directly.
               In this way buffered streams will cascade harmlessly. */
            flushBuffer();
            out.write(b, off, len);
            return;
        }
        // 判断缓冲区剩余的容量是否还够写入当前len的内容,如果不够则清空缓冲区
        if (len > buf.length - count) {
            flushBuffer();
        }
        // 将要写的数据先放入内存中,等待数据达到了缓冲区的长度后,再写到相应的outputStream中
        System.arraycopy(b, off, buf, count, len);
        count += len;
    }
    // 他不是每次都调用系统的write 而是积攒到8192byte后才调用一次write 大大减少了系统调用
    ```

    2. 实验如下

    ```java
    import java.io.BufferedOutputStream;
    import java.io.File;
    import java.io.FileOutputStream;

    public class OSFileIO {

        static byte[] data = "123456789\n".getBytes();
        static String path = "/data/io/out.txt";

        public static void main(String[] args) throws Exception {
            switch (args[0]) {
                case "0":
                    testBasicFileIO();
                    break;
                case "1":
                    testBufferedFileIO();
                    break;
                default:
                    break;
            }
        }

        public static void testBasicFileIO() throws Exception {
            File file = new File(path);
            FileOutputStream out = new FileOutputStream(file);
            while (true) {
                out.write(data);
            }
        }

        public static void testBufferedFileIO() throws Exception {
            File file = new File(path);
            BufferedOutputStream out = new BufferedOutputStream(new FileOutputStream(file));
            while (true) {
                out.write(data);
            }
        }
    }
    // 这段代码来源于网络
    ```
    将这段代码上传到linux目录
    ```shell
    /Users/huyunfei/Documents/study/testCase/io
    ```
    然后执行
    ```shell
    # 编译java文件
    javac OSFileIO.java
    # 使用strace 分析这个程序涉及到的系统调用
    strace -ff -o out java OSFileIO $1
    ```
    可以发现最终的结果是，不使用buffer的流，系统调用的write很多，而使用了buffer的达到了8192 byte才会进行一个系统write调用，所以很明显使用buffer后速度更快

> 第二个问题 这里涉及到的设计模式是什么
1. 流中涉及到的设计模式最经典的就是装饰者模式，就是在已有类上进行装饰增强。这样做的优点是可以不改变已有的类，比继承灵活，完全遵循开闭原则；缺点是增加了更多的类，多层装饰增加了代码复杂性

其实还有一些常用的流，比如对象流（可以直接传输对象）、还有我们一般对于文件的读写比较多，但是频繁的文件读写并不好，所以我们可以使用字节数组流，也称为内存流，

### 对象流

```java
package com.hyf.demo.io;

import java.io.*;

public class ObjectIOTest {

    public static void main(String[] args) {
        try {
            // 将一个对象序列化到文件中
            Student student = new Student("libei", 18);
            FileOutputStream fileOutputStream = new FileOutputStream("/Users/huyunfei/Downloads/java.txt");
            ObjectOutputStream writeObjectStream = new ObjectOutputStream(fileOutputStream);
            writeObjectStream.writeObject(student);
            
            // 将文件中对象反序列化出来

            FileInputStream fileInputStream = new FileInputStream("/Users/huyunfei/Downloads/java.txt");
            ObjectInputStream readObjectStream = new ObjectInputStream(fileInputStream);

            Student student1 = (Student) readObjectStream.readObject();
            System.out.println(student1);
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

// 这里一定要实现序列化接口
class Student implements Serializable {

    private String name;
    private Integer age;

    public Student(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
}
```

### 字节数组流

```java
package com.hyf.demo.io;

import java.io.*;

public class ByteArrayInputStreamTest {


    public static void main(String[] args) {
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();

        DataOutputStream dataOutputStream = new DataOutputStream(byteArrayOutputStream);

        try {
            dataOutputStream.writeDouble(Math.random());
            dataOutputStream.writeBoolean(true);
            ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(byteArrayOutputStream.toByteArray());
            DataInputStream dataInputStream = new DataInputStream(byteArrayInputStream);
            System.out.println(dataInputStream.available());
            System.out.println(dataInputStream.readBoolean());
            System.out.println(dataInputStream.readDouble());
            System.out.println(dataInputStream.available());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

常用的流就这些，我们在日常工作用到了查手册就好。

### 选用哪种流
> 记得刚才我们分析过了有一个BufferOutoutStream为什么快，那它和这个内存流ByteOutputStream有什么区别呢？

ByteOutputStream 会每次创建一个32个byte的buffer，每次写入的时候会对比剩余容量是否够用，如果不够用就grow这个buffer 继续写入，一直等数据写完，这些数据都是在内存的
```java
 public synchronized void write(byte b[], int off, int len) {
        if ((off < 0) || (off > b.length) || (len < 0) ||
            ((off + len) - b.length > 0)) {
            throw new IndexOutOfBoundsException();
        }
        // 确保buffer 足够
        ensureCapacity(count + len);
        System.arraycopy(b, off, buf, count, len);
        count += len;
    }
```

所以如果想要快速写入的话，使用ByteArrayOutputStream,如果资源不太够用的时候，应该选择BufferOutputStream

> 下一篇说一下各种IO的区别和背景，以及网络IO的演进