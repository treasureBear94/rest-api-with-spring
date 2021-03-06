# rest-api-with-spring
인프런 백기선님의 강의 (https://www.inflearn.com/course/spring_rest-api/dashboard)

테스트 작성 -> 코드 작성

---

controller 에서 repository 를 사용하고 
@WebMvcTest 만 있는 EventControllerTests에서 테스트를 실행하면 repository가 Bean으로 등록되지 않았다는 에러가 나온다.  

왜냐하면 @WebMvcTest는 웹용 슬라이스 테스트이기 때문에 repository 를 Bean으로 등록해주지 않는다. 
그래서 @MockBean을 이용해서 repository 를 등록해준다. 

이러면 또 다른 에러가 나는데, 이 repository는 Mocking된 것이기 때문에 save 등 어떤 동작을 수행해도 결과가 null로 나온다. 
우리는 객체에서 id를 가져오라고 하고 있기 때문에, NPE가 발생한다.

이를 위해, 실제로 어떻게 동작하라고, save를 했을 떄 어떻게 동작하라고 지정해주어야 한다. 
```java
Mockito.when(eventRepository.save(event)).thenReturn(event);
```
id도 넣어주어야 한다. 

---

modelMapper 적용했더니 다시 NPE 발생.
이유는 controller 에서 저장하려고 하는 객체는 modelMapper 를 통해 새로 만든 객체. 
그래서 test에서 만든 Event객체와 같지 않아. 그래서 repository mocking이 적용되지 않았고, 기본적으로 제공하는 null이 넘어갔고, 그래서 newEvent.getId() 할 때 NPE 가 발생한 것.

해결해 주기 위해서 mock을 안함. 그럴라면 슬라이스 테스트가 아니여야 함. @SpringBootTest 적용
이제 실제 repository를 사용해서 동작 -> 그러면 test에 넣어준 값(event.build로 만든)들은 무시가 됨 .

---

옳지 않은 값을 주면 bad request 응답.

spring boot 에서 제공하는 설정

```properties
spring.jackson.deserialization.fail-on-unknown-properties=true
```

---

에러를 내려주고 싶은데 아래처럼 body에 넣어주면 될 것 같은데 안됨 
```java
if(errors.hasErrors()) {
    return ResponseEntity.badRequest().body(errors);
}
```
BeanSerializer 에러 발생. errors 는 json으로 변환이 안됨. 

왜 event는 잘 넘어갔는데 errors는 에러가 발생할까? 

우리가 만들어준 객체인 Event는 javaBean 스펙을 따르기 때문에 ObjectMapper의 BeanSerializer 를 통해 직렬화할 수 있음.

errors 는 javaBean 스펙을 따르지 않기 떄문에 json으로 변환을 시도(MediaTypes.HAL_JSON_VALUE)하지만, 변환할 수 없음.

해결을 위해 ErrorsSerializer 구현 .

구현이 다 되었으면 이 ErrorsSerializer 를 ObjectMapper에 등록해줘야 함. -> 스프링 부트가 제공하는 @JsonComponent 사용하면 됨. 

---

test 코드 refactoring 하기 

### JUnit4 에서 .. 

JUnitParams 의존성 추가 
```xml
<dependency>
    <groupId>pl.pragmatists</groupId>
    <artifactId>JUnitParams</artifactId>
    <version>1.1.1</version>
    <scope>test</scope>
</dependency>
```

Test class에 RunWith 추가
```java
@RunWith(JUnitParamsRunner.class)
```

```java
class Test {

    @Test
    @Parameters({
            "0, 0, true",
            "100, 0, false",
            "0, 100, false"
    })
    void testFree(int basePrice, int maxPrice, boolean isFree) {
        // Given
        Event event = Event.builder()
                .basePrice(basePrice)
                .maxPrice(maxPrice)
                .build();

        // When
        event.update();

        // Then
        assertThat(event.isFree()).isEqualTo(isFree);
    }
}
```

좀 더 type-safe 하게
```java
class Test {

    @Test
    @Parameters(method = "paramsForTestFree")
    void testFree(int basePrice, int maxPrice, boolean isFree) {
        // Given
        Event event = Event.builder()
                .basePrice(basePrice)
                .maxPrice(maxPrice)
                .build();

        // When
        event.update();

        // Then
        assertThat(event.isFree()).isEqualTo(isFree);
    }

    private Object[] parametersForTestFree() {
        return new Object[]{
                new Object[]{0, 0, true},
                new Object[]{100, 0, false},
                new Object[]{0, 100, false},
                new Object[]{100, 200, false}
        };
    }
}
```

`parametersFor`뒤에 메서드 이름을 지정하면 `@Parameters` 만 써도 찾아서 적용된다.  


### JUnit5 에서 ... 

파라미터 테스트하고자 하는 테스트 위에 `@ParameterizedTest` 붙여주고, 메서드 이용해서 argument 넘길 시 `@MethodSource` 뒤에 메서드 이름을 적어주면 된다. 

@MethodSource 에 넘기는 메서드는 `static` 이여야하고 `Stream<Argumnets>` 를 리턴해야 한다. 

```java
class Test {

    @ParameterizedTest
    @MethodSource("parametersForTestFree")
    void testFree(int basePrice, int maxPrice, boolean isFree) {
        // Given
        Event event = Event.builder()
                .basePrice(basePrice)
                .maxPrice(maxPrice)
                .build();

        // When
        event.update();

        // Then
        assertThat(event.isFree()).isEqualTo(isFree);
    }

    private static Stream<Arguments> parametersForTestFree() {
        return Stream.of(
                Arguments.of(0, 0, true),
                Arguments.of(100, 0, false),
                Arguments.of(0, 100, false),
                Arguments.of(100, 200, false)
        );
    }
}
```

---

Controller test에  rest docs 적용하는데 links를 설정했음에도 불구하고 responseFields 다시 체크하라고 에러남.
links에 설정해줬으니까 될 줄 알았는데 안됨 

방법 1. `responseFields` 대신 `relaxedResponseFields` 사용. relaxedResponseFields 는 문서 일부분만 체크할 수 있어서 정확한 문서생성이 되지 않음.

방법 2. links 를 다시 `responseeFields에 채움`  
