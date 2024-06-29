# 介绍

帮助java开发者培养安全编码的习惯。

可以用来训练java代码审计。

# 安装

Windows的jdk版本过高，又不想一直切换，所以还是装在虚拟机里好。

数据库用win的phpstudy启动。注意修改src/main/resources/application.properties的数据库地址。

配置好后在虚拟机里构建jar包：

```text
mvn clean package -DskipTests
```

java -jar javasec-1.11.jar启动

http://192.168.1.124:8888/

还要修改数据库的host

update user set host = '%' where user ='root';

允许远程连接

然后刷新配置

```lua
flush privileges;
```

# SQL注入

## 1.JDBC

漏洞代码 - 语句拼接(Statement)

```java
package SQLi;

import java.sql.*;

public class JDBC {
    //采用Statement方法拼接SQL语句，导致注入产生
    public static String vull(String id) {
        String result = "";
        Connection conn = null;
        Statement stmt = null;
        ResultSet rs = null;
        try {
            // 加载MySQL驱动
            Class.forName("com.mysql.cj.jdbc.Driver");
            // 获取数据库连接
            conn = DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/test?useSSL=false&serverTimezone=UTC", "root", "123456");
            // 创建SQL语句对象
            stmt = conn.createStatement();
            String sql = "select * from users where id = '" + id + "'";
            // 执行查询
            rs = stmt.executeQuery(sql);
            // 处理结果集
            while (rs.next()) {
                result += "ID: " + rs.getString("id") + ", Name: " + rs.getString("user") + "\n";
            }
        } catch (SQLException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return result;
    }

    public static void main(String[] args) {
        System.out.println(vull("0' union select 1,database(),3#"));
    }
}
```

安全代码 - 过滤方法

```java
package SQLi;

import java.sql.*;
import java.util.Locale;

public class JDBC_vul1 {
    //漏洞代码-语句拼接（Statement）
    //采用Statement方法拼接SQL语句，导致注入产生
    public static String vul1(String id) {
        String result = "";
        Connection conn = null;
        Statement stmt = null;
        ResultSet rs = null;
        try {
            // 加载MySQL驱动
            Class.forName("com.mysql.cj.jdbc.Driver");
            // 获取数据库连接
            conn = DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/test?useSSL=false&serverTimezone=UTC", "root", "123456");
            // 创建SQL语句对象
            stmt = conn.createStatement();
            String sql = "select * from users where id = '" + id + "'";
            // 执行查询
            rs = stmt.executeQuery(sql);
            // 处理结果集
            while (rs.next()) {
                result += "ID: " + rs.getString("id") + ", Name: " + rs.getString("user") + "\n";
            }
        } catch (SQLException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return result;
    }

    // 采用黑名单过滤危险字符，同时也容易误伤（次方案）
    public static boolean checkSql(String content){
        String[] blacklist = {"'", ";", "--", "+", ",", "%", "=", ">", "*", "(", ")", "and", "or", "exec", "insert", "select", "delete", "update", "count", "drop", "chr", "mid", "master", "truncate", "char", "declare"};
        for( String s : blacklist) {
            if(content.toLowerCase().contains(s)){
                return true;
            }
        }
        return false;
    }

    public static void main(String[] args) {
        String id = "0' union select 1,database(),3#";
        if (checkSql(id)){
            System.out.println("hacker");
        }
        else {
            System.out.println(vul1(id));
        }
    }
}
```

漏洞代码 - 语句拼接(PrepareStatement)

```java
package Hello_JavaSec.SQLi;

import java.sql.*;

public class JDBC_vul2 {
    // PrepareStatement会对SQL语句进行预编译，但如果直接采取拼接的方式构造SQL，此时进行预编译也无用。
    public static String vul2(String id){
        String result = "";
        Connection conn = null;
        PreparedStatement stmt = null;
        ResultSet rs = null;
        try {
            // 加载MySQL驱动
            Class.forName("com.mysql.cj.jdbc.Driver");
            // 获取数据库连接
            conn = DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/test?useSSL=false&serverTimezone=UTC", "root", "123456");
            String sql = "select * from users where id = '" + id + "'";
            // 创建预编译的SQL语句
            stmt = conn.prepareStatement(sql);
            // 执行查询
            rs = stmt.executeQuery(sql);
            // 处理结果集
            while (rs.next()) {
                result += "ID: " + rs.getString("id") + ", Name: " + rs.getString("user") + "\n";
            }
        } catch (SQLException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return result;
    }
    public static void main(String[] args) {
        String id = "0' union select 1,database(),3#";
        System.out.println(vul2(id));
    }
}
```

