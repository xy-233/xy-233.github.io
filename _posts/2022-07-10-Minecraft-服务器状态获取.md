---
layout:     post   				    # 使用的布局（不需要改）
title:      Minecraft 服务器状态获取 				# 标题 
subtitle:   通过 Python 获取 Minecraft 服务器名称、版本、人数、在线玩家信息等 #副标题
date:       2022-07-10 				# 时间
author:     xy2333_						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Python
    - Socket
    - Flask
---
## 前言

最近开了个服务器和朋友玩 Minecraft，不过大家开服都是朝着联机去的，服务器只剩自己在线了就想退出了。下次想玩的时候一启动游戏发现没人在线，于是后面连启动都懒得启动了，玩的人越来越少成了恶性循环。

后面看到 [MCMOD 服务器列表](https://play.mcmod.cn/list/) 里面都是这样的：

![1683700180431](https://xy-233.github.io/img/posts/2022-07-10-Minecraft-服务器状态获取/1683700180431.png)

每个服务器后面都有版本和在线人数显示，就想到一定是有办法能直接获取服务器信息的。

## 查找资料

后面在网上搜索了一会，找到了一个能用的 API：`https://mcapi.us/server/status?ip=`，不过这个 API 服务器在国外，有时候会获取不到国内 MC 服务器的信息，于是继续找这个 API 的原理。

直到找到[这个网站](https://wiki.vg/)下的这段资料：

> ## Current (1.7+)
>
> This uses the regular client-server [protocol](https://wiki.vg/Protocol "Protocol"). For the general packet format, see that article.
>
> ### Handshake
>
> First, the client sends a [Handshake](https://wiki.vg/Protocol#Handshake "Protocol") packet with its state set to 1.
>
> | Packet ID      | Field Name       | Field Type                                                                                                                                                                                                                                                                                                                                            | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
> | -------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
> | 0x00           | Protocol Version | VarInt                                                                                                                                                                                                                                                                                                                                                | See[protocol version numbers](https://wiki.vg/Protocol_version_numbers "Protocol version numbers"). The version that the client plans on using to connect to the server (which is not important for the ping). If the client is pinging to determine what version to use, by convention `-1` should be set.[![Warning.png](https://wiki.vg/images/c/cb/Warning.png)](https://wiki.vg/File:Warning.png) Setting invalid (nonexistent) version as the protocol version *might* cause some servers to close connection after this packet``[![Warning.png](https://wiki.vg/images/c/cb/Warning.png)](https://wiki.vg/File:Warning.png) See [Protocol version numbers](https://wiki.vg/Protocol_version_numbers "Protocol version numbers") for a list of valid protocol versions. |
> | Server Address | String           | Hostname or IP, e.g. localhost or 127.0.0.1, that was used to connect. The Notchian server does not use this information. Note that SRV records are a complete redirect, e.g. if _minecraft._tcp.example.com points to mc.example.org, users connecting to example.com will provide mc.example.org as server address in addition to connecting to it. |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
> | Server Port    | Unsigned Short   | Default is 25565. The Notchian server does not use this information.                                                                                                                                                                                                                                                                                  |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
> | Next state     | VarInt           | Should be 1 for[status](https://wiki.vg/Protocol#Status "Protocol"), but could also be 2 for [login](https://wiki.vg/Protocol#Login "Protocol").                                                                                                                                                                                                                  |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
>
> ### Status Request
>
> The client follows up with a [Status Request](https://wiki.vg/Protocol#Status_Request "Protocol") packet. This packet has no fields. The client is also able to skip this part entirely and send a [Ping Request](https://wiki.vg/Protocol#Ping_Request "Protocol") instead.
>
> | Packet ID | Field Name    | Field Type | Notes |
> | --------- | ------------- | ---------- | ----- |
> | 0x00      | *no fields* |            |       |
>
> ### Status Response
>
> The server should respond with a [Status Response](https://wiki.vg/Protocol#Status_Response "Protocol") packet. Note that Notchian servers will for unknown reasons wait to receive the following [Ping Request](https://wiki.vg/Protocol#Ping_Request "Protocol") packet for 30 seconds before timing out and sending Response.
>
> | Packet ID | Field Name    | Field Type | Notes                                                                                 |
> | --------- | ------------- | ---------- | ------------------------------------------------------------------------------------- |
> | 0x00      | JSON Response | String     | See below; as with all strings this is prefixed by its length as a VarInt(2-byte max) |
>
> The JSON Response field is a [JSON](http://en.wikipedia.org/wiki/JSON "wikipedia:JSON") object which has the following format:
>
> ```
> {
>    "version": {
>        "name": "1.19.3",
>        "protocol": 761
>    },
>    "players": {
>        "max": 100,
>        "online": 5,
>        "sample": [
>            {
>                "name": "thinkofdeath",
>                "id": "4566e69f-c907-48ee-8d71-d7ba5aa00d20"
>            }
>        ]
>    },
>    "description": {
>        "text": "Hello world"
>    },
>    "favicon": "data:image/png;base64,<data>",
>    "enforcesSecureChat": true
> }
> ```

## 代码实现

既然方法有了，只要按顺序发包就好了，代码如下：

```python
import socket
import struct
import json
import time

class StatusPing:
    """ Get the ping status for the Minecraft server """

    def __init__(self, host='localhost', port=25565, timeout=5):
        """ Init the hostname and the port """
        self._host = host
        self._port = port
        self._timeout = timeout

    def _unpack_varint(self, sock):
        """ Unpack the varint """
        data = 0
        for i in range(5):
            ordinal = sock.recv(1)

            if len(ordinal) == 0:
                break

            byte = ord(ordinal)
            data |= (byte & 0x7F) << 7*i

            if not byte & 0x80:
                break

        return data

    def _pack_varint(self, data):
        """ Pack the var int """
        ordinal = b''

        while True:
            byte = data & 0x7F
            data >>= 7
            ordinal += struct.pack('B', byte | (0x80 if data > 0 else 0))

            if data == 0:
                break

        return ordinal

    def _pack_data(self, data):
        """ Page the data """
        if type(data) is str:
            data = data.encode('utf8')
            return self._pack_varint(len(data)) + data
        elif type(data) is int:
            return struct.pack('H', data)
        elif type(data) is float:
            return struct.pack('L', int(data))
        else:
            return data

    def _send_data(self, connection, *args):
        """ Send the data on the connection """
        data = b''

        for arg in args:
            data += self._pack_data(arg)

        connection.send(self._pack_varint(len(data)) + data)

    def _read_fully(self, connection, extra_varint=False):
        """ Read the connection and return the bytes """
        packet_length = self._unpack_varint(connection)
        packet_id = self._unpack_varint(connection)
        byte = b''

        if extra_varint:
            # Packet contained netty header offset for this
            if packet_id > packet_length:
                self._unpack_varint(connection)

            extra_length = self._unpack_varint(connection)

            while len(byte) < extra_length:
                byte += connection.recv(extra_length)

        else:
            byte = connection.recv(packet_length)

        return byte

    def get_status(self):
        """ Get the status response """
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as connection:
            connection.settimeout(self._timeout)
            connection.connect((self._host, self._port))

            # Send handshake + status request
            self._send_data(connection, b'\x00\x00', self._host, self._port, b'\x01')
            self._send_data(connection, b'\x00')

            # Read response, offset for string length
            data = self._read_fully(connection, extra_varint=True)

            # Send and read unix time
            self._send_data(connection, b'\x01', time.time() * 1000)
            unix = self._read_fully(connection)

        # Load json and return
        response = json.loads(data.decode('utf8'))
        response['ping'] = int(time.time() * 1000) - struct.unpack('L', unix)[0]

        return response
```

这样就能获取服务器的信息了，剩下的只需要将数据处理一下，挂到 Web 上。

我的处理思路是，创建一个 `list` 用来存储所有在线玩家名字。再和每分钟从服务器获取的所有在线玩家的名字 `list` 进行比较，多出来的名字就是上线，缺少的名字就是下线。

```python
if __name__ == '__main__':
    server = StatusPing(host='127.0.0.1', port=25565)
    print(server.get_status()['players']['online'])
    lastPlayerList = list(map(lambda sample: sample['name'], server.get_status()['players'].get('sample', [])))
    while True:
        try:
            status = server.get_status()
            playerList = list(map(lambda sample: sample['name'], status['players'].get('sample', [])))
            changes = set(playerList).symmetric_difference(lastPlayerList)
            for change in changes:
                if change in playerList:
                    print(change + '上线')
                    sqliteDB.insert(change, "上线")
                if change in lastPlayerList:
                    print(change + '离线')
                    sqliteDB.insert(change, "离线")
            lastPlayerList = playerList
        except (ConnectionResetError, json.decoder.JSONDecodeError):
            pass
        time.sleep(60)
```

然后将上下线数据存储在 SQLite 里，存储对应的时间、名字、上线/下线。

```python
import os
import sqlite3
import time

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
db_path = os.path.join(BASE_DIR, "mcServerPlayerLog.db")


def insert(name, change):
    con = sqlite3.connect(db_path)
    c = con.cursor()
    c.execute("INSERT INTO table_name (time,name,change) \
          VALUES ('{}','{}','{}')".format(time.strftime('%Y-%m-%d %H:%M'), name, change))
    con.commit()
    con.close()


def select():
    con = sqlite3.connect(db_path)
    c = con.cursor()

    cursor = c.execute("SELECT time, name, change from table_name order by time desc limit 50")
    logs = []
    for row in cursor:
        logs.append(
            {
                'time': row[0],
                'name': row[1],
                'change': row[2]
            }
        )

    con.close()

    return logs
```

最后用 flask 框架挂在 Web 上，打开 `ip:1002` 即可查看。

```python
import json

from flask import Flask, render_template

import sqliteDB
from StatusPing import StatusPing

app = Flask(__name__)

server = StatusPing(host="127.0.0.1",port=25565)


@app.route('/')
def hello_world():
    status = server.get_status()
    online = status['players']['online']
    names = ','.join(list(map(lambda sample: sample['name'], status['players'].get('sample', []))))
    data = json.dumps(sqliteDB.select())
    return render_template('index.html', data=data, online=online, names=names)


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=1002)
```

顺便把对应的 `templates` 附在这里（为了美观和方便，这里用了 Element-ui CDN）：

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <!-- import CSS -->
  <link rel="stylesheet" href="https://unpkg.com/element-ui/lib/theme-chalk/index.css">
</head>
<body>
  <p>当前在线：{{online}}人</p>
  <p>{{names}}</p>
  <div id="app">
    <el-table
    :data="tableData"
    stripe
    style="width: 100%">
    <el-table-column
      prop="time"
      label="时间">
    </el-table-column>
    <el-table-column
      prop="name"
      label="名称">
    </el-table-column>
    <el-table-column
      prop="change"
      label="状态">
    </el-table-column>
  </el-table>
  </div>
<p id="data" style="display: none">{{ data }}</p>
</body>
  <!-- import Vue before Element -->
  <script src="https://unpkg.com/vue@2.6.14/dist/vue.js"></script>
  <!-- import JavaScript -->
  <script src="https://unpkg.com/element-ui/lib/index.js"></script>
  <script>
    const data = JSON.parse(document.getElementById("data").innerText)
    new Vue({
      el: '#app',
      data: function() {
        return { tableData:data }
      }
    })
  </script>
</html>
```

## 最终效果

最终效果如下：

![1683701917742](https://xy-233.github.io/img/posts/2022-07-10-Minecraft-服务器状态获取/1683701917742.png)

（最后被朋友说这是打卡系统）
