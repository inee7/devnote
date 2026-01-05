# jojoldu spring boot
[https://github.com/jojoldu/freelec-springboot2-webservice](https://github.com/jojoldu/freelec-springboot2-webservice)

# 1. 인텔리제이로 스프링부트 시작하기

gradle로 프로젝트 만들고 스프링부트로 변경함

스프링 이니셜라이저로 하면 gradle에 대한 학습이 어려워서 .

```
근데 ext랑 def 차이가 뭘까? 여기서도 ext썼던데 차이가 뭘까?
def는 groovy의 지역변수리고 
ext는 gradle의 전역 맵 같은곳에 넣어두고 하위까지 쓸 수 있게 하는... 

```

여기서는 gradle4로 쓰고 있어서 레거시한 타입이네

mavenCentral은 업로드 난이도가 있어서 점점 jcenter로 옮겨가는 추세래

jcenter에서 mavenCentral에 자동 업로드 기능이 있다.

둘다 등록해서 사용한다.

# 2. 테스트 코드

TDD는 RGB로 실패하는 테스트코드 만들고 그다음 프로덕션 코드를 만들어 테스트 통과 시키고 프로덕션 코드를 리펙토링한다. 라는 것인데 여기선 TDD를 하지 않고 단순 유닛테스트 즉, 테스트코드 작성만 한다.

임베디드 톰캣 성능상 이슈 없음.

같은 환경의 서버로 편하게 배포 가능.

@RunWith 

-> 테스트 할때 JUnit에 내장된 실행자 외에 다른 실행자를 실행

@WebMvcTest 

-> Web에 집중하는 테스트 어노테이션. 컨트롤러만 사용하기 때문에 @Controller와 @ControllerAdvice등을 사용하고 @Service, @Component, @Repository 사용 불가

MockMvc 빈

-> 웹 API 테스트.

```java
mvc.perform(get("/hello")).andExpect(status().isOk()).andExpect(content().string("hello"));

```

junit 의 기본 asserThat는 추가적인 라이브러리가 필요하고 자동완성에 약함. [http://bit.ly/30vm9Lg](http://bit.ly/30vm9Lg)

```java
.andExpect(jsonPath("$.name", is(name)))
.andExpect(jsonPath("$.amount", is(amount)));

```

json 검증

# 3. JPA

@Entity 의 네이밍 기본값은 카멜케이스를 스네이크케이스로

> 스프링부트2.0부터는 @GeneratedValue 의 GenerationType.IDENTITY옵션 줘야 auto_increment 됨
> 

Entity클래스랑 Repository클래스랑 한 패키지에서 관리하는게 좋음 밀접한 관계이다. 

spring-data-jpa, h2 쓰기위해 디펜던시 추가 

```
compile('org.springframework.boot:spring-boot-starter-data-jpa')
compile('com.h2database:h2')
```

**도메인 패키지 만들고 엔티티 생성**

```java
@Getter
@NoArgsConstructor
@Entity
public class Posts {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(length = 500, nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT", nullable = false)
    private String content;

    private String author;

    @Builder // 생성자 위에 빌더면 해당 필드만 빌더로 생성 
    public Posts(String title, String content, String author) {
        this.title = title;
        this.content = content;
        this.author = author;
    }

}

```

**통합테스트 실행** 

```java

@SpringBootTest
@RunWith(SpringRunner.class)
public class PostsRepositoryTest {

    @Autowired PostsRepository postsRepository;

    @After
    public void cleanUp() {
        postsRepository.deleteAll();
    }

    @Test
    public void 게시글저장_불러오기() {
        String title = "테스트 게시글";
        String content = "테스트 본문";

        postsRepository.save(Posts.builder().title(title).content(content).author("jongin@gmail.com").build());

        List<Posts> postsList = postsRepository.findAll();

        Posts posts = postsList.get(0);
        assertThat(posts.getTitle()).isEqualTo(title);
        assertThat(posts.getContent()).isEqualTo(content);
    }
}
```

**spring jpa 설정**

```
spring.jpa.show-sql=true //쿼리 볼수 있게 
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect
// h2는 h2문법으로 쿼리 실행하는데 위에 같이 설정하면 MySql의 문법으로 실행 
```

**게시글 등록 api**

controller → service → repository

**api 통합 테스트** 

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class PostsApiControllerTest {
    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private PostsRepository postsRepository;

    @After
    public void tearDown() throws Exception {
        postsRepository.deleteAll();
    }

    @Test
    public void Posts_등록() {
        String title = "title";
        String content = "content";
        PostsSaveRequestDto postsSaveRequestDto = PostsSaveRequestDto.builder().title(title).content(content).author("author").build();

        String url = "http://localhost:" + port + "/api/v1/posts";

        ResponseEntity<Long> responseEntity = restTemplate.postForEntity(url, postsSaveRequestDto, Long.class);

        assertThat(responseEntity.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(responseEntity.getBody()).isGreaterThan(0L);

        List<Posts> all = postsRepository.findAll();
        assertThat(all.get(0).getTitle()).isEqualTo(title);
        assertThat(all.get(0).getContent()).isEqualTo(content);
    }
}
```

> Controller, ControllerAdvice, 외부 연동 api 테스트는 @WebMvcTest ,

통합 테스트하려면 @SpringBootTest + TestRest
> 

**수정,조회 api 개발 후 통합 테스트**

**h2 콘솔 설정 추가** 

`spring.h2.console.enabled=true`

**JPA Auditing**

java8 에 LocalDateTime을 써야되는이유는..
기존에 Date와 Calendar가 불변이 아니었고 Calendar 월 값 설계가 잘 못 되었다. 

LocalDate(Time)이 DB에 제대로 매핑되는건 Hibernate 5.2.10버전 부터!  (스프링1.x면 그 아래 버전이라 따로 이상 버전을 설정해야함)

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseTimeEntity {
    
    @CreatedDate
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updateAt;
}

```

@MappedSuperclass : 상속하면 해당 필드도 컬럼으로 인식 

@EntityListeners : 기능 포함, 여기서는 Auditing 기능 추가 

```java
@EnableJpaAuditing 
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

@EnableJpaAuditing를 활성화 시켜야된다. 

```java
@Test
public void BaseTimeEntity_등록() {
    LocalDateTime localDateTime = LocalDateTime.of(2019, 6, 4, 0, 0, 0);
    Posts posts = postsRepository.save(Posts.builder().title("title").content("content").author("author").build());
    assertThat(posts.getCreatedAt()).isAfter(localDateTime);
    assertThat(posts.getUpdateAt()).isAfter(localDateTime);
}
```

# 4. 화면

서버사이드템플릿으로 뷰를 개발. 

그중 머스테치로 개발. 

머스테치는 다양한 언어를 지원. Mustache.js, [Mustache.java](http://mustache.java) ...

```groovy
compile('org.springframework.boot::spring-boot-starter-mustache')
```

의존 추가 하고 플러그인 설치

src/main/resources/templates에 index.mustache 파일을 두면 스프링부트가 인식하고 로딩됨 

CDN방식으로 부트스트랩, 제이쿼리 추가 

페이징 로딩 속도를 위해서는 header에는 css, footer에 js

파일을 순차적으로 해석하는데 head에 로딩이 오래 걸리면 화면이 백지화 될 수 있다. head→body 

spring에서 resource/static 부터는 절대경로로 지정 가능 

삭제는 귀찮아서 생략 

# 5. 시큐리티

OAuth 쓰면 

* 로그인 시 보안
* 비밀번호찾기
* 회원정보 변경

개발 안해도 됨

스프링부트1.5 와 2.0이 OAuth 연동 방법에서 아주 큰 차이가 있다고 한다. 

spring-security-oauth2-autoconfigure을 쓰면 스프링2에서 OAuth2 1.0 처럼 쓸 수 있다. 

Oauth2 위해 구글클라우드콘솔에서 클라이언트ID 만듬 

id: [440507870772-8ac8ao5o4hcfnkfto70dopndt3esjg5q.apps.googleusercontent.com](http://440507870772-8ac8ao5o4hcfnkfto70dopndt3esjg5q.apps.googleusercontent.com/)

보안 비밀 : K_q5HQHso6Qaan_gIIIVAFhI

[https://console.cloud.google.com/apis/credentials/oauthclient/440507870772-8ac8ao5o4hcfnkfto70dopndt3esjg5q.apps.googleusercontent.com?project=scenic-crossbar-183401](https://console.cloud.google.com/apis/credentials/oauthclient/440507870772-8ac8ao5o4hcfnkfto70dopndt3esjg5q.apps.googleusercontent.com?project=scenic-crossbar-183401)

scope에 open id를 빼면 구글,네이버,카카오 마다 OAuth2Service를 안 만들어도 된다. 

```groovy
compile('org.springframework.boot:spring-boot-starter-oauth2-client')
시큐리티 의존성도 같이 담고 있더라. 
```

```java
@RequiredArgsConstructor
@EnableWebSecurity //시큐리티 설정하는 클래스라고 지정
public class SecurityConfig extends WebSecurityConfigurerAdapter { //시큐리티 설정 클래스 상속
    private final CustomOAuth2UserService customOauth2UserService;
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
                .headers().frameOptions().disable() // h2console에서 무시해야될 
                .and().authorizeRequests() // url별 권한 관리 
                .antMatchers("/", "/css/**", "/images/**", "/js/**", "/h2-console/**").permitAll() // 해당 url 모두 허용
                .antMatchers("/api/v1/**").hasRole(User.Role.USER.name()) //권한
                .anyRequest().authenticated()// 나머지 인증(로그인)까지만
                .and().logout().logoutSuccessUrl("/")//로그아웃시 
                .and().oauth2Login().userInfoEndpoint().userService(customOauth2UserService);// 소셜 로그인 시 후속 조치 
    }
}
```

**로그인 후속 처리 CustomOAuth2UserService**

엔티티에 직렬화를 했는데 연관관계인 자식이 있다면 성능이슈와 부수효과가 발생할 가능성이 높다.

```java
@RequiredArgsConstructor
@Service
public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {
    private final UserRepository userRepository;
    private final HttpSession httpSession;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2UserService delegate = new DefaultOAuth2UserService();
        OAuth2User oAuth2User = delegate.loadUser(userRequest);

        String registrationId = userRequest.getClientRegistration().getRegistrationId(); //서비스 별 id

        String userNameAttributeName = userRequest.getClientRegistration().getProviderDetails().getUserInfoEndpoint().getUserNameAttributeName(); // OAuth2 로그인 진행 시 키가 되는 필드

        OAuthAttributes oAuthAttributes // 로그인 후 oAuth2User의 attribute를 담을 클래스
                = OAuthAttributes.of(registrationId, userNameAttributeName, oAuth2User.getAttributes());

        User user = mergeUser(oAuthAttributes); //로그인 정보와 내 앱의 유저 정보와 동기화

        httpSession.setAttribute("user", new SessionUser(user)); // 세션에 저장 , 여기서 직렬화 객체여야 한다. 근데 엔티티를 직렬화하면 성능이슈와 부수효과가 있음

        return new DefaultOAuth2User(
                Collections.singleton(new SimpleGrantedAuthority(user.getRoleKey())), oAuthAttributes.getAttributes(), oAuthAttributes.getNameAttributeKey()
        );
    }

    private User mergeUser(OAuthAttributes oAuthAttributes) {
        User user = userRepository.findByEmail(oAuthAttributes.getEmail())
                .map(u -> u.update(oAuthAttributes.getName(), oAuthAttributes.getPicture()))
                .orElse(oAuthAttributes.toEntity());
        return userRepository.save(user);
    }
}
```

**로그인 테스트**

```html
<div class="row">
    <div class="col-md-6">
        <a href="/posts/save" role="button" class="btn btn-primary">글 등록</a>
        {{#userName}} // if문이 없어 #로 값이 있다면 
            Logged in as : <span id="user">{{userName}}</span>
                <a href="/logout" class="btn btn-info active" role="button">로그아웃</a>
        {{/userName}}
        {{^userName}}// 값이 없다면
            <a href="/oauth2/authorization/google" class="btn btn-success active" role="button">구글 로그인</a>
        {{/userName}}
    </div>
</div>
```

시큐리티에서 기본적으로 logout url을 제공 

/oauth2/authorization/google 도 기본적으로 제공

controller 

```java

@GetMapping("/")
public String index(Model model) {
    model.addAttribute("posts",postsService.findAllDesc());
    User user = (User)httpSession.getAttribute("user");
    if (user != null) {
        model.addAttribute("userName", user.getName());
    }
    return "index";
}
```

HttpSession을 빈으로 가져올수 있넹

**메소드 인자 가져와서 처리하는 어노테이션 개발** 

* 어노테이션 생성
* HandlerMethodArgumentResolver 인터페이스는 조건에 맞는 메소드가 있으면 파라미터로 넘길 수 있다.
* WebMvcConfigurer에 Resolver 추가


  ```java
  @RequiredArgsConstructor
  @Configuration
  public class WebConfig implements WebMvcConfigurer {
      private final LoginUserArgumentResolver loginUserArgumentResolver;
      
      @Override
      public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
          resolvers.add(loginUserArgumentResolver);
      }
  }	
  ```


**세션 날라가는거 방지를 위해** 

톰캣세션사용

* 여러 was에서 세션 공유 가능

MySql

* DB IO가 발생해서 로그인 요청 적은 어드민에서 사용
* 비용 적음

 Redis

* B2C에서 많이 사용한다 (?)
* 비용이 듬

MySql방식으로 해본다 

`compile('org.springframework.session:spring-session-jdbc')` 추가 

spring.session.store-type=jdbc 설정 추가 

그럼 알아서 세션위한 테이블 본다 (SPRING_SESSION, SPRING_SESSION_ATTRIBUTES)

**네이버 로그인 등록**

[https://developers.naver.com/apps/#/myapps/QnaPSxh4QB8XGxyfjplm/overview](https://developers.naver.com/apps/#/myapps/QnaPSxh4QB8XGxyfjplm/overview)

client 정보 넣고 

네이버가 스프링시큐리티 정식 지원하지 않기에 전부 수동으로 입력

**테스트 코드에 시큐리티 설정**

기본적인 설정파일은 테스트에 없으면 main꺼를 따라가지만 부가적인 설정은 따라가지 않는다. application-auth.properties

test 위한 시큐리티 의존성 `testCompile('org.springframework.security:spring-security-test')`

MockMvc에서 목롤 `@WithMockUser(roles = "USER")`

SpringBootTest에서 MockMvc 사용하는 방법

```java
@Autowired
    private WebApplicationContext webApplicationContext;
    private MockMvc mockMvc;

    @Before
    public void setUp() throws Exception {
        ...
        mockMvc = MockMvcBuilders
                .webAppContextSetup(webApplicationContext)
                .apply(SecurityMockMvcConfigurers.springSecurity())
                .build();
    }

	@Test
    @WithMockUser(roles = "USER")
    public void Posts_등록() throws Exception {
        String title = "title";
        String content = "content";
        PostsRequestV1 postsRequestV1 = PostsRequestV1.builder().title(title).content(content).author("author").build();

        String url = host + "/api/v1/posts";

        String result = mockMvc.perform(post(url)
                .contentType(MediaType.APPLICATION_JSON_UTF8)
                .content(new ObjectMapper().writeValueAsString(postsRequestV1)))
                .andExpect(status().isOk())
                .andReturn().getResponse().getContentAsString();

        Posts post = postsRepository.findById(Long.valueOf(result)).get();
        assertThat(post.getTitle()).isEqualTo(title);
        assertThat(post.getContent()).isEqualTo(content);
    }

```

@WebMvcTest 는 config관련 클래스 controller를 읽어서 나머지 빈에 대한 스캔이 없다. 

따라서 securityConfig는 읽지만 CustomOAuth2UserService는 읽을수가 없다

그래서 securityConfig를 스캔 대상에서 제거 

하지만 @EnableJpaAuditing을 사용할때 @Entity가 없게 되니 문제가 발생 

그래서 @SpringBootApplication쪽에서 @EnableJpaAuditing을 config 분리 

# 6. AWS

새 메일 계정으로 만듬

1년간 ec2 가능 

서울리전 선택 

인스턴스 시작 

AMI라는 이미지를 선택해야한다. 

아마존리눅스1을 선택

ec2 인스턴스 중단하고 다시 시작하면 ip 바뀐다 

고정 ip를 가지게 하기 위해 EIP를 설정 (탄력적IP)

15.165.118.82

EC2 서버 ssh 접속 위해 ssh 설정 

perm키를 ~/.ssh로 복사 

perm키를 600으로 권한 설정

config파일 생성해서 

```java
Host springboot-study
    HostName 15.165.118.82
    user ec2-user
    IdentityFile ~/.ssh/springboot-study.pem
```

넣고 700모드로 권한 설정 

그리고 `ssh springboot-study`

java8 설치 (기본적으로 java7 설치되어있네 지정 8으로 하고 7은 지워야해)

타임존 한국시간 KST로 변경 

hostname 설정

## 7. RDS 셋팅

DB 인스턴스 이름 : springboot-study-instance

마스터 이름 : admin 비번: 알고있는 그거 

 퍼블릭 예

db 이름 : jdb

파라미터그룹 생성

time zone 을 asian으로 

char set 설정 

utf8mb4는 이모지 저장 가능여부 포함 

max connection 설정 

파라미터그룹 변경 

rds 에  인바운드 추가 

ec2 보안그룹id를 넣고 내 pc ip로 추가 

접속 url (엔드포인트) : [springboot-study-instance.cgflyi7ltsjk.ap-northeast-2.rds.amazonaws.com](http://springboot-study-instance.cgflyi7ltsjk.ap-northeast-2.rds.amazonaws.com/):3306

character_set_database

collation_connection

```sql
alter database jdb
character set = 'utf8mb4'
collate = 'utf8mb4_general_ci';
```

```sql
select @@time_zone, now();

create table test (
    id bigint(20) not null auto_increment,
    content varchar(255) default null,
    primary key (id)
)engine=InnoDB;

insert into test (content)
values ('한글');
```

ec2에서 rds 접근 

```bash
> ssh springboot-study
> sudo yum install mysql #mysql-cli 설치 
> mysql -u admin -p -h springboot-study-instance.cgflyi7ltsjk.ap-northeast-2.rds.amazonaws.com
```

## 8. EC2에 배포

```bash
sudo yum install git
~/app/step1 에 clone
https로 클론 
./gradlew test
```

deploy.sh

```bash
PROJECT_NAME=springboot-study

cd $REPOSITORY/$PROJECT_NAME/

echo "> Git Pull..."

git pull

echo "> 프로젝트 Build 시작..."

./gradlew build

echo "> step1 디렉토리로 이동..."

cd $REPOSITORY

echo "> Build 파일 복사..."

cp $REPOSITORY/$PROJECT_NAME/build/libs/*.jar $REPOSITORY/

echo "> 현재 구동중인 애플리케이션 pid 확인... "

CURRENT_PID=$(pgrep -f ${PROJECT_NAME}*.jar)

echo "현재 구동중인 애플리케이션 pid: $CURRENT_PID"

if [ -z "$CURRENT_PID" ]; then
        echo "> 현재 구동 중인 애플리케이션이 없으므로 종료하지 않습니다."
else
    echo "> kill -15 $CURRENT_PID"
    kill -15 $CURRENT_PID
    sleep 5
fi

echo "> 새 애플리케이션 배포 ..."

JAR_NAME=$(ls -tr $REPOSITORY/ | grep *.jar | tail -n 1)

echo "> JAR Name : $JAR_NAME"

nohup java -jar $REPOSITORY/$JAR_NAME 2>&1 &
```

pgrep : process id 추출 

pgrep -f 프로세스이름 : 프로세스 이름으로 process id 추출 

 

[비교 연산자(이진)](https://wiki.kldp.org/HOWTO/html/Adv-Bash-Scr-HOWTO/comparison-ops.html)

쉘 if-else-fi

nohub 해줘야 터미널 접속 끊겨도 계속 구동함 

https로 하니 아이디 비번 해줘야 되서 ssh로 바꿈 

ssh-key등록 

소유자 설정 변경 

chown ec2-user /*

서버에 프로퍼티 생성

```bash
vim /home/ec2-user/app/application-oauth.properties
```

실행 [deploy.sh](http://deploy.sh) 수정 

```bash
nohup java -jar -Dspring.config.location=classpath:/application.properties, /home/ec2-user/app/application-oauth.properties $REPOSITORY/$JAR_NAME 2>&1 &
```

RDS 테이블 생성 

서비스테이블은 ddl을 로그를 통해 확인하고? schmea-mysql.sql 파일로 스프링세션테이블을 보면 나온다 

프로젝트에 mariaDB 드라이버 등록 

```bash
compile('org.mariadb.jdbc::mariadb-java-client')
```

[application-real.properties](http://application-real.properties) 추가 

```bash
spring.profiles.include=oauth,real-db
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect
spring.session.store-type=jdbc
```

실제 real-db에 대한 정보는 서버에 설정 

[application-real-db.properties](http://application-real-db.properties) 서버에 생성하고 db정보 입력 

[deploy.sh](http://deploy.sh)  에 설정 추가 

```bash
#! /bin/bash

REPOSITORY=/home/ec2-user/app/step1
PROJECT_NAME=study-springboot2-webservice

echo "작업 디렉토리 이동"
cd $REPOSITORY/$PROJECT_NAME/

echo "소스 업데이트"
echo `git pull`

echo "빌드"
echo `./gradlew build -x test`

cd $REPOSITORY

echo "실행 파일 복사"
cp $REPOSITORY/$PROJECT_NAME/build/libs/*.jar $REPOSITORY/

echo "이전에 실행 했는지 확인"

CURRENT_PID=$(pgrep -f ${PROJECT_NAME}*.jar)

if [ -z "$CURRENT_PID" ]; then
        echo "현재 구동 중인 애플리케이션이 없으므로 종료하지 않습니다."
else

    echo "이전에 구동 중이었던 PID: $CURRENT_PID 종료"
    kill -15 $CURRENT_PID
    sleep 5
fi

JAR_NAME=$(ls -tr $REPOSITORY/ | grep *.jar | tail -n 1)

echo "새 애플리케이션 배포 ... JAR Name : $JAR_NAME"

nohup java -jar -Dspring.config.location=classpath:/application.properties,/home/ec2-user/app/step1/property/application-oauth.properties,/home/ec2-user/app/step1/property/application-real-db.properties -Dspring.profiles.active=real $REPOSITORY/$JAR_NAME 2>&1 &
```

classpath:/application.properties,/home/ec2-user/app/step1/property/application-oauth.properties,/home/ec2-user/app/step1/property/application-real-db.properties

프로퍼티 사이에 띄어쓰기 하면 안됨.

ec2도메인 : [ec2-15-165-118-82.ap-northeast-2.compute.amazonaws.com](http://ec2-15-165-118-82.ap-northeast-2.compute.amazonaws.com/)

OSS에 EC2 등록 

## 9. CI배포

git push 되면 자동으로 테스트와 빌드가 되는 과정 → CI

운영서버에 자동으로 무중단 배포까지 진행되는게 → CD

마틴파울러의 CI 4가지 규칙 

* 모든 소스 코드가 현재 실행되고 누구든 현재의 소스에 접근할 수 있는 단일 지점을 유지
* 빌드 프로세스를 자동화해서 누구든 소스로부터 시스템을 빌드하는 단일 명령어를 사용할 수 있게 할 것
* 테스팅을 자동화해서 단일 명령어로 언제든지 시스템에 대한 건전한 테스트 수트를 실행할 수 있게 할 것
* 누구나 현재 실행 파일을 얻으면 지금까지 가장 완전한 실행 파일을 얻었다는 확인을 하게 할 것

> 백명석 클린코더스 TDD 추천
> 

### Travis CI

젠킨스는 설치해야해서 서버 하나 더 필요함 

웹서비스인 Tarvis가 편함 

aws에는 CodeBuild가 있지만 비용이 듬 

travis-ci.org에서 깃헙 로그인 → travis-ci.com으로 가야 private repo 가능
#dev/book