output:ID: 1, Name: test

安全代码 - 预编译

```java
package Hello_JavaSec.SQLi;

import java.sql.*;

public class JDBC_vul2 {
    // 正确的使用PrepareStatement可以有效避免SQL注入，使用？作为占位符，进行参数化查询
    public static String vul2(String id){
        String result = "";
        Connection conn = null;
        PreparedStatement stmt = null;
        ResultSet rs = null;
        try {
            // 加载MySQL驱动
            Class.forName("com.mysql.cj.jdbc.Driver");
            // 获取数据库连接
            conn = DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/test?useSSL=false&serverTimezone=UTC", "root", "123456");
            String sql = "select * from users where id = ?";
            // 创建预编译的SQL语句
            stmt = conn.prepareStatement(sql);
            stmt.setString(1,id);
            // 执行查询
            rs = stmt.executeQuery();
            // 处理结果集
            while (rs.next()) {
                result += "ID: " + rs.getString("id") + ", Name: " + rs.getString("user") + "\n";
            }
        } catch (SQLException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return result;
    }
    public static void main(String[] args) {
        String id = "0' union select 1,database(),3#";
        System.out.println(vul2("2"));
    }
}
```

`PreparedStatement` 将参数值作为纯数据处理，不会将其解释为SQL代码，从而防止SQL注入攻击。

漏洞代码 - JdbcTemplate

```java
// JDBCTemplate是Spring对JDBC的封装，如果使用拼接语句便会产生注入

public Map<String, Object> vul3(String id) {
    DriverManagerDataSource dataSource = new DriverManagerDataSource();
    ...
    JdbcTemplate jdbctemplate = new JdbcTemplate(dataSource);

    String sql_vul = "select * from users where id = " + id;
    // 安全语句
    // String sql_safe = "select * from users where id = ?";

    return jdbctemplate.queryForMap(sql_vul);
```

这段代码要Spring JDBC依赖，我没有，就直接粘贴了。

 安全代码 - ESAPI安全框架

```java
// ESAPI 是一个免费、开源的、网页应用程序安全控件库，它使程序员能够更容易写出更低风险的程序
// 官网：https://owasp.org/www-project-enterprise-security-api/

public String safe3(String id) {
    Codec<Character> oracleCodec = new OracleCodec();

    Statement stmt = conn.createStatement();
    String sql = "select * from users where id = '" + ESAPI.encoder().encodeForSQL(oracleCodec, id) + "'";

    ResultSet rs = stmt.executeQuery(sql);
}
```

安全代码 - 强制数据类型

```java
// 如果参数类型为boolean,byte,short,int,long,float,double等，sql语句无法拼接字符串，因此不存在注入

public Map<String, Object> safe4(Integer id) {
    String sql = "select * from users where id = " + id;
    return jdbctemplate.queryForMap(sql);
}
```

## 2.MyBatis

跳过。

# 任意文件操作

## 1.文件上传

漏洞代码

```java
// 允许上传任意文件导致的安全风险

public String singleFileUpload(@RequestParam("file") MultipartFile file, RedirectAttributes redirectAttributes) {
    try {
        byte[] bytes = file.getBytes();
        Path dir = Paths.get(UPLOADED_FOLDER);
        Path path = Paths.get(UPLOADED_FOLDER + file.getOriginalFilename());

        Files.write(path, bytes);
        redirectAttributes.addFlashAttribute("message","上传文件成功：" + path + "");
    } catch (Exception e) {
         return e.toString();
    }
    return "redirect:upload_status";
}
```

安全代码-白名单

```java
// 对上传文件后缀名做白名单处理

public String singleFileUploadSafe(@RequestParam("file") MultipartFile file,
                                   RedirectAttributes redirectAttributes) {
    try {
        byte[] bytes = file.getBytes();
        String fileName = file.getOriginalFilename();
        Path dir = Paths.get(UPLOADED_FOLDER);
        Path path = Paths.get(UPLOADED_FOLDER + file.getOriginalFilename());

        // 获取文件后缀名
        String Suffix = fileName.substring(fileName.lastIndexOf("."));

        String[] SuffixSafe = {".jpg", ".png", ".jpeg", ".gif", ".bmp", ".ico"};
        boolean flag = false;

        for (String s : SuffixSafe) {
            if (Suffix.toLowerCase().equals(s)) {
                flag = true;
                break;
            }
        }

        if (!flag) {
            return "只允许上传图片，[.jpg, .png, .jpeg, .gif, .bmp, .ico]";
        }
```

