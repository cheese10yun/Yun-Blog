---
layout: post
title: Spring Batch 간단 정리
subtitle: Sprong Batch 기초를 알아보자
catalog: true
header-img: 'https://i.imgur.com/avC1Xor.jpg'
tags:
- Spring Batch
date: 2018-11-20 01:12:00
---
> 출저 [처음으로 배우는 스프링 부트 2](http://www.kyobobook.co.kr/product/detailViewKor.laf?ejkGb=KOR&mallGb=KOR&barcode=9791162241264&orderClick=LAA&Kc=)을 보고 정리한 포스팅입니다. 배치 관련된 국내 서적 중에서 스프링 배치를 가장 잘 정리 한 거 같습니다.

스프링 배치는 벡엔드의 배치처리 기능을 구현하는 데 사용하는 프레임워크입니다. 스프링 부트 배치는 스프링 배치 설정 요소들을 간편화시켜 스프링 배치를 빠르게 설정하는 데 도움을 줍니다.

## 스프링 부트 배치의 장점
* 대용량 데이터 처리에 최적화되어 고성능을 발휘합니다.
* 효과적인 로깅, 통계 처리, 트랜잭션 관리 등 재사용 가능한 필수 기능을 지원합니다.
* 수동으로 처리하지 않도록 자동화되어 있습니다.
* 예외사항과 비정상 동작에 대한 방어 기능이 있습니다.
* 스프링 부트 배치는 반복적인 작업 프로세스를 이해하면 비니지스로직에 집중할 수 있습니다.

## 스프링 부트 배치 주의사항
스프링 부트 배치는 스프링 배치를 간편하게 사용 할 수 있게 래핑한 프로젝트입니다. 따라서 스프링 부트 배치와 스프링 배치에 모두에서 다음과 같은 주의사항을 염두해야 합니다.

* 가능하면 단순화해서 복잡한 구조와 로직을 피해야합니다.
* 데이터를 직접 사용하는 편이 빈번하게 일어나므로 데이터 무결성을 우지하는데 유효성 검사 등의 방어책이 있어야합니다.
* 배치 처리 시스템 I/O 사용을 최소화해야합니다. 잦은 I/O로 데이터베이스 커넥션과 네트워크 비용이 커지면 성능에 영향을 줄 수 있기 때문입니다. 따라서 가능하면 한번에 데이터를 조회하여 메모리에 저장해두고 처리를 한 다음. 그결과를 한번에 데이터베이스에 저장하는것이 좋습니다.
* 일반적으로 같은 서비스에 사용되는 웹 API, 배치, 기타 프로젝트들을 서로 영향을 줍니다. 따라서 배치 처리가 진행되는 동안 다른 프로젝트 요소에 영향을 주는 경우가 없는지 주의를 기울여야합니다.
* 스프링 부트는 배치 스케쥴러를 제공하지 않습니다. 따라서 배치 처리 기능만 제공하여 스케쥴링 기능은 스프링에서 제공하는 쿼치 프레임워크 등을 이용해야합니다. **리눅스 crontab 명령은 가장 간단히 사용 할 수 있지만 이는 추천하지 않습니다.** crontab의 경우 각 서버마다 따로 스케쥴러를 관리해야 하며 무엇보다 클러스터링 기능이 제공되지 않습니다. 반면에 쿼츠 같은 스케쥴링은 프레임워크를 사용한다면 클러스터링뿐만 아니라 다양한 스케쥴링 기능, 실행 이력 관리 등 여러 이점을 얻을 수 있습니다.

## 스프링 부트 배치 이해하기
배치의 일반적인 시나리오는 다음과 같은 3단계로 이루어집니다.

1. 읽기(read) : 데이터 저장소(일반적으로 데이터베이스)에서 특정 데이터 레코드를 읽습니다.
2. 처리(processing) : 원하는 방식으로 데이터 가공/처리 합니다.
3. 쓰기(write) : 수정된 데이터를 다시 저장소(데이터베이스)에 저장합니다.

배치 처리는 읽기 -> 처리 -> 쓰기 흐름을 갖습니다. 다음 그림은 스프링에서 이러한 배치 처리를 어떻게 구현 했는지 배치 처리와 관련된 객체의 관계를 보여줍니다.

<p align="center">
  <img src="https://github.com/cheese10yun/TIL/raw/master/assets/batch-obejct-relrationship.png">
</p>

* Job과 Step은 1:M
* Step과 ItemReader, ItemProcessor, ItemWriter 1:1
* Job이라는 하나의 큰 일감(Job)에 여러 단계(Step)을 두고, 각 단계를 배치의 기본 흐름대로 구성합니다.

### Job
* Job은 배치 처리 과정을 하나의 단위로 만들어 포현한 객체입니다. 또한 전체 배치 처리에 있어 항상 최상단 계층에 있습니다.
* 위에서 하나의 Job(일감) 안에는 여러 Step(단계)이 있다고 설명했던 바와 같이 **스프링 배치에서 Job 객체는 여러 Step 인스턴스를 포함하는 컨테이너 입니다**
* Job 객체를 만드는 빌더는 여러 개 있습니다. 여러 빌더를 통합하여 처리하는 공장인 JobBuilderFactory로 원하는 Job을 쉽게 만들수 있습니다.

```java
public class JobBuilderFactory {
    private JobRepostiroy jobrepository;

    public JobBuilderFactory(JobRepository jobRepository){
        this.jobrepository = jobrepository;
    }

    public JobBuilder get(String name){
        JobBuilder builder = new JobBuilder(name).repository(jobrepository);
        return builder;
    }
}
```
* JobBuilderFactory는 JobBuilder를 생성할 수 있는 get() 메서드를 포함하고 있습니다. get()메서드는 새로운 JobBuilder를 생성해서 반환하는 것을 확인할 수있습니다.
* JobBuilderFactory에서 생성되는 모든 JobBulder가 레포지토리를 사용합니다.
* JobBuilderFactory는 JobBuilder를 생성하는 역할만 수행합니다. 이렇게 생성된 JobBuilder를 이용해서 Job을 생성해야 하는데, 그렇다면 JobBuilder의 역할은 무엇인지 JobBuilder의 메서드를 통해 기능을 알아보겠습니다.

```java
public SimpleJobBuilder start(Step step){
    //(1)
    // Step을 추가해서 가장 기본이되는 SimpleJobBuilder를 생성합니다.
    return new SimpleJobBuilder(tihs).start(step);
}

public JobFlowBuilder start(Flow flow){
    //(2)
    // Flow를 실행할 JobFlowBuilder를 생성합니다.
    return new JobFlowBuilder(tihs).start(flow);
}

public JobFlowBuilder flow(Step step){
    //(3)
    // Step을 실행할 FlowJobBuilder를 생성합니다.
    return new JobFlowBuilder(tihs).start(step);
}
```
* JobBuilder는 직접적으로 Job을 생성하는 것이 아니라 별도의 구체적 빌더를 생성하여 변환하여 경우에 따라 Job 생성 방법이 모두 다를 수 있는 점을 유연하게 처리할 수 있습니다.

### JobInstance
* **JobInstance는 배치 처리에서 Job이 실행될 때 하나의 Job 실행 단위입니다.** 만약 하루에 한 번 씩 배치의 Job이 실행된다면 어제와 오늘 실행 각각 Job을 JobInstance라고 부를 수 있습니다.
* 각각의 JobInstance는 하나의 JobExecution을 갖는 것은아닙니다. 오늘 Job이 실행 했는데 실패했다면 다음날 동일한 JobInstance를 가지고 또 실행합니다.
* Job 실행이 실패하면 JobInstance가 끝난것으로 간주하지 않기 때문입니다. 그렇다면 JobInstance는 어제 실패한 JobExecution과 오늘의 성공한 JobExecution 두 개를 가지게 됩니다. **즉 JobExecution 는 여러 개 가질 수 있습니다.**

### JobExecution
* JobExecution은 JobIstance에 대한 한 번의 실행을 나타내는 객체입니다.
* 만약 오늘 Job이 실패해 내일 다시 동일한 Job을 실행하면 오늘/내일의 실행 모두 같은 JobInstance를 사용합니다.
* 실제로 JobExecution 인터페이스를 보면 Job 실행에 대한 정보를 담고 있는 도메인 객체가 있습니다. *JobExecution은 JobInstance, 배치 실행 상태, 시작 시간, 끝난 시간, 실패했을 때 메시지 등의 정보를 담고 있습니다. JobExecution 객체 안에 어떤 실행 정보를 포함 하고 있습니다.*

### JobParameters
* JobParameters는 Job이 실행될 때 필요한 파라미터들은 Map 타입으로 지정하는 객체 입니다.
* JobParameters는 JobInstance를 구분하는 기준이 되기도 합니다.
* JobParameters와 JobInstance는 1:1 관계입니다.

### Step
* Step은 실직적인 배치 처리를 정의하고 제어 하는데 필요한 모든 정보가 있는 도메인 객체입니다. Job을 처리하는 실질적인 단위로 쓰입니다.
* 모든 Job에는 1개 이상의 Step이 있어야 합니다.

#### StepExecution
* Job에 JobExecution Job실행 정보가 있다면 Step에는 StepExecution이라는 Step 실행 정보를 담는 객체가 있습니다.

### JobRepository
* JobRepository는 배치 처리 정보를 담고 있는 매커니즘입니다. 어떤 Job이 실행되었으면 몇 번 실행되었고 언제 끝났는지 등 배치 처리에 대한 메타데이터를 저장합니다.
* 예를들어 Job 하나가 실행되면 JobRepository에서는 배치 실행에 관련된 정보를 담고 있는 도메인 JobExecution을 생성합니다.
* JobRepository는 Step의 실행 정보를 담고 있는 StepExecution도 저장소에 저장하여 전체 메타데이터를 저장/관리하는 역할을 수행합니다.

### JobLauncher
* JobLauncher는 Job. JobParamerters와 함께 배치를 실행하는 인터페이스입니다.

### ItemReader
* ItemReader는 Step의 대상이 되는 배치 데이터를 읽어오는 인터페이스입니다. File, Xml Db등 여러 타입의 데이터를 읽어올 수 있습니다.

### ItemProcessor
* ItemProcessor는 ItemReader로 읽어 온 배치 데이터를 변환하는 역할을 수행합니다. 이것을 분리하는 이유는 다음과 같습니다.
* 비즈니스 로직의 분리 : ItemWriter는 저장 수행하고, ItemProcessor는 로직 처리만 수행해 역할을 명확하게 분리합니다.
* 읽어온 배치 데이터와 씌여질 데이터의 타입이 다를 경우에 대응할 수 있기 때문입니다.

### ItemWriter
* ItemWriter는 배치 데이터를 저장합니다. 일반적으로 DB나 파일에 저장합니다.
* ItemWriter도 ItemReader와 비슷한 방식을 구현합니다. 제네릭으로 원하는 타입을 받고 write() 메서드는 List를 사용해서 저장한 타입의 리스트를 매게변수로 받습니다.


## 휴면회원 배치 설계
<p align="center">
  <img src="https://github.com/cheese10yun/TIL/raw/master/assets/bach-process.png">
</p>

**가입한 회원 중 1년이 지나도록 상태 변화가 없는 회원을 휴면회원으로 전환하는 배치 처리**

* (1) DB에 저장된 데이터 중 1년간 업데이트되지 않은 사용자를 찾는 로직 ItemReader 구현합니다.
* (2) 대상 사용자 데이터의 상탯값을 휴면으로 전환하는 프로세스를 ItemProcessor에 구현합니다.
* (3) 상태값이 변환된 휴면회원을 실제DB에 저장하는 ItemWriter를 구현합니다.

## 휴면회원 배치 구현

배치처리 순서는 다음과 같습니다.

1. 휴면 회원 Job 설정
2. 휴면 회원 Step 설정
3. 휴면 회원 Reader, Processor, Writer 설정


### Job 설정
```java
@Configuration
public class InactiveUserJobConfig {
    ...
    @Bean
    public Job inactiveUserJob(JobBuilderFactory jobBuilderFactory, Step inactiveJobStep) { //(1)
        return jobBuilderFactory.get("inactiveUserJob")
                .preventRestart() //(2)
                .start(inactiveJobStep) //(3)
                .build();
    }
```

* (1) Job 생성을 직관적이고 편리하게 도와주는 빌더 JobBuilderFactory를 주입받습니다.
* (2) inactiveUserJob 이라는 JobBuilder를 생성하며 `preventRestart()` 설정을 통해 재실행을 막았습니다.
* (3) `start(inactiveJobStep)`은 파라미터에서 주입받은 휴면회원 관련 Step인 inactiveJobStep을 제일 먼저 실행하도록 설정하는 부분입니다.

기본적인 Job설정은 완료 했습니다. Step 설정을 진행하겠습니다.

### Step 설정
```java
...
@Bean
public Step inactiveJobStep(StepBuilderFactory stepBuilderFactory) {
    return stepBuilderFactory.get("inactiveUserStep") //(1)
            .<User, User> chunk(10) //(2)
            .reader(inactiveUserReader()) //(3)
            .processor(inactiveUserProcessor())
            .writer(inactiveUserWriter())
            .build();
}
```
* (1) `stepBuilderFactory.get("inactiveUserStep")`로 inactiveUserStep 이라는 이름의 StepBuilder를 생성합니다.
* (2) 제네릭을 사용해서 `chunk()` 의 입력과 추력 타입을 User로 설정 했습니다. chunk의 인자값은 10으로 설정해서 **쓰기 시에 청크 단위로 writer() 메서드를 실행시킬 단위를 지정했습니다. 즉 커밋단위가 10개입니다.**
* (3) step의 reader, proccsor, writer를 각각 설정했습니다.


### Reader설정
```java
@Bean
@StepScope //(1)
public QueueItemReader<User> inactiveUserReader() {
    //(2)
    List<User> oldUsers =
            userRepository.findByUpdatedDateBeforeAndStatusEquals(
                    LocalDateTime.now().minusYears(1),
                    UserStatus.ACTIVE);

    return new QueueItemReader<>(oldUsers); //(3)
}
```
* (1) 기본 빈 생성은 싱글턴이지만 @StepScope를 사용하면 해당 메서드는 Step의 주기에 따라 새로운 빈을 생성합니다. **즉, 각 Step의 실행마다 새로운 빈을 만들기 때문에 지연 생성이 가능합니다. 주의할 사항은 @StepScode는 기본 프록시 모드가 반환되는 클래스 타임을 참조하기 때문에 @StepScode를 사용하면 반드시 구현된 반환 타입을 명시해 변환해야합니다.** 해당 예제는 QueueItemReader<User>라고 명시했습니다.
* (2) `findByUpdatedDateBeforeAndStatusEquals()` 메서드를 통해서 휴면 회원 리스트를 가져옵니다.
* (3) QueueItemReader 객체를 생성하고 불러온 휴면회원 타깃 대상을 데이터 객체에 넣어 반환합니다.

```java
public class QueueItemReader<T> implements ItemReader<T> {
    private Queue<T> queue;

    public QueueItemReader(List<T> data) {
        this.queue = new LinkedList<>(data); //(1)
    }

    @Override
    public T read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
        return queue.poll(); //(2)
    }
}
```
QueueItemReader는 큐를 사용해서 자장하는 ItemReader 구현체입니다. ItemReader의 기본 반환 타입은 단수형인데 그 에 따라 구현하면 User 객체 1개씩 DB에 select 요청 하므로 매우 비효율적인 방식이 될 수 있습니다.

* (1) QueueItemReader를 사용해서 휴면회원으로 지정될 타깃 데이터를 한번에 불러와 큐에 담아 놓습니다.
* (2) reade() 메서드를 사용할 때 큐의 `poll()`메서드를 통해서 큐에서 데이터를 하나씩 반환합니다.


### Processor 설정
```java
public ItemProcessor<User, User> inactiveUserProcessor() {
    return user -> user.setInactive();
}
```
읽어온 타깃 데이터를 휴면 회원으로 전환시키는 Processor입니다. reader에서 읽은 User를 휴면 상태로 전환화는 Processor 메서드를 추가하는 예입니다.

### Writer 설정
```java
public ItemWriter<User> inactiveUserWriter() {
    return ((List<? extends User> users) -> userRepository.saveAll(users));
}
```
ItemWriter는 리스트 타입을 앞서 설정한 청크 단위로 받습니다. 청크 단위를 10으로 설정했기 때문에 users에게 휴면회원 10개가 주어지며 saveAll()메서드를 통해서 한번에 DB에 저장합니다.

## 배치 심화
* 다양한 ItemReader 구현 클래스
* 다양한 ItemWriter 구현 클래스
* JobParameter 사용하기
* 테스트 시에만 H2 DB를 사용 하도록 설정하기
* 청크 지향 프로세싱
* 배치 인터셉터 Listener 설정하기
* 어노테이션 기반 Listener 설정하기

### 다양한 ItemReader 구현 클래스

기존에는 QueueItemReader 객체를 사용 해서 모든 데이터를 한번에 와서 배치처치를 진행했습니다. **하지만 수백, 수천 개 이상의 데이터를 한번에 가져와서 메모리에 올려놓게되면 좋지 않습니다.** 이때 배치 프로젝트에서 제공하는 PagingItemRedaer 구현체를 사용 사용할 수있습니다. 구현체는 크게 JdbcPagingItemReader, JpaPagingItemRedaer, HibernatePagingItemRdaer가 있습니다. 해당 예쩨에서는 JpaPagingItemRedaer를 사용하겠습니다.

```java
@Bean(destroyMethod="") //(1)
@StepScope
public JpaPagingItemReader<User> inactiveUserJpaReader(@Value("#{jobParameters[nowDate]}") Date nowDate) {
    JpaPagingItemReader<User> jpaPagingItemReader = new JpaPagingItemReader<>();
    jpaPagingItemReader.setQueryString("select u from User as u where u.createdDate < :createdDate and u.status = :status"); //(2)

    Map<String, Object> map = new HashMap<>();
    LocalDateTime now = LocalDateTime.ofInstant(nowDate.toInstant(), ZoneId.systemDefault());
    map.put("createdDate", now.minusYears(1));
    map.put("status", UserStatus.ACTIVE);

    jpaPagingItemReader.setParameterValues(map); //(3)
    jpaPagingItemReader.setEntityManagerFactory(entityManagerFactory); //(4)
    jpaPagingItemReader.setPageSize(CHUNK_SIZE); //(5)
    return jpaPagingItemReader;
}
```

* (1) 스프링에서 destroyMethod를 사용해서 삭제할 빈을 자동으로 추적합니다. destroyMethod=""를 설정하면 warring 메세지를 제거할 수 있습니다.
* (2) JpaPagingItemReader를 사용하면 쿼리를 직접 짜거 실행 하는 방법밖에는 없습니다.
* (3) 쿼리리에서 사용된 updateDate, status 파라미터를 Mpa에 추가해서 사용할 파라미터를 설정합니다
* (4 )트랜잭션을 관리해줄 entityManagerFactory를 설정합니다.
* (5) 한번에 읽어올 크기를 CHUNK_SIZE 만큼 할당합니다.

### 다양한 ItemWriter 구현 클래스

ItemReader와 마찬가지로 상황에맞는 여러 구현 클래스를 제공합니다. JPA를 사용하고 있음으로 JpaItemWriter를 적용합니다.

```java
private JpaItemWriter<User> inactiveUserWriter() {
    JpaItemWriter<User> jpaItemWriter = new JpaItemWriter<>();
    jpaItemWriter.setEntityManagerFactory(entityManagerFactory);
    return jpaItemWriter;
}
```
### JobParameter 사용하기
JobParameter를 사용해서 Step을 실행시킬 때 동적으로 파라미터를 주입시킬 수 있습니다.

### 테스트 시에만 H2 데이터베이스를 사용하도록 설정
```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureTestDatabase(connection = EmbeddedDatabaseConnection.H2)
public class DemoApplication {
    ....
}
```
`@AutoConfigureTestDatabase(connection = EmbeddedDatabaseConnection.H2)` 어노테이션으로 간단하세 설정 가능합니다

### 청크 지향 프로세싱
<p align="center">
  <img src="https://github.com/cheese10yun/TIL/raw/master/assets/chun-process.png">
</p>
청크 지향 프로세싱은 트랜잭션 경계 내에서 청크 단위로 데이터를 읽고 생성하는 프로그래밍 기법입니다. 청크란 아이템이 트랜잭션에 커밋되는 수를 말합니다. read한 데이터 수가 지정한 청크 단위와 칠치하면 write를 수행하고 트랜잭션을 커밋합니다. Step 설정에서 chunk()로 커밋 단위를 지정했던 부분입니다. 즉 기존에도 계속 사용해온 방법이 청크 지향 프로세싱입니다.

청크 지향프러그래밍의 이점은 1000개 개의 데이터에 대해 배치 로직을 실행한다고 가정했을 때 청크로 나누지 않았을 때는 하나만 실패해도 다른 성공한 999개의 데이터가 롤백됩니다. 그런데 청크 단위를 10으로 해서 배치처리를 하면 도중에 배치 처리에 실패하더라도 다른 청크는 영향을 받지 않습니다. 이러한 이유로 스프링 배치에 청크 단위로 프로그래밍을 지향합니다.

### 배치 인터셉터 Listener 설정하기
배치 흐름에서 전후 처리를 하는 Listener를 설정할 수 있습니다. 구체적으로 Job의 전후 처리 Step의 전후 처리 각 청크 단위의 전후 처리 등 세세한 과정 실행시 특정 로직을 할당해 제어할 수있습니다. 가장 대표적인 예로는 로깅 작업이 있습니다.

### 어노테이션 기반 Listener 설정하기
배치 인터셉터 인터페이스를 활용해서 사용하는 방법도 있고 애노테이션을 사용해서 활용하는 방법도 있습니다. 대표적으로 `@BefroeStep, @AsfterStep` 등이 있습니다. 해당 어노테이션으로 시작 전후에 로그를 남기는 설정도 가능합니다.

### JobParameter 사용하기
JapParameter를 사용해 Step을 실행시킬 때 동적으로 파라미터를 주입시클 수 있습니다.

### Step의 흐름을 제어하는 Flow

Step의 가장 기본적은 흐름은 `읽기-처리-쓰기` 입니다. 여기서 세부적인 조건에 따라서 Step의 실행 여부를 정할 수 있습니다. 이런 흐름을 제어하는 `Flow` 제공 합니다.

<p align="center">
  <img src="https://github.com/cheese10yun/TIL/raw/master/assets/batch-flow.png">
</p>

흐름에 조건에 해당하는 부분을 `JobExecutionDecider` 인터페이스를 사용해 구현 할 수 있습니다. `JobExecutionDecider` 인터페이스는 `decide()` 메서드 하나만 제공합니다.

```java
public interface JobExecutionDecider {
    FlowExecutionStatus decide(JobExecution jobExecution, @Nullable StepExecution stepExecution);
}

public Class xxxJobExecutionDecider implements  JobExecutionDecider {

    @Override
    public FlowExecutionStatus decide(JobExecution jobExecution, @Nullable StepExecution stepExecution){
        if(특정 조건...){ // (1)
            return FlowExecutionsStatus.COMPLETED; // (2)
        }
        return FlowExecutionsStatus.FAILED; // (3)

    }
}
```
* (1) 특정 조건에 대한 로직
* (2) 조건에 만족하고 JobStep을 실행 시킬 경우 `COMPLETED` 리턴
* (3) 조건에 만족하지 않고 JobStep을 **실행 하지않을 경우**  `FAILED` 리턴

`Flow` 조건으로 사용될 경우 InactiveJobExceutionDecider 클래스를 구현 했습니다. 이를 사용할 Flow를 구현 해야합니다. `Step` 메서드가아닌 `Flow`를 주압 받고 주입받은 `Flow`를 빈으로 등록해야합니다.

```java
@Bean
public Flow xxxJobFlow(Step xxxJobStep){
    FlowBuilder<Flow> flowBuilder = new FlowBuilder<>("xxxJobFlow"); // (1)

    return flowBuilder
        .start(new xxxJobExcetuinDeicder()) // (2)
        .on(FlowExecutionStatus.FAILED.getName()).end() // (3)
        .on(FlowExecutionStatus.COMPLETED.getName()).to(xxxJobStep).end(); // (4)
}
```
* (1) `FlowBuilder`를 시용해서 Flow 객체를 생성합니다.
* (2) 위에서 작성한 `xxxJobExecutionDecider` 클래스를 `start()` 으로 설정해 맨 처음 시작하도록 합니다.
* (3) `xxxJobExecutionDecider` 클래스의 decide() 메서드를 통해 리턴 값이 `FAILED` 일 경우 `end()` 메서드를 사용해서 끝나도록 설정합니다.
* (4) `xxxJobExecutionDecider` 클래스의 decide() 메서드를 통해 리턴 값이 `COMPLETED` 일 경우 기존에 설정한 `xxxJobStep`을 실행하도록 설정합니다.




## 재시도
네트워크 접속이 끊어지거나 장비가 다운되는 등 실패 시나리오는 다양합니다. 시스템은 언젠가 복구 될테니 다시 한번 시도는 해볼 가치는 있습니다.

### 스템 구성하기
```java
@Bean
public Step step1() {
    return steps.get("user xxxxx")
    .<User, User>chunk(10)
        .faulTolerant()
            .retryLimit(3).retry(XXXXXException.class)
    .render(something())
    .writer(something())
    .build();
}
```
자바 구성으로 재시도를 활성화 할 경우, 첫 번째 스텝은 오류를 허용하도록 만들어야 재시도 제한 횟수 및 재시도 대상 예외를 지정할 수 있습니다. 먼저 `faulTolerant()`로 오류 허용 스탭을 얻은후, `retryLimit()` 메서드로 재시도 제한 횟수를, `retry()` 메서드로 재시도 대상 예외를 발생합니다.

### 재시도 템플릿
* 스프링 배치가 제공하는 재시도 및 복구 서비스를 코드에 활용하는 다른 방법도 있습니다. 재시도 로직을 구현된 커스텀 ItemWriter<T>를 작성하거나 아예 전체 서비스 인터페이스에 재시도 기능을 입힐 수 있습니다.
* 스프링 배치 RetryTemplate은 바로 이런 용도로 만들어진 클래스입니다. 비니지스 로직과 재시도 로직을 분리해서 마치 재시도 없이 한 번만 시도하는 것처럼 코드를 작성할 수 있개 해줍니다.
* 재시도 -> 실패 -> 복구 반복적인 과정을 간명한 하나의 API 메서드로 호출로 감싼 `RetryTemplate`는 여러 가지 유스 케이스를 지원합니다.

```java
public class RetryableUserRegistrationServiceItemWriter implements ItemWriter<UserRegistration> {
    private static final Logger logger = LoggerFactory.getLogger(RetryableUserRegistrationServiceItemWriter.class);
    private final UserRegistrationService userRegistrationService;
    private final RetryTemplate retryTemplate;

    public RetryableUserRegistrationServiceItemWriter(UserRegistrationService userRegistrationService, RetryTemplate retryTemplate) {
        this.userRegistrationService = userRegistrationService;
        this.retryTemplate = retryTemplate;
    }

    /**
     * takes aggregated input from the reader and 'writes' them using a custom implementation.
     */
    public void write(List<?extends UserRegistration> items)
        throws Exception {
        for (final UserRegistration userRegistration : items) {
            UserRegistration registeredUserRegistration = retryTemplate.execute(
                    (RetryCallback<UserRegistration, Exception>) context -> userRegistrationService.registerUser(userRegistration));

            logger.debug("Registered: {}", registeredUserRegistration);
        }
    }
}

  ....
  @Bean
    public RetryTemplate retryTemplate() {
        RetryTemplate retryTemplate = new RetryTemplate();
        retryTemplate.setBackOffPolicy(backOffPolicy());
        return retryTemplate;
    }

    @Bean
    public ExponentialBackOffPolicy backOffPolicy() {
        ExponentialBackOffPolicy backOffPolicy = new ExponentialBackOffPolicy();
        backOffPolicy.setInitialInterval(1000);
        backOffPolicy.setMaxInterval(10000);
        backOffPolicy.setMultiplier(2);
        return backOffPolicy;
    }
```
* 재시도 시간 간격을 정하는 BackOffplicy는 RetryTemplate의 유용한 기능입니다. 실제로 실패 직후 재시도하는 시간 간격을 점점 늘려 여러 클라이언트가 같은 호출 할때 스텝이 잠기지 않도록 예방하는 수단으로 활용할 수있습니다.

### AOP 기반 재시도
스프링 배치가 제공하는 AOP 어드바이저를 이용해서 RetryTempate 처럼 사용할 수 있습니다. 프록시 전체에 재시 로직 어드바이스를 추가하면 RetryTempate이 빠진 본래 코드로 그대로 사용가능합니다.

```java
@Retryable(backoff = @Backoff(delay = 1000, maxDely = 10000, multiplier = 2))
public User batchSomething(){....}
```
**구성 클래스에 반드시 @EnableRety 를추가 해야합니다.**

## Spring Batch Table

<p align="center">
  <img src="https://github.com/cheese10yun/TIL/raw/master/assets/meta-data-erd.png">
</p>

### BATCH_JOB_INSTANCE

```sql
CREATE TABLE `BATCH_JOB_INSTANCE` (
  `JOB_INSTANCE_ID` bigint(20) NOT NULL,
  `VERSION` bigint(20) DEFAULT NULL,
  `JOB_NAME` varchar(100) NOT NULL,
  `JOB_KEY` varchar(32) NOT NULL,
  PRIMARY KEY (`JOB_INSTANCE_ID`),
  UNIQUE KEY `JOB_INST_UN` (`JOB_NAME`,`JOB_KEY`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

JOB_INSTANCE_ID | VERSION | JOB_NAME | JOB_KEY
----------------|---------|----------|--------
1 | 0 | inactiveUserJob | df9e59b818ab301226e71dcf67795b07
2 | 0 | inactiveUserJob | 34c2f2838f31f237450a6c7659e36995
3 | 0 | orderDailySumJob | d41d8cd98f00b204e9800998ecf8427e
4 | 0 | orderDailySumJob | d6832decf796311c39d3d934a9d7cfd5
5 | 0 | orderDailySumJob | 212470e06656926b4b339a42dc5d64c3

JOB_INSTANCE_ID는 BATCH_JOB_INSTANCE 테이블의 PK, JOB_NAME 수행한 Batch Job Name

**BATCH_JOB_INSTANCE 테이블은 Job Parameter에 따라 생성됩니다.** Job Parameter는 Spring Batch가 실행될때 외부에서 받을 수 있는 파라미터 입니다. **같은 Batch Job 이라도 Job Parameter가 다르면 다른 BATCH_JOB_INSTANCE 에 기록됩니다.** BATCH_JOB_EXECUTION_PARAMS 기반으로 JOB_KEY를 만들며 해당 값은 유니크 제약조건이 있기 때문에 같은 BATCH_JOB_EXECUTION_PARAMS을 넘기는 경우 생성되지 않습니다.

### BATCH_JOB_EXECUTION
```sql
CREATE TABLE `BATCH_STEP_EXECUTION` (
  `STEP_EXECUTION_ID` bigint(20) NOT NULL,
  `VERSION` bigint(20) NOT NULL,
  `STEP_NAME` varchar(100) NOT NULL,
  `JOB_EXECUTION_ID` bigint(20) NOT NULL,
  `START_TIME` datetime NOT NULL,
  `END_TIME` datetime DEFAULT NULL,
  `STATUS` varchar(10) DEFAULT NULL,
  `COMMIT_COUNT` bigint(20) DEFAULT NULL,
  `READ_COUNT` bigint(20) DEFAULT NULL,
  `FILTER_COUNT` bigint(20) DEFAULT NULL,
  `WRITE_COUNT` bigint(20) DEFAULT NULL,
  `READ_SKIP_COUNT` bigint(20) DEFAULT NULL,
  `WRITE_SKIP_COUNT` bigint(20) DEFAULT NULL,
  `PROCESS_SKIP_COUNT` bigint(20) DEFAULT NULL,
  `ROLLBACK_COUNT` bigint(20) DEFAULT NULL,
  `EXIT_CODE` varchar(2500) DEFAULT NULL,
  `EXIT_MESSAGE` varchar(2500) DEFAULT NULL,
  `LAST_UPDATED` datetime DEFAULT NULL,
  PRIMARY KEY (`STEP_EXECUTION_ID`),
  KEY `JOB_EXEC_STEP_FK` (`JOB_EXECUTION_ID`),
  CONSTRAINT `JOB_EXEC_STEP_FK` FOREIGN KEY (`JOB_EXECUTION_ID`) REFERENCES `BATCH_JOB_EXECUTION` (`JOB_EXECUTION_ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

JOB_EXECUTION_ID | VERSION | JOB_INSTANCE_ID | CREATE_TIME | START_TIME | END_TIME | STATUS | EXIT_CODE | EXIT_MESSAGE | LAST_UPDATED | JOB_CONFIGURATION_LOCATION
-----------------|---------|-----------------|-------------|------------|----------|--------|-----------|--------------|--------------|---------------------------
1 | 2 | 1 | 2019-07-02 06:51:22 | 2019-07-02 06:51:22 | 2019-07-02 06:51:23 | COMPLETED | COMPLETED |  | 2019-07-02 06:51:23 | NULL
2 | 2 | 2 | 2019-07-02 07:14:00 | 2019-07-02 07:14:00 | 2019-07-02 07:14:01 | COMPLETED | COMPLETED |  | 2019-07-02 07:14:01 | NULL
3 | 2 | 3 | 2020-01-13 11:00:50 | 2020-01-13 11:00:50 | 2020-01-13 11:00:50 | COMPLETED | COMPLETED |  | 2020-01-13 11:00:50 | NULL
4 | 2 | 4 | 2020-01-13 11:59:15 | 2020-01-13 11:59:15 | 2020-01-13 11:59:15 | COMPLETED | COMPLETED |  | 2020-01-13 11:59:15 | NULL
5 | 2 | 5 | 2020-01-13 12:48:53 | 2020-01-13 12:48:53 | 2020-01-13 12:48:54 | COMPLETED | COMPLETED |  | 2020-01-13 12:48:54 | NULL

* `JOB_EXECUTION_ID` 칼럼은 `BATCH_JOB_INSTANCE` 테이블의 PK를 참조 하고 있습니다.
* `BATCH_STEP_EXECUTION` 와 `BATCH_JOB_INSTANCE`는 부모 자식관계입니다.
* BATCH_STEP_EXECUTION는 자신의 부모 BATCH_JOB_INSTANCE 성공/실패 내역을 모두 갖고 있습니다.
* jobParameters : Job 실행에 필요한 매개변수 데이터입니다.
* jobInstance : Job 실행 단위가 되는 객체입니다.
* stepExecutuons : StepExecutuon을 여러개 가질 수 있는 Collection 타입입니다.
* status : Job 실행 상태를 나타내는 필드(Enum)입니다. 상태값으로는 `COMPLETED, STARTING, STARTED, STOPPING, STOPPED, FAILED, ABANDONED, UNKNOWN` 이 있습니다.
* startTime : Job이 실행된 시간입니다. null이면 시작되지 않았다는 의미 입니다.
* createTime : JobExecution이 생성된 시간입니다.
* endTime: JobExecution이 끝난 시간입니다.

<p align="center">
  <img src="https://github.com/cheese10yun/TIL/raw/master/assets/job-job-instance-job-execution.png">
</p>

* `Job`: 특정 잡, 2달이상 로그인안한 유저 휴면 회원 처리 등
* `Job Instance`: Job Parameter를 실행한 Job(Job Parameter 단위로 생성)
* `Job Execution`: Job Parameter로 실행한 Job의 실행, 1번 째 시도 혹은 그 다음 등

### BATCH_JOB_EXECUTION_PARAMS
JOB_EXECUTION_ID | TYPE_CD | KEY_NAME | STRING_VAL | DATE_VAL | LONG_VAL | DOUBLE_VAL | IDENTIFYING
-----------------|---------|----------|------------|----------|----------|------------|------------
5 | STRING | requestDate | 2019-10-13 | 1970-01-01 00:00:00 | 0 | 0 | N
5 | LONG | run.id |  | 1970-01-01 00:00:00 | 2 | 0 | Y
5 | STRING | version | 12 | 1970-01-01 00:00:00 | 0 | 0 | Y
5 | STRING | -job.name | orderDailySumJob | 1970-01-01 00:00:00 | 0 | 0 | N

**BATCH_JOB_EXECUTION에 대한 Parameter정보들이 저장되는 곳이다** BATCH_JOB_EXECUTION, BATCH_JOB_EXECUTION_PARAMS 1:N 관계이며 위 테이블은 ID 5번에 들어가는 parameter 정보들이 저장된다

## 참고
* [처음으로 배우는 스프링 부트 2](https://kyobobook.co.kr/product/detailViewKor.laf?mallGb=KOR&ejkGb=KOR&barcode=9791162241264&orderClick=JAj)를 정리한 글입니다.
* [기억보단 기록을 - spring-batch-in-action](https://github.com/jojoldu/spring-batch-in-action/blob/master/3_%EB%A9%94%ED%83%80%ED%85%8C%EC%9D%B4%EB%B8%94%EC%97%BF%EB%B3%B4%EA%B8%B0.md)
