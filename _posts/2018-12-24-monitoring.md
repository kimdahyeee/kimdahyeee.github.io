###### 최근에 회사에서 모니터링 관련 세미나를 듣게 되어서, 해당 내용을 정리하려고 한다 : )
###### (내용은 정확하지 않을 수 있습니다 :-/ )

## 모니터링이란 ?
모니터링 종류에는 데이터베이스 모니터링, 서비스 모니터링, 서버 모니터링.. 등등 다양한 모니터링이 존재합니다. 모니터링이 필요한 이유는 장애를 즉각 대처하거나, 미리 사전에 방지하기 위함입니다.

## 모니터링 종류
###### (제가 일하는 곳의 모니터링 종류)
1. IMON 사용
	- 아파치, 톰캣의 정상 여부 확인
	- API에 대한 모니터링도 가능
2. batch, daemon 모니터링
	- daemon이 정상적으로 작동하는지 확인하는 모니터링
	- 해당 프로세스가 정상적으로 띄워져 있는지 확인
	- 데이터 처리가 되지 않는 대상 ( ex. 3일 이상 리포트를 받지 않는 대상) 처리
3. 예외 처리에 대한 모니터링
	- 예외 건수 (실패 건수)를 모니터링하여 cs 처리를 위함

> 해당 모니터링은 개발자, 운영자, 관리자를 분리하여 보여주도록 합니다. 코드, 스크립트 방식으로 모니터링 코드를 구현하고 알림을 받도록 설정합니다. 

## 모니터링 구현 사례
- [카카오 모니터링](http://tech.kakao.com/2016/08/25/kemi/)
- [스프링캠프2015-Spring boot 를 적용한 전사모니터링 시스템 backend 개발 사례](https://www.slideshare.net/JeminHuh/spring-boot-backend)