## 2.目录遍历

漏洞代码

```java
// 文件路径没做限制，通过../递归下载任意文件，如下载../../../../../../../etc/passwd文件

public String download(String filename, HttpServletResponse response) {
    String filePath = System.getProperty("user.dir") + "/logs/" + filename;
    try (InputStream inputStream = new BufferedInputStream(Files.newInputStream(Paths.get(filePath)))) {
        response.setHeader("Content-Disposition", "attachment; filename=" + filename);
        response.setContentLength((int) Files.size(Paths.get(filePath)));
        response.setContentType("application/octet-stream");

        // 使用 Apache Commons IO 库的工具方法将输入流中的数据拷贝到输出流中
        IOUtils.copy(inputStream, response.getOutputStream());
        log.info("文件 {} 下载成功，路径：{}", filename, filePath);
        return "下载文件成功：" + filePath;
    } catch (IOException e) {
        log.error("下载文件失败，路径：{}", filePath, e);
        return "未找到文件：" + filePath;
    }
}
```

安全代码 - 过滤

```java
// 过滤..和/ ，可以防止遍历操作

public String safe(String filename) {
    if (!Security.checkTraversal(filename)) {
        String filePath = System.getProperty("user.dir") + "/logs/" + filename;
        return "安全路径：" + filePath;
    } else {
        return "检测到非法遍历";
    }
}

public static boolean checkTraversal(String content) {
    return content.contains("..") || content.contains("/");
}
```

漏洞代码 - 任意路径遍历

```java
// 攻击者可能会传入 "../../" 来访问项目目录以外的文件

public String fileList(String filename) {
    String filePath = System.getProperty("user.dir") + "/logs/" + filename;
    StringBuilder sb = new StringBuilder();

    File f = new File(filePath);
    File[] fs = f.listFiles();

    if (fs != null) {
        for (File ff : fs) {
            sb.append(ff.getName()).append("<br>");
        }
        return sb.toString();
    }
    return filePath + "目录不存在！";
}
```

# 失效的身份认证

## 1.JWT弱加密

漏洞代码

```java
// 设置了简单的密钥，可爆破从而进行伪造jwt

private static final String SECRET = "123456";
private static final String B64_SECRET = Base64.getEncoder().encodeToString(SECRET.getBytes(StandardCharsets.UTF_8));

public static String generateTokenByJjwt(String userId) {
    return Jwts.builder()
        .setHeaderParam("typ", "JWT")
        .setHeaderParam("alg", "HS256")
        .setIssuedAt(new Date())
        .setExpiration(new Date(System.currentTimeMillis() + EXPIRE))
        .claim("userid", userId)
        .signWith(SignatureAlgorithm.HS256, B64_SECRET)
        .compact();
}
```

安全代码：

设置复杂的密钥

## 2.验证码复用

 漏洞代码

```java
 // 未清除session中的验证码，导致可复用

 if (!CaptchaUtil.ver(captcha, request)) {
     model.addAttribute("msg", "验证码不正确");
     return "login";
 }
```

安全代码

```java
// 不管账号正确与否，一次请求后都清掉验证码

request.getSession().removeAttribute("captcha");
```

# 跨站脚本

## 1.反射型

漏洞代码

```java
// 简单的反射型XSS，没对输出做处理。当攻击者输入恶意js语句时可触发

@GetMapping("/reflect")
public static String input(String content) {
    return content;
}
```

安全代码 -  自定义过滤

```java
// 将特殊字符做转义

private static String XssFilter(String content) {
    content = StringUtils.replace(content, "&", "&amp;");
    content = StringUtils.replace(content, "<", "&lt;");
    content = StringUtils.replace(content, ">", "&gt;");
    content = StringUtils.replace(content, "\", "&quot;");
    content = StringUtils.replace(content, "'", "&#x27;");
    content = StringUtils.replace(content, "/", "&#x2F;");
    return content;
}
```

注：这是HTML字符实体

安全代码 - htmlEscape方法

