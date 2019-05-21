> 本篇转载至：<https://www.cnblogs.com/xdp-gacl/p/3803033.html>

### 一、会话的概念

　　会话可简单理解为：用户开一个浏览器，点击多个超链接，访问服务器多个web资源，然后关闭浏览器，整个过程称之为一个会话。
　　有状态会话：一个同学来过教室，下次再来教室，我们会知道这个同学曾经来过，这称之为有状态会话。

### 二、会话过程中要解决的一些问题？

　　每个用户在使用浏览器与服务器进行会话的过程中，不可避免各自会产生一些数据，程序要想办法为每个用户保存这些数据。

### 三、保存会话数据的两种技术

#### 3.1、Cookie

　　Cookie是客户端技术，程序把每个用户的数据以cookie的形式写给用户各自的浏览器。当用户使用浏览器再去访问服务器中的web资源时，就会带着各自的数据去。这样，web资源处理的就是用户各自的数据了。

#### 3.2、Session

　　Session是服务器端技术，利用这个技术，服务器在运行时可以为每一个用户的浏览器创建一个其独享的session对象，由于session为用户浏览器独享，所以用户在访问服务器的web资源时，可以把各自的数据放在各自的session中，当用户再去访问服务器中的其它web资源时，其它web资源再从用户各自的session中取出数据为用户服务。

### 四、Cookie使用范例

#### 4.1、使用cookie记录用户上一次访问的时间

```java
package gac.xdp.cookie;

import java.io.IOException;
import java.io.PrintWriter;
import java.util.Date;
import javax.servlet.ServletException;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * @author gacl
 * cookie实例：获取用户上一次访问的时间
 */
public class CookieDemo01 extends HttpServlet {

    public void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        //设置服务器端以UTF-8编码进行输出
        response.setCharacterEncoding("UTF-8");
        //设置浏览器以UTF-8编码进行接收,解决中文乱码问题
        response.setContentType("text/html;charset=UTF-8");
        PrintWriter out = response.getWriter();
        //获取浏览器访问访问服务器时传递过来的cookie数组
        Cookie[] cookies = request.getCookies();
        //如果用户是第一次访问，那么得到的cookies将是null
        if (cookies!=null) {
            out.write("您上次访问的时间是：");
            for (int i = 0; i < cookies.length; i++) {
                Cookie cookie = cookies[i];
                if (cookie.getName().equals("lastAccessTime")) {
                    Long lastAccessTime =Long.parseLong(cookie.getValue());
                    Date date = new Date(lastAccessTime);
                    out.write(date.toLocaleString());
                }
            }
        }else {
            out.write("这是您第一次访问本站！");
        }
        
        //用户访问过之后重新设置用户的访问时间，存储到cookie中，然后发送到客户端浏览器
        Cookie cookie = new Cookie("lastAccessTime", System.currentTimeMillis()+"");//创建一个cookie，cookie的名字是lastAccessTime
        //将cookie对象添加到response对象中，这样服务器在输出response对象中的内容时就会把cookie也输出到客户端浏览器
        response.addCookie(cookie);
    }

    public void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        doGet(request, response);
    }

}
```

第一次访问时这个Servlet时，效果如下所示：

