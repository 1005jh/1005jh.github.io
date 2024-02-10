---
title: docker를 활용한 mysql slave db
date: "2023-12-05T12:36:37.121Z"
template: "post"
draft: false
category: "docker"
tags:
  - "docker"
  - "nestjs"
  - "mysql"

description: "docker를 활용한 mysql slave db"
---

jmeter를 활용해 테스트를 진행 중 master, slave db를 적용한 환경에서의 테스트가 필요했다.
하지만 실제 db에 replication을 적용해 테스트를 진행하기에는 비용적으로 너무 소모가 심했다.
그래서 로컬환경에서 테스트를 하기 위해 docker를 활용하여 mysql의 master, slave 환경을 구성했다.

참고할 많은 자료가 있지만 이 <a href="https://velog.io/@hyunho058/Mysql-%EB%B6%84%EC%82%B0%EC%B2%98%EB%A6%ACReplication-with-docker">블로그</a>가 정리가 잘 되어있어 참고해 작업을 진행했고, spring 환경에서의 적용이 필요하면 참고하면 좋을 것 같다. 그러면 nestjs 환경에서 docker를 활용해 mysql의 master, slave 세팅을 시작해보자.

```
app
├── docker-compose.yml
├── src
├── master
    ├── Dockerfile
    └── my.cnf
└── slave
    ├── Dockerfile
    └── my.cnf

```

구조는 위와같이 되어있다.

```yml
version: "3"
services:
  db-write:
    build:
      context: ./
      dockerfile: master/Dockerfile
    restart: always
    environment:
      MYSQL_DATABASE: "aliy"
      MYSQL_USER: "user"
      MYSQL_PASSWORD: "password"
      MYSQL_ROOT_PASSWORD: "password"
    ports:
      - "3321:3306"
    volumes:
      - master:/var/lib/mysql
      - master:/var/lib/mysql-files

  db-read:
    build:
      context: ./
      dockerfile: slave/Dockerfile
    restart: always
    environment:
      MYSQL_DATABASE: "aliy"
      MYSQL_USER: "user"
      MYSQL_PASSWORD: "password"
      MYSQL_ROOT_PASSWORD: "password"
    ports:
      - "3322:3306"
    volumes:
      - slave:/var/lib/mysql
      - slave:/var/lib/mysql-files

volumes:
  master:
  slave:
```

위처럼 yml파일을 작성을 해주고, 각 master, slave에 대한 dockerfile을 만들어줘야 한다.
우선 master에 대한 파일이다.

```dockerfile
# master
FROM mysql:8.0.33
ADD ./master/my.cnf /etc/mysql/my.cnf
```

```cnf
<!-- master my.cnf -->
[mysqld]
log_bin = mysql-bin
server_id = 10
binlog_do_db=aliy
default_authentication_plugin=mysql_native_password
```

아래는 slave에 대한 파일이다.

```dockerfile
# slave
FROM mysql:8.0.33
ADD ./slave/my.cnf /etc/mysql/my.cnf
```

```cnf
<!-- slave my.cnf -->
[mysqld]
log_bin = mysql-bin
server_id = 11
relay_log = /var/lib/mysql/mysql-relay-bin
log_slave_updates = 'ON'
read_only = 'ON'
default_authentication_plugin=mysql_native_password
```

위와 같이 작업이 끝나면 `docker-compose up -d` 명령어를 통해 실행을 시켜준다.

성공적으로 실행이 되었다면 master db에 접속을 한다. 접속 후 terminal tab으로 이동 후 `mysql -u root -p`를 통해 접속 후 아래 명령어를 차례로 입력해준다.

```SQL
create user 'replicationUser'@'%' identified by 'password';

GRANT REPLICATION SLAVE ON *.* TO 'replicationUser'@'%';
```

위 명령어는 사용자를 생성하고, 권한을 부여하는 명령어이다. 사용자 생성과정에서 에러가 생긴다면 생성된 유저인지 체크를 해보면 된다.
작업이 완료되면 `show master status;`명령어를 통해 file과 position을 확인해준다.