```java
// 采用Spring自带的方法会对特殊字符全转义

import org.springframework.web.util.HtmlUtils;

@GetMapping("/safe1")
public static String safe1(String content) {
    return HtmlUtils.htmlEscape(content);
}
```

安全代码 - 黑白名单

```java
// 场景：针对富文本的处理方式，需保留部分标签

import org.jsoup.Jsoup;
import org.jsoup.safety.Safelist;

public static String safe3(String content) {
    Safelist whitelist = (new Safelist())
           .addTags("p", "hr", "div", "img", "span", "textarea")  // 设置允许的标签
           .addAttributes("a", "href", "title")          // 设置标签允许的属性, 避免如nmouseover属性
           .addProtocols("img", "src", "http", "https")  // img的src属性只允许http和https开头
           .addProtocols("a", "href", "http", "https");
    return Jsoup.clean(content, whitelist);
}
```

安全代码 - OWASP Java Encoder

```java
// 采用OWASP Java Encoder会对特殊字符全转义

import org.owasp.encoder.Encode;
public static String safe5(String content){
    return Encode.forHtml(content);
}
```

 安全代码 - ESAPI

```java
// ESAPI 是一个免费、开源的、网页应用程序安全控件库，它使程序员能够更容易写出更低风险的程序
// 官网：https://owasp.org/www-project-enterprise-security-api/

import org.owasp.esapi.ESAPI;
public static String safe4(String content){
    return ESAPI.encoder().encodeForHTML(content);
}
```

## 2. 存储型

漏洞代码

```java
public String save(HttpServletRequest request) {
    String content = request.getParameter("content");
    xssMapper.add(content);
    return "success";
 }
```

安全代码

```java
// 方案一、后端输入转义
public String safeStore(HttpServletRequest request) {
    String content = request.getParameter("content");
    String safe_content = HtmlUtils.htmlEscape(content);
    xssMapper.add(safe_content);
    return "success";
}

// 方案二、前端输出转义
$('#xssTableSafe tbody').append('<tr><td>' + item.id + '</td><td>' + $('<div/>').text(item.content).html() + '</td></tr>');
```

# SSRF

SSRF全称：Server-Side Request Forgery，即 服务器端请求伪造。是一个由攻击者构造请求，在目标服务端执行的一个安全漏洞。攻击者可以利用该漏洞使服务器端向攻击者构造的任意域发出请求，目标通常是从外网无法访问的内部系统。简单来说就是利用服务器漏洞以服务器的身份发送一条构造好的请求给服务器所在内网进行攻击。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240604195932.png)

详见：https://xz.aliyun.com/t/12227

漏洞代码

```java
// url参数没做限制，可调用URLConnection发起任意请求，比如请求内网，或使用file等协议读取文件

public static String URLConnection(String url) {
    try {
        URL u = new URL(url);
        URLConnection conn = u.openConnection();
        BufferedReader reader = new BufferedReader(new InputStreamReader(conn.getInputStream()));
        String content;
        StringBuffer html = new StringBuffer();

        while ((content = reader.readLine()) != null) {
            html.append(content);
        }
        reader.close();
        return html.toString();

    } catch (Exception e) {
        return e.getMessage();
    }
}
```

安全代码 - 白名单方式

```java
// 白名单是最佳修复方案
public String URLConnection3(String url) {
    if (!Security.isHttp(url)) {
        return "不允许非http/https协议!!!";
    } else if (!Security.isWhite(url)) {
        return "非可信域名！";
    } else {
        return HttpClientUtil.URLConnection(url);
    }
}
```

漏洞代码 - 绕过

```java
// SSRF修复经常碰到的问题，虽然过滤了内网地址，但通过短链接跳转、IP进制的方式可以绕过

public String URLConnection2(String url) {
    if (!Security.isHttp(url)) {
        return "不允许非http协议!!!";
    } else if (Security.isIntranet(Security.urltoIp(url))) {
        return "不允许访问内网!!!";
    } else {
        return HttpClientUtils.URLConnection(url);
    }
}
```

经测试IP进制不可绕过，猜测原因是`Security.urltoIp` 方法可能会将 IP 地址标准化。

短链接：https://www.985.so/

尝试了一下，需要再次确认才能访问，不知是否可行。

安全代码 - 过滤方式