![](<https://github.com/bintoYu/Note/raw/master/picture/%E7%AC%AC%E4%B8%80%E6%AC%A1%E8%AE%BF%E9%97%AEservlet.png>)

点击浏览器的刷新按钮，进行第二次访问，此时就服务器就可以通过cookie获取浏览器上一次访问的时间了。

#### 4.1、给cookie设置有效期

```java
//用户访问过之后重新设置用户的访问时间，存储到cookie中，然后发送到客户端浏览器
Cookie cookie = new Cookie("lastAccessTime", System.currentTimeMillis()+"");//创建一个cookie，cookie的名字是lastAccessTime
//设置Cookie的有效期为1天
cookie.setMaxAge(24*60*60);
//将cookie对象添加到response对象中，这样服务器在输出response对象中的内容时就会把cookie也输出到客户端浏览器
response.addCookie(cookie);
```

这样即使关闭了浏览器，下次再访问时，也依然可以通过cookie获取用户上一次访问的时间。

### 五、Cookie注意细节

1. 一个Cookie只能标识一种信息，它至少含有一个标识该信息的名称（NAME）和设置值（VALUE）。
2. 一个WEB站点可以给一个WEB浏览器发送多个Cookie，一个WEB浏览器也可以存储多个WEB站点提供的Cookie。
3. 浏览器一般只允许存放300个Cookie，每个站点最多存放20个Cookie，每个Cookie的大小限制为4KB。
4. **如果创建了一个cookie，并将他发送到浏览器，默认情况下它是一个会话级别的cookie（即存储在浏览器的内存中），用户退出浏览器之后即被删除。若希望浏览器将该cookie存储在磁盘上，则需要使用maxAge，并给出一个以秒为单位的时间。将最大时效设为0则是命令浏览器删除该cookie。**

#### 5.1、删除Cookie

**注意：删除cookie时，path必须一致，否则不会删除**

```java
package gac.xdp.cookie;

import java.io.IOException;

import javax.servlet.ServletException;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * @author gacl
 * 删除cookie
 */
public class CookieDemo02 extends HttpServlet {

    public void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        //创建一个名字为lastAccessTime的cookie
        Cookie cookie = new Cookie("lastAccessTime", System.currentTimeMillis()+"");
        //将cookie的有效期设置为0，命令浏览器删除该cookie
        cookie.setMaxAge(0);
        response.addCookie(cookie);
    }

    public void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        doGet(request, response);
    }
}
```

#### 5.2、cookie中存取中文

要想在cookie中存储中文，那么必须使用URLEncoder类里面的encode([String](eclipse-javadoc:%E2%98%82=JavaWeb_Cookie_Study_20140715/D:%5C/MyEclipse10%5C/Common%5C/binary%5C/com.sun.java.jdk.win32.x86_1.6.0.013%5C/jre%5C/lib%5C/rt.jar%3Cjava.net(URLEncoder.class%E2%98%83URLEncoder~encode~Ljava.lang.String;~Ljava.lang.String;%E2%98%82String) s, [String](eclipse-javadoc:%E2%98%82=JavaWeb_Cookie_Study_20140715/D:%5C/MyEclipse10%5C/Common%5C/binary%5C/com.sun.java.jdk.win32.x86_1.6.0.013%5C/jre%5C/lib%5C/rt.jar%3Cjava.net(URLEncoder.class%E2%98%83URLEncoder~encode~Ljava.lang.String;~Ljava.lang.String;%E2%98%82String) enc)方法进行中文转码，例如：

```java
Cookie cookie = new Cookie("userName", URLEncoder.encode("孤傲苍狼", "UTF-8"));
response.addCookie(cookie);
```

在获取cookie中的中文数据时，再使用URLDecoder类里面的decode([String](eclipse-javadoc:%E2%98%82=JavaWeb_Cookie_Study_20140715/D:%5C/MyEclipse10%5C/Common%5C/binary%5C/com.sun.java.jdk.win32.x86_1.6.0.013%5C/jre%5C/lib%5C/rt.jar%3Cjava.net(URLDecoder.class%E2%98%83URLDecoder~decode~Ljava.lang.String;~Ljava.lang.String;%E2%98%82String) s, [String](eclipse-javadoc:%E2%98%82=JavaWeb_Cookie_Study_20140715/D:%5C/MyEclipse10%5C/Common%5C/binary%5C/com.sun.java.jdk.win32.x86_1.6.0.013%5C/jre%5C/lib%5C/rt.jar%3Cjava.net(URLDecoder.class%E2%98%83URLDecoder~decode~Ljava.lang.String;~Ljava.lang.String;%E2%98%82String) enc)进行解码，例如：

```java
URLDecoder.decode(cookies[i].getValue(), "UTF-8")
```