# RateLimiter
* nginx rate limit 기능 테스트 및 파라미터 튜닝

## delay 테스트
* 기본 파라미터
    * key : binary_remote_addr
    * buffer size : 10m
    * limit rate : 12r/m(5초마다 요청 프록시)
    * request queue size(burst) : 5
* 테스트 시나리오
    * 동일한 클라이언트에서 10ms 간격으로 20번 요청 전송

### default (delay = 0)
* nginx config
```
location ~ ^/default/(.+) {
    limit_req zone=myzone burst=5;
    set $proxy_host $host;
    proxy_pass http://app_server/$1$is_args$args;
}
```

* 테스트 결과
    * client log
    ![image](https://user-images.githubusercontent.com/48702893/144171316-002597fd-69bb-44cf-bc4f-274cdc1de2a6.png)
    
    * nginx log
    ![image](https://user-images.githubusercontent.com/48702893/144171322-a481964c-c098-48de-b4b5-738d801ec645.png)
    
    * was log
    ![image](https://user-images.githubusercontent.com/48702893/144171328-f9d58192-3d8d-4a9d-be11-a5403f865689.png)
    
* 결론
    * request queue에 수용 가능한 요청 수인 5개를 제외하고 나머지 요청은 모두 429 에러 응답
        > 악의적인 사용자를 차단하는게 목적이므로, 요청 리소스수가 가장 많은 페이지의 요청 횟수로 request queue size 설정
    * limit rate 으로 설정한 주기인 5초마다 WAS 로 요청 전송
        > peek time tps 기준으로 WAS 가 수용 가능한 최대 부하수준까지 limit rate 을 설정하는것이 성능상 좋아보임
### nodelay
* nginx config
```
location ~ ^/nodelay/(.+) {
    limit_req zone=myzone burst=5 nodelay;
    set $proxy_host $host;
    proxy_pass http://app_server/$1$is_args$args;
}
```

* 테스트 결과
    * client log
    ![image](https://user-images.githubusercontent.com/48702893/144172474-a6b71f35-06b0-4c4c-be5a-3d6af1128482.png)
    
    * nginx log
    ![image](https://user-images.githubusercontent.com/48702893/144172479-6aee4e23-1662-4d1f-9ec2-cc65370715f3.png)
    
    * was log
    ![image](https://user-images.githubusercontent.com/48702893/144172485-8b6e74fd-1fe8-44f8-ac96-f0b43127edbd.png)

* 결론
    * WAS가 부하를 감당하지 못하여 (e.g. peek time 시 급격한 tps 증가) 혼잡제어가 필요한 경우가 아니면 delay나 default 보단 nodelay 가 성능적으로 우수해보임

### delay
* nginx config
```
location ~ ^/delay/(.+) {
    limit_req zone=myzone burst=5 delay=3;
    set $proxy_host $host;
    proxy_pass http://app_server/$1$is_args$args;
}
```

* 테스트 결과
    * client log
    ![image](https://user-images.githubusercontent.com/48702893/144172958-0583b4e5-b4ab-4bbe-8a44-1e1cd1f93436.png)
    * nginx log
    ![image](https://user-images.githubusercontent.com/48702893/144172963-33bfcaad-4fee-4eec-b2b6-35686ee132b3.png)
    * was log
    > no request sent was

* 결론
    * [document](https://www.nginx.com/blog/rate-limiting-nginx/#Two-Stage-Rate-Limiting) 에 따르면 delay 로 설정한 수의 request 는 nodelay 로 바로 WAS 로 프록시, 그를 초과하는 요청은 delay 되어 WAS로 프록시 된다고 설명되어있으나 테스트 결과 WAS 로 프록시 없이 모든 요청에 바로 429 에러 응답
    * 추가적인 분석 필요

### dry run
* nginx config
```
limit_req_zone $binary_remote_addr zone=myzone:10m rate=12r/m;
limit_req_dry_run on;
```

* 테스트 결과
    * client log
    ![image](https://user-images.githubusercontent.com/48702893/144350619-c2e11212-8c1e-4c91-b5e7-46366c3ddd3e.png)
    * nginx log
    ![image](https://user-images.githubusercontent.com/48702893/144350628-45dc266b-dc13-45ce-b030-05f15ba057fe.png)
    
* 결론
    * dry run mode 는 rate 설정을 통한 제한 수행하지 않음 > request 큐에 들어오는 즉시 WAS 로 프록시 되기떄문에, request 큐도 의미가 없어지고 rate limiting 이 되지 않음
    * 식별버퍼로 설정한 메모리 공간이 가득 찼을때에만 에러 응답

 
