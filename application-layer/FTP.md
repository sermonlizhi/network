## 一、FTP文件传输协议

传输层使用TCP协议，向远程主机上传输文件或从远程主机接收文件，服务器端口号为21

控制连接和数据连接分开：

控制连接在21号端口，数据连接在20号端口

请求文件资源过程：

- FTP客户端与FTP服务器通过端口21联系，并使用TCP为传输协议
- 客户端通过控制连接获得身份确认(用户名/密码)
- 客户端通过控制连接发送命令浏览远程目录(list)
- 收到一个传输文件命令时，服务器主动打开一个到客户端的数据连接
- 一个文件传输完成后，服务器关闭当前数据连接
- 服务器打开第二个TCP数据连接用来传输另一个文件
- 控制连接称为带外，数据连接称为带内
- FTP服务器维护用户的状态信息：当前路径、用户账户与控制连接对应



## 二、FTP响应/命令

- 命令样例
  - 命令在控制连接上以ASCⅡ文本方式传送
  - USER username/PASS  password
  - LIST：请服务器返回远程主机当前目录的文件列表
  - RETY fileName：从远程主机的当前目录检索文件(gets)
  - STOR  fileName：向远程主机的当前目录存放文件(puts)
- 返回码样例
  - 状态码和状态信息(同HTTP)
  - 331 Username OK，password  required
  - 125  data  connection  already  open; transfer starting
  - 425  cant open data connection
  - 452  Error writing file



