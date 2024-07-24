# 使用服务器发送事件
开发一个使用服务器发送事件的 Web 应用程序是很容易的。你只需要在服务器上编写一些代码将事件流传输到前端，而客户端的代码在处理传入事件部分几乎与 websocket 相同。这是一个单向的连接，所以你不能从客户端发送事件到服务器。

## 从服务端接收事件
### 创建一个 EventSource 实例
使用生成事件的脚本 URL 创建一个 EventSource 对象，用来开启与服务器的连接并从中接收事件。例如：
```
const evtSource = new EventSource("ssedemo.php");
```
如果生成事件的脚本不同源，应该创建一个新的包含 URL 和 options 参数的 EventSource 对象。例如，假设客户端脚本在 example.com 上：
```
const evtSource = new EventSource("//api.example.com/ssedemo.php", {
  withCredentials: true,
});
```
### 监听 message 事件
如果服务器发送的消息中没有 event 字段，则这些消息会被视为 message 事件。为了接收这些 message 事件，需要为 message 事件附加一个事件处理程序。
```
evtSource.onmessage = function (event) {
  const newElement = document.createElement("li");
  const eventList = document.getElementById("list");

  newElement.innerHTML = "message: " + event.data;
  eventList.appendChild(newElement);
};
```
上面的代码监听了那些从服务器发送来的 message 事件，然后把消息内容显示在页面文档的列表中。
### 监听自定义事件
如果服务器发送的消息中定义了 event 字段，就会以 event 中给定的名称作为事件接收。例如：
```
evtSource.addEventListener("ping", (event) => {
  const newElement = document.createElement("li");
  const eventList = document.getElementById("list");
  const time = JSON.parse(event.data).time;
  newElement.textContent = `ping at ${time}`;
  eventList.appendChild(newElement);
});
```
每当服务器发送一个 event 字段设置为 ping 的消息时，就会调用这段代码；这段代码接着解析 data 字段中的 JSON，并输出这些信息。
## 从服务器端发送事件
发送事件的服务器端脚本需要使用 text/event-stream MIME 类型响应内容。每个通知以文本块形式发送，并以一对换行符结尾。有关事件流的格式的详细信息，请参见事件流格式。
演示的 PHP 代码如下：
```
date_default_timezone_set("America/New_York");
header("Cache-Control: no-store");
header("Content-Type: text/event-stream");

$counter = rand(1, 10);
while (true) {
  // Every second, send a "ping" event.
  echo "event: ping\n";
  $curDate = date(DATE_ISO8601);
  echo 'data: {"time": "' . $curDate . '"}';
  echo "\n\n";
  // Send a simple message at random intervals.
  $counter--;
  if (!$counter) {
    echo 'data: This is a message at time ' . $curDate . "\n\n";
    $counter = rand(1, 10);
  }
  ob_end_flush();
  flush();
  // Break the loop if the client aborted the connection (closed the page)
  if (connection_aborted()) break;
  sleep(1);
}
```
这段代码每秒钟会生成一个类型为“ping”的事件，每个事件的数据都是一个 JSON 对象，包含对应于事件生成时间的 ISO 8601 时间戳。同时，还会在随机的时间间隔内发送一个简单的消息（没有事件类型）。这个循环会一直运行，不受连接状态的影响，因此在检查到连接关闭的情况时需要手动中断循环（例如客户端关闭页面）。
## 错误处理
当发生错误时（例如网络超时或与访问控制有关的问题），会生成一个错误事件。可以通过在 EventSource 对象上实现 onerror 回调来编程地处理它：
```
evtSource.onerror = (err) => {
  console.error("EventSource failed:", err);
};
```
## 关闭事件流
默认情况下，如果客户端和服务器之间的连接关闭，则连接将重新启动。可以使用 .close() 方法终止连接。
```
evtSource.close();
```
## 事件流格式
事件流是一个简单的文本数据流，文本应该使用 UTF-8 格式的编码。事件流中的消息由一对换行符分开。以冒号开头的行为注释行，会被忽略。
> 备注： 注释行可以用来防止连接超时，服务器可以定期发送一条消息注释行，以保持连接不断。
每条消息由一行或多行文字组成，列出该消息的字段。每个字段由字段名表示，后面是冒号，然后是该字段值的文本数据。
### 字段
规范中规定了下面这些字段：
`event`
一个用于标识事件类型的字符串。如果指定了这个字符串，浏览器会将具有指定事件名称的事件分派给相应的监听器；网站源代码应该使用 addEventListener() 来监听指定的事件。如果一个消息没有指定事件名称，那么 onmessage 处理程序就会被调用。
`data`
消息的数据字段。当 EventSource 接收到多个以 data: 开头的连续行时，会将它们连接起来，在它们之间插入一个换行符。末尾的换行符会被删除。
`id`
事件 ID，会成为当前 EventSource 对象的内部属性“最后一个事件 ID”的属性值。
`retry`
重新连接的时间。如果与服务器的连接丢失，浏览器将等待指定的时间，然后尝试重新连接。这必须是一个整数，以毫秒为单位指定重新连接的时间。如果指定了一个非整数值，该字段将被忽略。
