---
layout: post
title: Swagger 설정
author: 이강운
subtitle: Rest Api 문서 자동화 라이브러리 Swagger 설정하기
categories: spring
tags: [spring]
---

개요
-------------

Spring에서 개발된 REST API를 자동 문서화 라이브러리 Swagger에 대한 설명이다.
Swagger는 Spring Rest doc과 더불어 대표적인 REST API 문서 라이브러리이다.
REST API는 개발자의 의도가 API를 이용하는 개발자에게 의도가 명확하게 전달해야한다.
군더더기 없이 개발을 진행하여도 이를 문서화 시켜 공개하면 API를 이용하는 개발자들은 API의 구조를 빠르게 이해할 수 있다. 하지만, 문서 작업은 또 하나의 자원을 낭비하게 된다. 본 문서는 Swagger를 통해 REST API 문서화를 진행한다.

[Swagger 공식홈페이지][1]에서 다양한 확인을 할 수 있다.

<br>

환경
-------------
> 자바 : Java 11<br>
> 스프링 : Springboot 2.5.x<br> 
> 빌드툴 : Gradle<br>
> Swagger 버전: v3.0<br>
> 기타라이브러리 : lombok, spring-security<br>

<br>

진행
-------------
Swagger는 개발된 REST API 프로젝트에 Swagger 관련 설정을 추가하는 방식이다.<br>
따라서 문서에서는 학생 REST API를 예시로 설명을 진행하겠다.

***Step.01 학생 클래스 생성***
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Student {
    private String name;    // 이름
    private String cls;     // 반
    private int score;      // 점수
}
```

***Step.02 학생 저장데이터 생성***
```java
private static List<Student> students = List.of(
        new Student("Lee", "A", 78),
        new Student("Kim", "B", 94),
        new Student("Park", "C", 59),
        new Student("Hong", "A", 59),
        new Student("Byoen", "C", 59),
        new Student("Yu", "D", 83)
);
```
*> 저장데이터의 경우 간단한 테스트를 위해 In-Memory 형식으로 지정*<br><br>

***Step.03 학생 API 생성***
```java
@RestController
@RequestMapping("/api/student")
public class StudentController {
    private static List<Student> students = List.of(
            new Student("Lee", "A", 78),
            new Student("Kim", "B", 94),
            new Student("Park", "C", 59),
            new Student("Hong", "A", 59),
            new Student("Byoen", "C", 59),
            new Student("Yu", "D", 83)
    );

    @GetMapping("/getAllStudent")
    public List<Student> getStudents() {
        return students;
    }

    @GetMapping("/{name}")
    public Student getStudent(@PathVariable("name") String name) {
        return students.stream()
                .filter(x -> x.getName().equalsIgnoreCase(name))
                .collect(Collectors.toList()).get(0);
    }

    @PatchMapping("/{name}")
    public ResponseEntity modifyStudent(@PathVariable("name") String name, @RequestBody Student student) {
        Student nameOfStudent = students.stream()
                .filter(s -> name.equals(s.getName()))
                .findFirst()
                .orElseThrow(RuntimeException::new);

        nameOfStudent.setCls(student.getCls());
        nameOfStudent.setScore(student.getScore());

        return new ResponseEntity(nameOfStudent, HttpStatus.OK);
    }

    @DeleteMapping("/{name}")
    public ResponseEntity deleteStudent(@PathVariable("name") String name) {
        boolean isDeleted = students.removeIf(s -> name.equals(s.getName()));

        return isDeleted ? new ResponseEntity(String.format("%s 삭제성공", name), HttpStatus.OK)
                    : new ResponseEntity("삭제 실패", HttpStatus.BAD_REQUEST);
    }

    @GetMapping("/byClass/{cls}")
    public ResponseEntity getStudentByClass(@PathVariable("cls") String cls) {
        List<Student> studentsbyClass = students.stream()
                .filter(x -> x.getCls().equalsIgnoreCase(cls))
                .collect(Collectors.toList());

        return new ResponseEntity(studentsbyClass, HttpStatus.OK);
    }
}
```
*[각각 전체조회], [이름으로 조회], [정보변경], [학생정보 삭제], [수업명으로 조회]를 의미한다*<br><br>

***Step.04 API 통신 확인***

![학생 전체 조회]("/assets/images/posts/swagger/swagger_01.png")

<br><br>

***Step.05 Swagger 라이브러리 추가***<br>
 API 통신을 확인하였다면 프로젝트에 Swagger라이브러리를 추가한다.

```gradle
implementation 'io.springfox:springfox-swagger2:3.0.0'
implementation 'io.springfox:springfox-swagger-ui:3.0.0'
```

라이브러리 import 확인 후 Swagger 설정을 Spring에 등록해준다.<br>

```java
@Configuration
@EnableSwagger2
public class SwaggerConfiguration implements WebMvcConfigurer {
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("학생 API 문서")
                .description("학생 API 통신과 관련된 API 설명 문서입니다.")
                .build();
    }

    @Bean
    public Docket commonApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("학생")
                .apiInfo(this.apiInfo())
                .select()
                .apis(RequestHandlerSelectors
                        .basePackage("org.study.swagger.domain"))   
                        //swagger 적용 패키지
                .build();
    }
}
```

Swagger 라이브러리 내부에 존재하는 Resource에 대한 처리도 ResourceHandler에 추가한다.<br>
```java
@Configuration
@EnableWebMvc
public class WebConfiguration implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // swagger-ui
        registry.addResourceHandler("swagger-ui.html").addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**").addResourceLocations("classpath:/META-INF/resources/webjars/");
    }
}

