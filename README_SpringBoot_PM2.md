> [!note] PM2로 결정한 이유
>  nohub, forever, pm2 등 백그라운드 실행 패키지가 있는데 forever는 업데이트가 중단 되었고 nohup은 자동으로 꺼지는 에러가 있다고 해서 프로세스가 잘 유지되고 쉽게 관리할 수 있는 PM2를 사용해 배포 했음
# _PM2로 배포하기_

> _배포 환경_
```
[manager@dev-was ~]$ cat /etc/redhat-release
CentOS Linux release 7.9.2009 (Core)
```

> _Spring Boot version_
```
plugins {  
    java  
    id("org.springframework.boot") version "3.2.1"  
    id("io.spring.dependency-management") version "1.1.4"  
}
```

-1. _NPM 설치_
```null
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash

. ~/.nvm/nvm.sh

nvm install --lts
```

-2. _PM2 설치_
```null
npm install -g pm2
```

-3. _Gradle build_

-4. _PM2로 실행시키기 위한 파일이름.json 생성_
```javascript
// server.json
{
  "apps" : [{
	name   : "[프로그램이름]",  // pm2 name
	script : "java",           // 앱 실행 스크립트
	args   : [                 // CLI로 실행되는 인자
		"-jar",
		"[libs 디렉토리]/[스냅샷 파일이름].jar"
	],
	"watch": [                 // 폴더 또는 특정 폴더를 감시하여 변경되면 Reload를 진행
		"[libs 디렉토리]/[스냅샷 파일이름].jar"
	],
	"log_date_format": "YYYY-MM-DD HH:mm Z",  // 로그 날짜 형식
	"exec_interpreter": "",     // 인터프리터 경로
	"exec_mode": "fork",        // fork, cluster 모드 중 선택
	"autorestart": true,        // 프로세스 실패 시 자동으로 재시작할지 선택
	"output": "~/logs/pm2_server/console.log",      // 로그 출력 경로 설정
	"error": "~/logs/pm2_server/consoleError.log",  // 에러 로그 출력 경로 설정
  }]
}
```
#### Fields 설명

**Info**
- `name`: 어플리케이션 이름
- `interpreter`: 인터프리터 경로 (기본값은 `Node` 입니다.)
- `interpreter_args`: 인터프리터 옵션입니다.
- `script`: pm2 시작에 대한 시작 파일 경로 (`interpreter`으로 실행됩니다.)
- `cwd`: 앱이 실행될 디렉토리
- `args`: CLI로 실행되는 인자

**Advanced**
- `instances`: 인스턴스 수를 결정할 수 있으며, **0**으로 작성하면 현재 가능한 CPU 코어 수만큼 실행됩니다.
- `exec_mode`: **fork**모드로 실행할지, **cluster**모드로 실행할지 선택
- `watch`: 폴더 또는 특정 폴더를 감시하여 변경되면 Reload를 진행합니다.
- `ignore_watch`: watch를 제외할 경로를 작성합니다.
- `max_memory_restart`: 설정한 메모리를 초과할 시, 자동으로 재시작됩니다.
- `source_map_support`: 소스맵 활성화 여부를 선택합니다.

**Logs**
- `log_date_format`: 로그 날짜 형식 (기본값은 YYYY-MM-DD HH:mm Z 입니다.)
- `output`: 로그 출력 경로를 설정합니다.
- `error`: 에러 로그 출력 경로를 설정합니다.

**Control Flow** [(Reference)](https://pm2.keymetrics.io/docs/usage/signals-clean-restart/#cleaning-states-and-jobs)
- `min_uptime`: 어플리케이션이 가동되었다고 생각하는 최소 시간
- `kill_timeout`: 최정 SIGKILL을 보내기까지의 시간
- `wait_ready`: Reload 대기 이벤트 대신에 어플리케이션에서의 **process.send('ready')** 를 기다립니다.
- `autorestart`: 프로세스를 1회만 실행시키고 싶을 때, **false**로 설정합니다.

-5. _PM2 실행_
#### Commands

여기서 `restart`와 `reload`의 차이는 `restart`는 프로세스를 종료하고 다시 시작하는 것과 다르게, `reload`는 0초 다운타임 재로드가 됩니다.

```shell
$ pm2 start ecosystem.config.json

$ pm2 stop ecosystem.config.json

$ pm2 restart ecosystem.config.json

$ pm2 reload ecosystem.config.json

$ pm2 delete ecosystem.config.json
```

### _예상치 못한 시스템 재부팅?_

어플리케이션이 종료되는 것은 PM2가 알아서 관리해주지만, 하지만 시스템이 재부팅되어 PM2 자체가 종료된다면 과연 누가 관리해줄까요?  
이를 위해서 다음과 같은 방법이 존재합니다.

```shell
$ pm2 save

$ pm2 startup

[PM2] You have to run this command as root. Execute the following command:
      sudo su -c "env PATH=$PATH:/home/unitech/.nvm/versions/node/v14.3/bin pm2 startup <distribution> -u <user> --hp <home-path>
```

위 처럼 출력된 명령어(`sudo...`)를 복사/붙여넣기 해주면 됩니다.

추가적으로, Node 버전이 바뀐다던지, 하나의 서버에서 여러개의 사용자 계정이 존재하는 경우에는 다음 [Documentation](https://pm2.keymetrics.io/docs/usage/startup/)을 참고하여 설정해주면 됩니다.

## _PM2 로그 관리_

위 Config 파일에서 설정한 로그 경로에 저장되는 로그는 우리가 삭제해주지 않는한 계속 쌓이게 됩니다. 이렇게 계속 쌓이게 된다면 서버 용량 부족으로 이어질 수도 있는데요. 이를 해결하기 위해서 로그 파일을 관리해주는 `pm2-logrotate`라는 plugin을 사용해줍시다.

사용방법은 다음과 같습니다.

```shell
# npm으로 설치하는 것이 아닌 pm2로 설치해주어야 합니다!
$ pm2 install pm2-logrotate

# 옵션 설정은 다음과 같이 설정해줍니다.
$ pm2 set pm2-logrotate:<option> <value>

$ pm2 set pm2-logrotate:max_size 1K
$ pm2 set pm2-logrotate:retain 10
```

**Options**

- `max_size`: 로그 파일 사이즈 제한 크기 (기본값은 10M)
- `retain`: 로그 파일을 최대 몇개까지 가지고 있을 것인지 설정 (기본값은 30개)
- `compress`: 로그 파일을 gzip으로 압축할 것인지 여부 (기본값은 false)
- `dateFormat`: 로그 파일 날짜 폼새 (기본값은 YYYY-MM-DD_HH-mm-ss)
- `workerInterval`: 로그 파일 사이즈를 확인하는 1초마다의 회수 (기본값 초당 30회)
- `rotateInterval`: cron job (기본값은 '0 0 * * *')
