
# node连接MySQL及基本用法


node的[mysql包](https://www.npmjs.com/package/mysql)

```
$ npm install mysql
```


连接流程

```js
const mysql  = require('mysql')  //引入模块

//创建一个connection
/**
* option参数说明
* database 数据库名              charset    链接字符集
* socktePath链接到unix的路径     timezone   时区
* connectTimeout 超时时间        stringifyObjects 是否序列化对象
* typeCast 是否将列值转化为本地JavaScript类型之.. 等等
*/
const connection = mysql.createConnection({     
  host     : '192.168.0.200',       //主机
  user     : 'root',               //MySQL认证用户名
  password : '123456',        //MySQL认证用户密码
  port: '3306',                   //端口号
})
// 启动连接
connection.connect(function(err){
  if(err){        
    console.log('query - :'+err);
    return
  }
  console.log('mysql connect succeed!');
});  
//执行SQL语句
connection.query('SELECT 1 + 1 AS solution', function(err, rows, fields) { 
  if (err) {
    console.log('[query] - :'+err);
    return
  }
  console.log('The solution is: ', rows[0].solution);  
})
//关闭连接
connection.end(function(err){
  if(err){        
    return;
  }
    console.log('[connection end] succeed!');
})
```


### MySQL的CRUD简单介绍

```js
const DATABASE = 'database1'
const TABLE = 'table1'

// C
const add = `insert into ${DATABASE}(name) values(?)`
const params = 'xiaoming'

connection.query(add, params, (err, result) => {
  if(err) {
    console.log(err)
    return
  }
  console.log('insert success:', result)
})

// D
const remove = `delete from ${DATABASE} where name = 'xiaowang'`
connection query(remove, (err, result) => {
  if(err) {
    console.log(err)
    return
  }
  console.log('delete success:', result)
})

// Retrieve
const retrieve = `select * from ${TABLE}`
connection.query(retrieve, (err, result, fields) => {
  if(err) {
    console.log(err)
    return
  }
  console.log(result)
})

// U
var update = `update ${DATABASE} set name = 'xiaohong' where name = ?`;
var param = ['xiaolv'];
connection.query(update, param, function (error, result) {
    if(error) {
      console.log(error);
      return
    }
    console.log(result)
});
``` 


### 连接池

> 数据库连接池负责分配、管理和释放数据库连接，它允许应用程序重复使用一个现有的数据库连接，而不是再重新建立一个；释放空闲时间超过最大空闲时间的数据库连接来避免因为没有释放数据库连接而引起的数据库连接遗漏。这项技术能明显提高对数据库操作的性能。

使用createPool创建，可以监听，或者直接使用，也可以共享一个链接或管理多个连接

```js
/*
* option连接池配置
* waitForConnections 当连接池没有连接或超出最大限制时，设置为true且会把链接放入队列，设置为false会返回error
* connectionLimit    链接限制数，默认10
* queueLimit         最大连接请求队列限制，设置为0表示不限制，默认：0
**/

const pool = mysql.createPool({
  host: '192.168.0.195',
  user: 'root',
  password: '1234',
})

// 监听connect事件
pool.on('connect', connection => {
  connection.query('')
})

// 直接使用
pool.query(`...`, (err, result) => {
  // ...
})

// 共享
pool.getConnection( (err, connection) => {
  // ...
})
```


### 默认重连

数据库可以因为各种原因导致连接不上，这种就必须有重连接机制！主要判断errorcode:PROTOCOL_CONNECTION_LOST 

```js
const mysql = require('mysql');
const db_config = {
  host     : '192.168.0.200',       
  user     : 'root',              
  password : 'abcd',       
  port: '3306',                   
  database: 'sample'  
};

let connection
function handleDisconnect() {
  connection = mysql.createConnection(db_config);      
  // 在connect回调里递归自己                                         
  connection.connect(function(err) {              
    if(err) {                                     
      console.log("进行断线重连：" + new Date());
      setTimeout(handleDisconnect, 2000);   //2秒重连一次
      return;
    }         
     console.log("连接成功");  
  });                                                                           
  connection.on('error', function(err) {
    console.log('db error', err);
    if(err.code === 'PROTOCOL_CONNECTION_LOST') { 
      handleDisconnect();                         
    } else {                                      
      throw err;                                 
    }
  });
}
handleDisconnect()

```