```
<br>
이후 서비스를 기동시켜 **[호스트명]/swagger-ui/index.html**로 접속하여 Swagger의 동작을 확인한다.<br>
본 문서는 Swagger 3.0.0으로 진행하여 /swagger-ui/index.html 이지만, 2.x.x 버전으로 테스트를 진행하는 경우 /swagger-ui.html 으로 확인해야 한다.

--
![Swagger 확인](/assets/images/_posts/swagger/swagger_02.png)
--

<br>
현재는 문서의 제목, 설명 정도만 설정하였기 때문에 자세한 REST API의 기능은 확인할 수 없다.


***Step.06 Swagger 설정***<br>
구현한 API 문서화를 위한 설정을 추가한다.

우선 StudentController에 아래와 같이 추가한다.<br>
각 어노테이션은 아래와 같다.<br>

- @Api : API에 대한 설명을 기술
- @ApiOperation : Ap의 동작 부분을 기술, notes 값을 통해 추가 기술, hiddent을 통해 숨김여부 설정
<br>

```java
@RestController
@RequestMapping("/api/student")
@Api("학생 API")
public class StudentController {
    // 컬렉션 생략

    @GetMapping("/getAll")
    @ApiOperation("전체 학생 조회")
    public List<Student> getStudents() {
        return students;
    }

    @GetMapping("/{name}")
    @ApiOperation("이름으로 학생 조회")
    public Student getStudent(@PathVariable("name") String name) {
        return students.stream()
                .filter(x -> x.getName().equalsIgnoreCase(name))
                .collect(Collectors.toList()).get(0);
    }

    @PatchMapping("/{name}")
    @ApiOperation("학생 정보 변경")
    public ResponseEntity modifyStudent(@PathVariable("name") String name, @RequestBody Student student) {
        // ...
    }

    @DeleteMapping("/{name}")
    @ApiOperation("학생 삭제")
    public ResponseEntity deleteStudent(@PathVariable("name") String name) {
        // ...
    }

    @GetMapping("/byClass/{cls}")
    @ApiOperation(value = "수업명으로 학생 조회", notes = "요청값으로 수업 정보(A, B, C, D) 요청 가능")
    public ResponseEntity getStudentByClass(@PathVariable("cls") String cls) {
        // ...
    }
}
```

학생 API는 학생(Student)를 기반으로 동작하기 때문에 모델에 대한 문서도 설정하겠다.
- @ApiModel : API 모델에 대한 설명 기술
- @ApiModelProperty : 모델의 인자값에 대한 설명 기술
<br>

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@ApiModel("학생")
public class Student {
    @ApiModelProperty(notes = "이름")
    private String name;

    @ApiModelProperty(notes = "수업")
    private String cls;

    @ApiModelProperty(notes = "점수")
    private int score;
}
```

이후 재기동하여 확인

![Swagger API 확인]({{site.url}}/assets/images/_posts/swagger/swagger_03.png)


![Swagger API Model 확인](/assets/images/_posts/swagger/swagger_04.png)

<br><br>

***번외) Swagger Security 접근 설정***<br>

Swagger 역시 경로를 기반으로 동작하는 라이브러리다. <br>
문서에 접속하는 방식과 더불어 swagger 라이브러리에 내장된 리소스에 접근한다. <br>
특히 Security가 동작하는 프로젝트의 경우 Swagger 리소스관련 경로를 예외처리 해줘야 원할하게 동작한다.

아래의 코드를 통해 swagger 관련 예외를 처리한다

1) Security ignore 
```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    private static final String[] SWAGGER_REQUEST = {
            // -- Swagger UI v2
            "/v2/api-docs",
            "/swagger-resources",
            "/swagger-resources/**",
            "/configuration/ui",
            "/configuration/security",
            "/swagger-ui.html",
            "/webjars/**",
            // -- Swagger UI v3 (OpenAPI)
            "/v3/api-docs/**",
            "/swagger-ui/**"
    };

    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring()
                .antMatchers(SWAGGER_REQUEST)
                .requestMatchers(PathRequest.toStaticResources().atCommonLocations());
    }
}
```
<br>

마치며
-------------
Swagger UI를 설정하고 확인하는 법을 작성하였다. Swagger UI 통해 REST API 문서화를 효과적으로 할 수 있다.

<br>

참고
-------------
[Swagger Official Site][1] <br>
[Baeldung - Setting Up Swagger 2 with a Spring REST API][2] <br>

[1]: https://swagger.io/tools/swagger-ui/
[2]: https://www.baeldung.com/swagger-2-documentation-for-spring-rest-api