```SQL
show master status;
-- 결과
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      682 | aliy         |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

그 후 master의 `IPv4Address`를 확인하기 위해 터미널로 돌아와 `docker network ls`를 해준다.

```
NETWORK ID     NAME             DRIVER    SCOPE
772bbbb08bb5   aliy_net-mysql   bridge    local
```

이처럼 뜨는 걸 확인할 수 있는데 여기서 name을 이용해 `docker inspect aliy_net-mysql` 명령어를 입력해준다.

```
[
    {
        "Name": "aliy_net-mysql",
        "Id": "772bbbb08bb5934a6714c33e7c5287a3a54596e33d1d8344bdd486d29ff839af",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.26.0.0/16",
                    "Gateway": "172.26.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "30be0d7d3169a2f7f33527972e0c5deaccf9b9e00b5e93d7678d9e7b0ff58f6c": {
                "Name": "aliy-db-read-1",
                "EndpointID": "4110530eb5f723f249fe4befd0ccb3ab0b292b16383e592cdeeffbf310163aee",
                "MacAddress": "02:42:ac:1a:00:02",
                "IPv4Address": "172.26.0.2/16",
                "IPv6Address": ""
            },
            "9e110b4581ffd76c1202e6d84e04e9fdd6e2670a1b994f94cf233c66b7a62111": {
                "Name": "aliy-db-write-1",
                "EndpointID": "f35911e2ca400f7cd49cc45e16d5b7296ffd190cfcfd0d941f95f1aa8a084ba7",
                "MacAddress": "02:42:ac:1a:00:03",
                "IPv4Address": "172.26.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {
            "com.docker.compose.network": "net-mysql",
            "com.docker.compose.project": "aliy",
            "com.docker.compose.version": "2.15.1"
        }
    }
]
```

그러면 위와같이 나오는 걸 확인할 수 있고 여기서 얻은 정보들로 slave db에 접속해 세팅을 하면 된다.
slave db에 접속 후

```SQL
CHANGE MASTER TO MASTER_HOST='172.26.0.3', -- write db의 IPv4Address
MASTER_USER='replicationUser', -- write db에서 생성한 유저의 이름
MASTER_PASSWORD='password', -- write db에서 생성한 유저의 password
MASTER_LOG_FILE='mysql-bin.000003', -- write db에서 show master status;를 통해 확인한 file
MASTER_LOG_POS=682, -- wirte db에서 show master status;를 통해 확인한 position
GET_MASTER_PUBLIC_KEY=1;
```

위와 같이 입력해준다.
이후 slave를 시작해주고, 확인을 해보면

```SQL
start slave;

show slave status\G;
-- 결과
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 172.26.0.3
                  Master_User: replicationUser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 682
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 326
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 682
              Relay_Log_Space: 536
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 13
                  Master_UUID: c03c121d-c7f1-11ee-9d4a-0242ac1a0003
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 1
            Network_Namespace:
1 row in set, 1 warning (0.00 sec)
```

위처럼 나온다. 여기서

```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

이 두 옵션이 중요하다. No로 표기되어 있으면 연동되지 않은 것이다.

그러면 프로젝트로 돌아와 `config`설정을 해준다.

```
app
└── src
    └── config
          ├── typeorm.config.ts
          └── app.config.ts
```

위의 두개의 파일을 세팅을 아래와 같이 해준다.

```javascript
// config/typeorm.config.ts
export const databaseConfig = () => ({
  type: "mysql",
  entities: ["dist/**/*.entity{.ts,.js}"],
  migrations: ["dist/migrations/*{.ts,.js}"],
  replication: {
    master: {
      host: "localhost",
      port: 3321,
      username: "root",
      password: "password",
      database: "aliy",
    },

    slaves: [
      {
        host: "localhost",
        port: 3322,
        read: true,
        username: "root",
        password: "password",
        database: "aliy",
      },
    ],
  },
  synchronize: true,
});
```

여기서 slaves는 `ConnectionOptions`에서 반복문으로 돌며 작업하기 때문에 배열형식으로 하지 않으면 안된다.

```js
// config/app.config.ts
import { databaseConfig } from "./typeorm.config";

export default () => ({
  database: { ...databaseConfig() },
});
```

위와 같이 세팅이 끝나면 app.module.ts로 가서

```js
@Module({
  imports: [
    ...
    ConfigModule.forRoot({
      isGlobal: true,
      load: [appConfig],
    }),
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => {
        const config = configService.get<ConnectionOptions>('database');
        return config;
      },
      inject: [ConfigService],
    }),
    ...]})
...
```

위와 같이 작성을 해주고, `npm run start`를 통해 애플리케이션을 실행해준다.

그러고 docker에서 master, slave를 접속해서 확인을 해본다면

```SQL
show databases;
-- 결과
+--------------------+
| Database           |
+--------------------+
| aliy               |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.02 sec)
```

위처럼 설정해준 db가 master, slave에 뜨는 걸 확인할 수 있다.
test로 유저를 넣고, master, slave에 작 적용이 되어 있는지 확인하면 된다.

```SQL
use aliy;

select * from user\G;
-- 결과
*************************** 1. row ***************************
       createdAt: 2023-12-05 09:20:35.399988
       updatedAt: 2023-12-05 09:20:35.399988
              id: 1
        nickname: NULL
    ...
1 row in set (0.00 sec)
```

적용이 잘 되었다면 위와같이 확인해볼 수 있을 것이다.

참고

<a href="https://velog.io/@hyunho058/Mysql-%EB%B6%84%EC%82%B0%EC%B2%98%EB%A6%ACReplication-with-docker">https://velog.io/@hyunho058/Mysql-%EB%B6%84%EC%82%B0%EC%B2%98%EB%A6%ACReplication-with-docker</a>

<a href="https://jupiny.com/2017/11/07/docker-mysql-replicaiton/">https://jupiny.com/2017/11/07/docker-mysql-replicaiton/</a>

<a href="https://docs.docker.com/">https://docs.docker.com/</a>
