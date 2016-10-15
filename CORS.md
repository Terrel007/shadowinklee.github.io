### 写在前面 ###

分享本文旨在，熟悉JS跨域的常用实践。下面所有例子中域名`alvin.morningstar.com`是localhost映射的，站点A是部署在IIS服务器，而站点B则是启动的Node服务器。

### JSONP跨域 ###

JSONP是JSON with padding(填充式JSON或参数式JSON)的简写,是应用JSON的一种新方法。JSONP看起来与JSON差不多，只不过是被包含在函数调用中的JSON,由两部分组成:回调函数和数据。回调函数的名字在请求中指定，在后台获取函数名与JSON数据拼接成函数调用返回前台实现跨域。

场景: 站点A `http://alvin.morningstar.com`下的index.html页面，利用script标签的src属性可以指定跨域url,访问站点B `http://alvin.morningstar.com:9000`下的一个api,并以表格形式展示其返回数据。

效果如下:

![index页面](http://i.imgur.com/EKv1Whv.png)

代码如下：


- index.html
```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <title>跨域请求</title>
    <style>
        table td {
            border: 1px solid black;
            text-align: center;
        }
    </style>
</head>
<body>
    <script>
        var globalObj = {
            a: 1
        };

        function displayUserData(data) {
            var table = document.createElement('table');
            table.border = 1;
            table.width = "60%";
            var tbody = document.createElement('tbody');
            table.appendChild(tbody);

            var colName = [];
            for (var key in data) {
                var row = data[key];
                if (colName.length == 0 && row && typeof row == 'object') {
                    colName = Object.keys(row);
                }
            }

            //创建第一行
            tbody.insertRow(0);
            for (var i = 0, m = colName.length; i < m; i++) {
                tbody.rows[0].insertCell(i).appendChild(document.createTextNode(colName[i]));
            }
            for (var key in data) {
                var j = 0;
                var row = tbody.insertRow(table.rows.length), rowData = data[key];
                for (var cellData in rowData) {
                    row.insertCell(j++).appendChild(document.createTextNode(rowData[cellData]));
                }
            }
            document.body.appendChild(table);
        }
    </script>
    <script src="http://alvin.morningstar.com:9000/Users?callback=displayUserData"></script>
</body>
</html>
	
```

- /Users api

```javacript
app.get('/Users', function(req, res) {
    var queryUrl = url.parse(req.url).query;
    var queryObj = qs.parse(queryUrl);
    
    fs.readFile(__dirname + '/data/' + 'users.json', 'utf8', function(err, data) {
        if (err) return;

        if (queryObj.callback) {
            res.end(queryObj.callback + '(' + data + ')');
        } else { res.end(data); }
    })
});
```

### iframe跨域访问和同源访问 ###

场景:站点A `http://alvin.morningstar.com`下的`a.html`页面，嵌入了两个iframe页面：一个是来自站点B `http://alvin.morningstar.com:9000`下的`node_a.html`页面，属于跨域访问; 另外一个是来自与A站点同源的另一个静态页面`index.html`。

![a页面](http://i.imgur.com/pfrCgEr.png)

代码如下:
- a.html
```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <title>iframe 跨域操作1</title>
</head>
<body>
    <iframe id="iframe1" src="http://alvin.morningstar.com:9000/public/node_a.html"></iframe>
    <div>iframe1来自于<code>http://alvin.morningstar.com:9000</code>,跟当前页面不同源</div>
    <br/>
    <iframe id="iframe2" src="http://alvin.morningstar.com/TestDemo/CORS/index.html"></iframe>
    <div>iframe2来自于<code>http://alvin.morningstar.com</code>,跟当前页面同源</div>
    <script>
        window.onload = function () {
            console.group("访问第一个iframe");
            var iframe1 = document.getElementById('iframe1');
            try{
                var iframe1_window = iframe1.contentWindow;
                showAvailablePropDesc(iframe1_window);

                var doc1 = iframe1.contentDocument;

            }catch(err){
                console.log(err);
            }
            console.groupEnd("访问第一个iframe");

            console.group("访问第二个iframe");
            var iframe2 = document.getElementById('iframe2');
            try {
                var iframe2_window = iframe2.contentWindow;
                showAvailablePropDesc(iframe2_window);
                console.log(iframe2_window.globalObj);

                var doc2 = iframe2.contentDocument;
                var div = document.createElement('div');
                div.innerHTML = '看到这条信息,表明你可以操作同源iframe文档对象';
                doc2.body.appendChild(div);
            } catch (err) {
                console.log(err);
            }
            console.groupEnd("访问第二个iframe");

        }

        //用于展示对象本身的属性
        function showAvailablePropDesc(obj) {
            var properties = Object.getOwnPropertyNames(obj);
            for (var i = 0; i < properties.length; i++) {
                var propDesc = Object.getOwnPropertyDescriptor(obj, properties[i]);
                console.log('key:' + properties[i] + ' ', propDesc);
            }
        }
    </script>
</body>
</html>

```
- node_a.html
```javascript
<!DOCTYPE html>
<html lang="en">

<head>
    <title></title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
</head>

<body>
    <div>
        Page node_a Served by Node server.
    </div>
</body>

</html>
```
- index.html 如JSONP例子列出


浏览器的同源安全策略会限制a.html对嵌入的iframe1页面的访问，通过下图可以看出只能访问iframe1页面window对象的少数属性，并且无法访问它的document对象。

![跨域window对象](http://i.imgur.com/NKT4pCa.png)


但是对于同源的iframe2页面，不仅可以访问其window对象，还可以获取其document对象，对document对象进行操作，改变iframe2页面文档。


### 设置document.domain实现父子域之间的 ###

考虑以下场景, 站点A `http://alvin.morningstar.com`下的`b.html`页面，通过iframe嵌入站点B `http://alvin.morningstar.com:9000`下的`node_b.html`页面，它们有相同的域名后缀`morningstar.com`,可以同时为b和node_b页面设置:

```javascript
document.domain='morningstar.com'
```

即可实现跨域操作。效果如下:

![b页面](http://i.imgur.com/8vQZKMn.png)

代码：
- b.html
```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <title>iframe 跨域操作2</title>
</head>
<body>
    <iframe id="iframe1" src="http://alvin.morningstar.com:9000/public/node_b.html"></iframe>
    <script>
        document.domain = 'morningstar.com';
        window.onload = function () {
            var iframe1 = document.getElementById('iframe1');
            var doc1 = iframe1.contentDocument;
            
            var div = doc1.createElement('div');
            doc1.body.appendChild(div);
            var txt = doc1.createTextNode('通过设置document.domain实现操作跨域iframe的文档');
            div.appendChild(txt);
        }
    </script>
</body>
</html>

```

- node_b.html
```html
<!DOCTYPE html>
<html lang="en">

<head>
    <title></title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
</head>

<body>
    <div>
        Page node_b Served by Node server.
    </div>
</body>
<script>
    document.domain = 'morningstar.com';
</script>

</html>
```



### HTML5 API window.postMessage跨域 ###

场景: 站点A `http://alvin.morningstar.com`下的`c.html`页面，通过iframe嵌入站点B `http://alvin.morningstar.com:9000` 下的`node_c.html`。

![c页面](http://i.imgur.com/VD6ky44.png)

通过HTML5提供的`window.postMessage`方法，实现客户端间通信。


> otherWindow.postMessage(message,targetOrigin)

- `otherWindow`:其他窗口的一个引用，比如`iframe`的`contentWindow`属性、执行`window.open`返回的窗口对象、或者是命名过或数值索引的`window.frames`。

- `message`:将要发送到其他window的数据。支持字符串，对象数据，会被结构化克隆算法序列化。

- `targetOrigin`:指定哪些窗口能够接收到消息事件，其值可以是字符串"*"(表示无限制) 或者一个URI

效果如下:

![window.postMessage客户端跨域通信](http://i.imgur.com/xxBpR8v.png)

代码如下：
- c.html
```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <title>HTML5 API:window.postMessage实现跨域</title>
</head>
<body>
    <iframe id="iframe1" src="http://alvin.morningstar.com:9000/public/node_c.html"></iframe>
    <script>
        //确保iframe加载完成
        window.onload = function () {
            var iframe1 = document.getElementById('iframe1');
            iframe1.contentWindow.postMessage('hello,i\'am from c.html page', 'http://alvin.morningstar.com:9000');
            iframe1.contentWindow.postMessage({a:1}, 'http://alvin.morningstar.com:9000');
        }

        window.addEventListener('message', function (event) {
            if (event.origin !== 'http://alvin.morningstar.com:9000') return;
            console.log(event);
            console.log(event.data);
        }, false);
    </script>
</body>
</html>


```
- node_c.html
```html
<!DOCTYPE html>
<html lang="en">

<head>
    <title></title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
</head>

<body>
    <div>
        Page node_c Served by Node server.
    </div>
</body>
<script>
    window.addEventListener('message', function(event) {
        if (event.origin !== 'http://alvin.morningstar.com') return;
        var data = event.data;
        var msg = typeof data == 'object' ? 'data object' : data;
        console.log(data);
        event.source.postMessage('Hi,I received your message,you send me :"' + msg + '"', event.origin);
    }, false);
</script>

</html>
```

    

### HTTP访问控制(CORS) ###

场景一：站点A `http://alvin.morningstar.com`下的`d.html`页面，通过XMLHttpRequest请求获取站点B的 `http://alvin.morningstar.com:9000/Users` API数据，是一个简单的跨站请求。如果服务器端返回响应消息头没有`Access-Control-Allow-Origin`，那么返回数据将会被浏览器拦截。

![No Access-Control-Allow-Origin header](http://i.imgur.com/7eAre7N.png)

如果我们在后台添加响应头`Access-Control-Allow-Origin`



> res.header("Access-Control-Allow-Origin", "http://alvin.morningstar.com");

那么就能正常获取数据，结果如下:

![Access-Control-Allow-Origin](http://i.imgur.com/Fr40fSj.png)

代码如下：
- d.html
```javascript
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <title>HTTP Access Control CORS</title>
</head>
<body>
    <script>
        var xhr = window.XMLHttpRequest && new XMLHttpRequest();
        xhr.onreadystatechange = function () {
            if (xhr.readyState == 4 && xhr.status == 200) {
                console.log(xhr.responseText);
                
            }
        }
        xhr.open('GET', 'http://alvin.morningstar.com:9000/Users', true);
        xhr.send();
    </script>
</body>
</html>
```

- /Users
```javascript
app.get('/Users', function(req, res) {
    var queryUrl = url.parse(req.url).query;
    var queryObj = qs.parse(queryUrl);
    res.header("Access-Control-Allow-Origin", "http://alvin.morningstar.com");
    fs.readFile(__dirname + '/data/' + 'users.json', 'utf8', function(err, data) {
        if (err) return;

        if (queryObj.callback) {
            res.end(queryObj.callback + '(' + data + ')');
        } else { res.end(data); }
    })
});
```

场景二: 站点A `http://alvin.morningstar.com`下的`e.html`页面，通过XMLHttpRequest将数据post至站点B的 `http://alvin.morningstar.com:9000/addUser`，并且设置了自定义的request header。这种情况下，浏览器会先发送Preflight请求，结果如下：

![Preflight request](http://i.imgur.com/xkLL0a6.png)

代码如下：

- e.html
```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <title></title>
</head>
<body>
    <script>
        var xhr = window.XMLHttpRequest && new XMLHttpRequest();
        var user = {
            name: 'alvin',
            password: '123456',
            profession: 'Front-end Developer'
        }
        var postData = 'data=' + JSON.stringify(user);
        xhr.onreadystatechange = function () {
            if (xhr.readyState == 4 && xhr.status == 200) {
                console.log(xhr.responseText);

            }
        }
        xhr.open('post', 'http://alvin.morningstar.com:9000/addUser', true);
        xhr.setRequestHeader('X-Custom-Header', 'javascript');
        xhr.send(postData);
    </script>
</body>
</html>

```

- /addUSer

```javascript
app.post('/addUser', function(req, res) {
    var postData = '';

    req.on('data', function(chunk) {
        postData += chunk;
    })
    req.on('end', function() {
        var params = qs.parse(postData);
        var user = params && params['data'];

        fs.readFile(__dirname + '/data/' + 'users.json', 'utf8', function(err, data) {
            if (err) return;

            data = JSON.parse(data);

            var index = Object.keys(data).length;
            index++;
            user['id'] = index;
            data['user' + index] = user;
            fs.writeFile(__dirname + '/data/' + 'users.json', JSON.stringify(data), function() {
                console.log('写入成功');
                res.end(JSON.stringify(data));
            })
        })
    });
});
```
由于服务器端返回响应消息头里没有跨域设置，返回数据会被浏览器拦截。

接下来，我在node后端设置响应头，添加如下代码：
```javascript
app.all('*', function(req, res, next) {
    res.header('Access-Control-Allow-Origin', 'http://alvin.morningstar.com');
    res.header('Access-Control-Allow-Headers', 'X-Custom-Header');
    res.header('Access-Control-Allow-Methods', 'OPTIONS,POST');
    next();
})

```
结果如下：

![HTTP Access Control CORS ](http://i.imgur.com/TQOBOCU.png)

第一次Prefilght请求返回的响应头里包含了服务器端的跨域设置，第二次post请求添加数据成功并且返回。