```java
public String HTTPURLConnection(String url) {
    // 校验 url 是否以 http 或 https 开头
    if (!Security.isHttp(url)) {
        log.error("[HTTPURLConnection] 非法的 url 协议：" + url);
        return "不允许非http/https协议!!!";
    }

    // 解析 url 为 IP 地址
    String ip = Security.urltoIp(url);
    log.info("[HTTPURLConnection] SSRF解析IP：" + ip);

    // 校验 IP 是否为内网地址
    if (Security.isIntranet(ip)) {
        log.error("[HTTPURLConnection] 不允许访问内网：" + ip);
        return "不允许访问内网!!!";
    }

    try {
        return HttpClientUtils.HTTPURLConnection(url);
    } catch (Exception e) {
        log.error("[HTTPURLConnection] 访问失败：" + e.getMessage());
        return "访问失败，请稍后再试!!!";
    }
}

// 1. 判断是否为http协议
public static boolean isHttp(String url) {
    return url.startsWith("http://") || url.startsWith("https://");
}

// 2. 解析IP地址
public static String urltoIp(String url) {
    try {
        URI uri = new URI(url);
        String host = uri.getHost().toLowerCase();
        InetAddress ip = Inet4Address.getByName(host);
        return ip.getHostAddress();
    } catch (Exception e) {
        return "127.0.0.1";
    }
}

// 3. 判断是否为内网IP
public static boolean isIntranet(String url) {
    Pattern reg = Pattern.compile("^(127\\.0\\.0\\.1)|(localhost)|^(10\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3})|^(172\\.((1[6-9])|(2\\d)|(3[01]))\\.\\d{1,3}\\.\\d{1,3})|^(192\\.168\\.\\d{1,3}\\.\\d{1,3})$");
    Matcher match = reg.matcher(url);
    Boolean a = match.find();
    return a;
}

// 4. 不允许302跳转
HttpURLConnection conn = (HttpURLConnection) u.openConnection();
conn.setInstanceFollowRedirects(false); // 不允许重定向或者对重定向后的地址做二次判断
conn.connect();
                    
```

# 远程代码执行

漏洞代码 - ProcessBuilder方式

```java
// new ProcessBuilder(command).start()
// 功能是利用ProcessBuilder执行ls命令查看文件，但攻击者通过拼接; & |等连接符来执行自己的命令。

public static String processbuilderVul(String filepath) throws IOException {
    String[] cmdList = {"sh", "-c", "ls -l " + filepath};
    ProcessBuilder pb = new ProcessBuilder(cmdList);
    pb.redirectErrorStream(true);
    Process process = pb.start();

    // 获取命令的输出
    InputStream inputStream = process.getInputStream();
    BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
    String line;
    StringBuilder output = new StringBuilder();
    while ((line = reader.readLine()) != null) {
        output.append(line).append("\n");
    }
    return output.toString();
}
```

安全代码 - 过滤方式

```java
// 自定义黑名单，这里过滤了常见的管道符，可自行添加

public static boolean checkOs(String content) {
    String[] black_list = {"|", ",", "&", "&&", ";", "||"};
    for (String s : black_list) {
        if (content.contains(s)) {
            return true;
        }
    }
    return false;
}
```

漏洞代码 - Runtime方式

```java
// Runtime.getRuntime().exec(cmd)

public static String vul(String cmd) {
    StringBuilder sb = new StringBuilder();
    try {
        Process proc = Runtime.getRuntime().exec(cmd);
        InputStream fis = proc.getInputStream();
        InputStreamReader isr = new InputStreamReader(fis);
        BufferedReader br = new BufferedReader(isr);
     ...
```

安全代码 - 白名单方式

```java
// 使用白名单替换黑名单。黑名单需要不断更新，而白名单只需要指定允许执行的命令，更容易维护。

public static String safe(String cmd) {
    // 定义命令白名单
    Set<String> commands = new HashSet<\>();
    commands.add("ls");
    commands.add("pwd");

    // 检查用户提供的命令是否在白名单中
    String command = cmd.split("\\s+")[0];
    if (!commands.contains(command)) {
        return "命令不在白名单中";
    }
    ...
}
```

漏洞代码 - ProcessImpl

