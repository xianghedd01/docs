### 如何生成一个可信的 Cookie

因为 Cookie 都是服务器端创建的，所以，生成一个可信 Cookie 的关键在于，客户端无法伪造出 Cookie。
用什么方法可以防止伪造？数学理论告诉我们，单向函数就可以防伪造。
例如，计算 md5 就是一个单向函数。假设写好了函数 md5(String s)，根据输入可以很容易地计算结果：
`md5("hello") => "b1946ac92492d2347c6235b4d2611184"`
但是，根据结果"b1946...11184"反推输入却非常困难。
利用单向函数，我们可以生成一个防伪造的 Cookie。
例如，用户以用户名"admin"，口令"hello"登录成功后，要生成 Cookie，我们就可以用 md5 计算：
`md5("hello") => "b1946ac92492d2347c6235b4d2611184"`
然后，把 md5 值和用户名"admin"串起来构成一个 Cookie 发送给客户端：
`"admin:b1946ac92492d2347c6235b4d2611184"`

当客户端把上面的 Cookie 发给服务器时，服务器如何验证该 Cookie 是有效的呢？可以按照以下步骤：

1. 服务器把 Cookie 分解成用户名"admin"和 md5 值"b1946...11184"；
2. 根据用户名"admin"从数据库中找到该用户的记录，并继续找到该用户的口令"hello"；
3. 服务器根据数据库中存储的口令计算 md5("hello")并与客户端 Cookie 的 md5 值对比。

如果对比一致，说明 Cookie 是有效的。
现在可以愉快地为用户创建 Cookie 了！
且慢！

从理论到实践还差着一个工程的距离。上面的算法仅仅解决了基本的验证，在实际应用中，存在如下严重问题：

1. 简单的 md5 值很容易被彩虹表攻击，从而直接得到用户原始口令；
2. 用户名被暴露在 Cookie 中，如果用 email 作为用户名，用户的 email 就被泄露了；
3. Cookie 没有设置有效期（注意浏览器发过来的 Cookie 不一定真是浏览器发的），导致一旦登录，永久有效；
4. 其他若干问题。

如何解决？方法是计算 hash 的时候，不仅只包含用户口令，还包含 Cookie 过期时间，以及其他相关随机数，这样计算的 hash 就非常安全。
举个栗子：
假设用户仍以用户名"admin"，口令"hello"登录成功，系统可以知道：

1. 该用户的 id，例如，1230001；
2. 该用户的口令，例如，"hello"；
3. Cookie 过期时间，可由当前时间戳＋固定时长计算，例如，1461288165；
4. 系统固定的一个随机字符串，例如，"secret"。

把上面 4 部分拼起来，得到：
`"1230001:hello:1461288165:secret"`
计算上述字符串的 md5，得到：`"d9753...004d5"`。
最后，按照用户 id，过期时间和最终的 hash 值，拼接得到 Cookie 如下：
`"1230001:1461288165:d9753...004d5"`

当浏览器发送 Cookie 回服务器时，我们就可以按照下面的方式验证 Cookie：

1. 把 Cookie 分割成三部分，得到用户 id，过期时间和 hash 值；
2. 如果过期时间已到，直接丢弃；
3. 根据用户 id 查找用户，得到用户口令；
4. 按照生成 Cookie 时的算法计算 md5，与 Cookie 自带的 hash 值对比。

如果用户自己对 Cookie 进行修改，无论改用户 id、过期时间，还是 hash 值，都会导致最终计算结果不一致。
即使用户知道自己的 id 和口令，也知道服务器的生成算法，他也无法自己构造出有效的 Cookie，原因就在于计算 hash 时的“系统固定的随机字符串”他不知道。
这个“系统固定的随机字符串”还有一个用途，就是编写代码的开发人员不知道生产环境服务器配置的随机字符串，他也无法伪造 Cookie。
md5 算法还可以换成更安全的 sha1/sha256。
现在我们就解决了如何生成一个可信 Cookie 的问题。
