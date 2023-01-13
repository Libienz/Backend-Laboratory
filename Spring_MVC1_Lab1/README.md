<details>
<summary>05 Spring MVC 기본 기능 </summary>
<div markdown="1">

## 프로젝트 생성
- packaging에서 Jar와 War의 차이
  - Jar: 내장 서버를 사용(톰캣) webapp 경로 사용하지 않음. 내장 서버 사용에 최적화 되어 있다. 요즈음은 주로 Jar사용
  - War: 내장 서버도 사용 가능하지만 주로 외부 서버에 빌드 파일을 올릴 때 사용

## Logging
- 로그에 대해 간단히 알아보자
- 이제 sout이 아닌 별도의 로깅 라이브러리를 사용하여 로그를 출력할 것
- 참고로 로그 관련 라이브러리도 많고, 깊게 들어가면 끝이 없기에 최소한의 사용 방법만 알아보자

### 로깅 라이브러리
- 스프링 부트 라이브러리를 사용하면 스프링 부트 로깅 라이브러리가 함께 포함된다.
- 스프링 부트 로깅 라이브러리는 기본으로 다음 로깅 라이브러리를 사용
  - SLF4J
  - Logback
- 여러 로그 라이브러리를 통합하여 (어댑터 패턴 등등) 사용할 수 있도록 인터페이스로 제공하는 것이 SLF4J
- Logback은 로그 라이브러리 (구현체) 실무에서 Logback 많이 사용한다

### 로그 선언, 호출

```java
package hello.springmvc.basic;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RestController
public class LogTestController {

//    private final Logger log = LoggerFactory.getLogger(getClass());

    @RequestMapping("/log-test")
    public String logTest() {
        String name = "Spring";
        System.out.println("name = " + name);
        log.trace("trace log=" + name); //이렇게 쓰면 안된다 출력 안할 건데 선연산이 들어가버림
        log.debug("debug log={}", name);
        log.info("info log={}", name);
        log.warn("warn log={}", name);
        log.error("error log={}", name);

        log.info("info log={}", name);

        return "ok";
    }
}

```

- @RestController
  - @Controller는 반환 값이 String이면 뷰 이름으로 인식되어 뷰를 찾고 뷰가 렌더링 됨
  - @RestController는 반환 값으로 뷰를 찾는 것이 아니라 HTTP 메시지 바디에 바로 입력
  - @ResponseBody와 관련 있는데 뒤에서 더 자세히 볼 것임
- 로그의 출력 내용
  - 시간, 로그 레벨, 프로세스 ID, 쓰레드 명, 클래스 명, 로그 메시지
- 로그 레벨은 다음과 같다.
  - TRACE > DEBUG > INFO > WARN > ERROR
  - 로그 레벨 설정을 변경하며 노출 시킬 로그 레벨을 정할 수 있다.
  - 보통 개발 서버는 debug이상으로 심각한 로그를 출력
  - 운영 서버는 info 출력
- @Slf4j로 로그 선언 부분을 대체 할 수 있다. (롬복이 대신 써준다)

### 올바른 로그 사용법
- 선연산이 되지 않게 하자 
- log.debug("data=" + data)
  - 위와 같이 써도 로그 출력은 올바로 됨 하지만 debug로그를 노출시키지 않을 예정임에도 파라미터 연산이 먼저되어 서버의 자원을 잡아먹는다
  - 이렇게 쓰면 혼난다.
  - 다음과 같이 쓰자 log.debug("data = {}", data)
  - {}가 서식지정자 마냥 치환된다.

### 로그 사용시 장점
- 쓰레드 정보, 클래스 이름 같은 부가 정보를 함께 볼 수 있고 출력 모양을 조정 간으
- 로그 레벨에 따라 노출여부를 결정 가능
- 콘솔에만 아니라 파일, 네트워크 등 로그를 별도의 위치에 남길 수도 있음
- 특히 파일로 남길 때는 일별 특정 용량에 따라 로그를 분할하는 것도 가능
- 성능도 sout보다 파워풀하다 (내부 버퍼링, 멀티 쓰레드 등등)



</div>
</details>