```java
// ProcessImpl 是更为底层的实现，Runtime和ProcessBuilder执行命令实际上也是调用了ProcessImpl这个类
// ProcessImpl 类是一个抽象类不能直接调用，但可以通过反射来间接调用ProcessImpl来达到执行命令的目的

public static String vul(String cmd) throws Exception {
    // 首先，使用 Class.forName 方法来获取 ProcessImpl 类的类对象
    Class clazz = Class.forName("java.lang.ProcessImpl");

    // 然后，使用 clazz.getDeclaredMethod 方法来获取 ProcessImpl 类的 start 方法
    Method method = clazz.getDeclaredMethod("start", String[].class, Map.class, String.class, ProcessBuilder.Redirect[].class, boolean.class);

    // 使用 method.setAccessible 方法将 start 方法设为可访问
    method.setAccessible(true);

    // 最后，使用 method.invoke 方法来调用 start 方法，并传入参数 cmd，执行命令
    Process process = (Process) method.invoke(null, new String[]{cmd}, null, null, null, false);
}
```

这个之前学Java反序列化了解过，有时间得复习一下。

漏洞代码 - 脚本引擎代码注入

```java
// 通过加载远程js文件来执行代码，如果加载了恶意js则会造成任意命令执行
// 远程恶意js: var a = mainOutput(); function mainOutput() { var x=java.lang.Runtime.getRuntime().exec("open -a Calculator");}
// ⚠️ 在Java 8之后移除了ScriptEngineManager的eval

public void jsEngine(String url) throws Exception {
    ScriptEngine engine = new ScriptEngineManager().getEngineByName("JavaScript");
    Bindings bindings = engine.getBindings(ScriptContext.ENGINE_SCOPE);
    String payload = String.format("load('%s')", url);
    engine.eval(payload, bindings);
}
```

漏洞代码 - Groovy执行命令

```java
// 不安全的使用Groovy调用命令

import groovy.lang.GroovyShell;
@GetMapping("/groovy")
public void groovy(String cmd) {
    GroovyShell shell = new GroovyShell();
    shell.evaluate(cmd);
}
```

# 反序列化漏洞

漏洞代码 - ObjectInputStream

```java
// readObject，读取输入流,并转换对象。ObjectInputStream.readObject() 方法的作用正是从一个源输入流中读取字节序列，再把它们反序列化为一个对象。
// 生成payload：java -jar ysoserial-0.0.6-SNAPSHOT-BETA-all.jar CommonsCollections5 "open -a Calculator" | base64

public String cc(String base64) {
    try {
        base64 = base64.replace(" ", "+");
        byte[] bytes = Base64.getDecoder().decode(base64);

        ByteArrayInputStream stream = new ByteArrayInputStream(bytes);

        // 反序列化流，将序列化的原始数据恢复为对象
        ObjectInputStream in = new ObjectInputStream(stream);
        in.readObject();
        in.close();
        return "反序列化漏洞";
    } catch (Exception e) {
        return e.toString();
    }
}
```

安全代码 - 黑白名单

```java
// 使用Apache Commons IO的ValidatingObjectInputStream，accept方法来实现反序列化类白/黑名单控制

public String safe(String base64) {
    try {
        base64 = base64.replace(" ", "+");
        byte[] bytes = Base64.getDecoder().decode(base64);

        ByteArrayInputStream stream = new ByteArrayInputStream(bytes);
        ValidatingObjectInputStream ois = new ValidatingObjectInputStream(stream);

        // 只允许反序列化Student class
        ois.accept(Student.class);
        ois.readObject();

        return "ValidatingObjectInputStream";
     } catch (Exception e) {
        return e.toString();
}
```

漏洞代码 - SnakeYaml

```java
// 远程服务器支持用户可以输入yaml格式的内容并且进行数据解析，没有做沙箱，黑名单之类的防控

public void yaml(String content) {
    Yaml y = new Yaml();
    y.load(content);
}
```

没见过，暂时跳过

安全代码 - SnakeYaml

```java
// SafeConstructor 是 SnakeYaml 提供的一个安全的构造器。它可以用来构造安全的对象，避免反序列化漏洞的发生。

public void safe(String content) {
    Yaml y = new Yaml(new SafeConstructor());
    y.load(content);
    log.info("[safe] SnakeYaml反序列化: " + content);
}
```

漏洞代码 - XMLDecoder

```java
// XMLDecoder在JDK 1.4~JDK 11中都存在反序列化漏洞安全风险。攻击者可以通过此漏洞远程执行恶意代码来入侵服务器。在项目中应禁止使用XMLDecoder方式解析XML内容

String path = "src/main/resources/payload/calc-1.xml";
File file = new File(path);
FileInputStream fis = null;
try {
    fis = new FileInputStream(file);
} catch (Exception e) {
    e.printStackTrace();
}

BufferedInputStream bis = new BufferedInputStream(fis);
XMLDecoder xmlDecoder = new XMLDecoder(bis);
xmlDecoder.readObject();
xmlDecoder.close();
```

