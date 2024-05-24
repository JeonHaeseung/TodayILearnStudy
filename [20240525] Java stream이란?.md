### TIL - Java stream이란?
---
먼저, Java Stream이 무엇일까? 자바에서 스트림이란 선언적 및 기능적 스타일로 처리할 수 있는 데이터 시퀀스라고 한다. 스트림은 데이터 컬렉션에 대한 필터링, 매핑 및 축소와 같은 작업을 진행할 수 있다. 나는 스트림을 주로 List로 DB에서 가져온 데이터를 DTO에 맵핑해서 반환할 때 사용했다. 이 방식으로 사용하면 List를 하나하나 for 문을 돌면서 DTO에 맵핑하지 않아도 되어서 훨씬 선언적으로 맵핑이 가능해 편리하다. 

```java
// ChatListResponseDTO에 매핑
List<GetChatDto> getChatDtos = chatList.stream()
        .map(chat -> GetChatDto.builder()
                .id(chat.getId())
                .createdDate(chat.getCreatedDate().toString())
                .text(chat.getText())
                .caseNumber(chat.getCaseNumber())
                .chatType(chat.getChatType().toString())
                .build())
        .toList();
```

그런데 스트림이라는 게 정확히 뭐길래 이런 일이 가능한 것일까? 찾아보니, 스트림은 데이터 소스를 둘러싼 `래퍼`로 해당 데이터 소스로 작업할 수 있게 하고 대량 처리를 편리하고 빠르게 만든다. 데이터를 저장하거나, 원본 데이터를 전혀 수정하지 않는다는 점에서 **자료구조가 아니다.** Java 8부터는 `Collection` 인터페이스에 `stream()` 메소드를 추가해, 위에서 본 것처럼 List에 단순히 `stream()`이라는 메소드를 사용하는 것만으로도 스트림을 얻을 수 있다.

위에서는 맵핑하기 위해 스트림을 사용했지만, 아래와 같이 필터링에도 사용 가능하다.
```java
@Test
public void whenFilterEmployees_thenGetFilteredStream() {
    Integer[] empIds = { 1, 2, 3, 4 };
    
    List<Employee> employees = Stream.of(empIds)
      .map(employeeRepository::findById)
      .filter(e -> e != null)
      .filter(e -> e.getSalary() > 200000)
      .collect(Collectors.toList());
    
    assertEquals(Arrays.asList(arrayOfEmps[2]), employees);
}
```

또한 아래처럼 sort하는 데에도 스트림을 사용할 수 있다.
```java
@Test
public void whenSortStream_thenGetSortedStream() {
    List<Employee> employees = empList.stream()
      .sorted((e1, e2) -> e1.getName().compareTo(e2.getName()))
      .collect(Collectors.toList());

    assertEquals(employees.get(0).getName(), "Bill Gates");
    assertEquals(employees.get(1).getName(), "Jeff Bezos");
    assertEquals(employees.get(2).getName(), "Mark Zuckerberg");
}
```

근데 그래서 스트림이 정확히 뭐길래 이런 일이 가능한 것일까? Java 컬렉션은 일반적으로 메모리에 데이터를 저장하고 검색는 자료 구조이고, Java Stream은 단순히 데이터 소스(를 둘러싼 랩퍼)이다. 그래서 기존의 자료구조를 스트림에 갖다 붙일 수 있다.

<img style="width: 100%;" src="https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/8f4fcd98-1cc9-40a3-954a-eb3af6f6fa60">

스트림이 만들어진 이유는 다양한 중간작업(예를 들어, 맵핑, 필터링 등)을 파이프라인처럼 이어 붙일 수 있고, 병렬화가 가능하기 때문이다. 예를 들어서 위에서 본 예시처럼 List를 DTO로 맵핑할 때 꼭 for문을 돌면서 하나하나 맵핑할 필요가 없고, 각 객체를 병렬적으로 한꺼번에 처리할 수도 있다. 스트림은 이런 상황에서 장점을 가진다.

### QnA
---
(스터디 이후 채우기)

### 레퍼런스
---
- (medium의 Java Streams)[https://medium.com/@TechiesSpot/understanding-java-streams-a-beginners-guide-5b76fbe5df98]
- (stackify의 Java 8 streams)[https://stackify.com/streams-guide-java-8/]
- (reddit의 what exactly are streams)[https://www.reddit.com/r/learnjava/comments/f3tkwa/what_exactly_are_streams/?rdt=55440]
- (geeksforgeeks의 Stream in Java)[https://www.geeksforgeeks.org/stream-in-java/]