# 表达式注入

漏洞代码

```java
// PoC: T(java.lang.Runtime).getRuntime().exec(%22open%20-a%20Calculator%22)

public String vul(String ex) {
    ExpressionParser parser = new SpelExpressionParser();

    // StandardEvaluationContext权限过大，可以执行任意代码，默认使用
    EvaluationContext evaluationContext = new StandardEvaluationContext();

    Expression exp = parser.parseExpression(ex);
    String result = exp.getValue(evaluationContext).toString();
    return result;
}
```

安全代码

```java
// SimpleEvaluationContext 旨在仅支持 SpEL 语言语法的一个子集。它不包括 Java 类型引用，构造函数和 bean 引用

public String spelSafe(String ex) {
    ExpressionParser parser = new SpelExpressionParser();

    EvaluationContext simpleContext = SimpleEvaluationContext.forReadOnlyDataBinding().build();

    Expression exp = parser.parseExpression(ex);
    String result = exp.getValue(simpleContext).toString();
    return result;
}
```

漏洞代码 - 黑名单绕过

```java
// 黑名单正则，尝试绕过执行恶意代码

public String vul2(String ex) {
    String[] black_list = {"java.+lang", "Runtime", "exec.*\\("};
    for (String s : black_list) {
        Matcher matcher = Pattern.compile(s).matcher(ex);
        if (matcher.find()) {
            return "黑名单过滤";
        }
    }

    ExpressionParser parser = new SpelExpressionParser();
    Expression exp = parser.parseExpression(ex);
    String result = exp.getValue().toString();
    return result;
}
```

# XML外部实体

xxe漏洞：https://zhuanlan.zhihu.com/p/700860965

漏洞代码 - XMLReader

```java
public String XMLReader(@RequestBody String content) {
    try {
        XMLReader xmlReader = XMLReaderFactory.createXMLReader();
        // 修复：禁用外部实体
        // xmlReader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
        xmlReader.parse(new InputSource(new StringReader(content)));
        return "XMLReader XXE";
    } catch (Exception e) {
        return e.toString();
    }
}
```

漏洞代码 - DocumentBuilder

```java
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
// 修复: 禁用外部实体
// factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
DocumentBuilder builder = factory.newDocumentBuilder();
```

漏洞代码 - SAXReader

```java
SAXReader sax = new SAXReader();
// 修复：禁用外部实体
// sax.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
sax.read(new InputSource(new StringReader(content)));
```

漏洞代码 - Unmarshaller

```java
/**
  *  PoC
  * Content-Type: application/xml
  * <?xml version="1.0" encoding="UTF-8"?><!DOCTYPE student[<!ENTITY out SYSTEM "file:///etc/hosts">]><student><name>&out;</name></student>
  */
public String Unmarshaller(@RequestBody String content) {
    try {
        JAXBContext context = JAXBContext.newInstance(Student.class);
        Unmarshaller unmarshaller = context.createUnmarshaller();

        XMLInputFactory xif = XMLInputFactory.newFactory();
        // 修复: 禁用外部实体
        // xif.setProperty(XMLConstants.ACCESS_EXTERNAL_DTD, "");
        // xif.setProperty(XMLConstants.ACCESS_EXTERNAL_STYLESHEET, "");

        XMLStreamReader xsr = xif.createXMLStreamReader(new StringReader(content));

        Object o = unmarshaller.unmarshal(xsr);
        return o.toString();

} catch (Exception e) {
    e.printStackTrace();
}
```

漏洞代码 - SAXBuilder

```java
@RequestMapping(value = "/SAXBuilder")
public String SAXBuilder(@RequestBody String content) {
    try {
        SAXBuilder saxbuilder = new SAXBuilder();
        // 修复: saxbuilder.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
        saxbuilder.build(new InputSource(new StringReader(content)));
        return "SAXBuilder XXE";
    } catch (Exception e) {
        return e.toString();
    }
}
```

安全代码 - 黑名单过滤

```java
public static boolean checkXXE(String content) {
    String[] black_list = {"ENTITY", "DOCTYPE"};
    for (String s : black_list) {
        if (content.toUpperCase().contains(s)) {
            return true;
        }
    }
    return false;
}
```

