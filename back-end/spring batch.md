# Spring batch 를 이용한 스케줄러 구현 (Job 구현 위주. Quartz는 따로 X)

## 1. Spring batch Job (Quartz) 흐름도

```text
Job
  |
  v
StepBuilder
  |
  v
JobFlowBuilder
  |
  v
Quartz Scheduler Regist -----> JobLauncher.run()
                                      |
                                      v
                                    Step
          .-----------------------------------------------------------------------------.
          |                                                                             |
          Reader -> StepListener -> Processor -> Writer -> Listener -> nextStep(optional)

```

## 2. Spring batch에서 item을 스텝별로 컨트롤 할 수 있도록 도와주는 3요소

### 1. Reader

reader는 특정 resource(DB 데이터, 로컬 파일 등)을 읽어 item(DTO클래스)으로 매핑시켜 return 해주는 역할을 한다.

#### 로컬 특정 경로의 데이터 파일을 읽어 item으로 return 하는 Custom ItemReader 구현 예제

연계 데이터 송수신 부하를 최소화하기 위해 중복되는 데이터는 파일명에 구분자로 붙어 전달되는 경우이다.<br>
때문에 파일명에 있는 특정 값을 추출하는 로직이 포함된다.

```java
/**
 * 파일명에 붙은 문자열을 item에 넣어 처리하기 위한 custom ItemReader 구현
 *
 * @author LDB
 *
 */

@Slf4j
@Component
public class TestInfoItemReader implements ItemReader<TestInfo> {

  @Value("${file.down-path}") //프로퍼티 값은 컴포넌트로 등록해야 읽을 수 있다.
  private String downPath;

  private Queue<BufferedReader> readerQueue = new LinkedList<>();
  private Queue<String> fileNameQueue = new LinkedList<>();

  private BufferedReader currentReader;
  private String currentValue1;
  private String currentValue2;
  private String currentGroupKey;

  private boolean isInit = false;

  private void setFileQueue() throws IOException {
    if (!isInit) {
      String fileName = "TestInfo";
      String currentDate = CustomDateUtils.getCurrentDate();
      // 오늘 날짜에 해당하는 경로에 있는 TestInfo.tar.gz 을 읽는다. 와일드카드는 중복 전달 가능성을 고려하여 숫자값이 붙어 넘어오는 파일을 처리할 수 있도록 하기 위함이다.
      final String resourcesPath = downPath + "/" + currentDate + "/" + fileName + "*.tar.gz";
      // 압축을 해제할 경로를 지정한다.
      final String decompressOutputDir = downPath + "/" + currentDate;
      Resource[] resources = new PathMatchingResourcePatternResolver().getResources("file:" + resourcesPath);

      List<File> targetFiles = new ArrayList<>();
      Arrays.stream(resources).forEach(resource -> {
        try {
          // 경로에 압축을 해제한다. 압축해제로직은 조금만 검색해봐도 나오니 여기선 설명 패스.
          List<File> decompressesFiles = Decompressor.decompressTarGzToFiles(resource, decompressOutputDir);
          targetFiles.addAll(decompressesFiles);
        } catch (IOException e) {
          e.printStackTrace();
          throw new RuntimeException("리소스 압축 해제를 실패했습니다", e);
        }
      });

      // 압축 해제한 파일마다 BufferedReader를 만들어 큐에 넣는다.
      for (int i = 0; i < targetFiles.size(); i++) {
        this.readerQueue.offer(new BufferedReader(new FileReader(targetFiles.get(i))));
        this.fileNameQueue.offer(targetFiles.get(i).getName()); // 압축 해제된 파일은 Value1_Value2_파일명.txt 와 같은 형식이다.
      }

      isInit = true;
    }
  }

  private void setWork() {
    if (currentReader == null && readerQueue.size() != 0) {
      this.currentReader = readerQueue.poll();

      String fileName = fileNameQueue.poll();
      this.currentValue1 = fileName.split("_")[0];
      this.currentValue2 = fileName.split("_")[1];
      this.currentGroupKey = TSIDGenerator.generateTsid();

      log.debug("[SetWork] fileName : {} , readerQueueSize: {}", fileName, readerQueue.size());
    }
  }

  private void resetWork() throws IOException {
    if (this.currentReader != null) {
      this.currentReader.close();
      this.currentReader = null;
    }
    this.currentValue1 = null;
    this.currentValue2 = null;
    this.currentGroupKey = null;
  }

  private String readLine() throws IOException {
    if (currentReader != null) {
      return currentReader.readLine();
    } else {
      return null;
    }
  }


  // item reader 메인로직
  @Override
  public TestInfo read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {

    setFileQueue(); // file queue를 만든다 (최초진입)
    setWork(); // 현재 대상 queue를 뽑고, Value1, Value2 등 세팅한다.

    String line = readLine(); // setCurrentWork에서 만들어진 currentReader에서 line을 읽는다.
    if (line == null) { // line이 없으면
      resetWork(); // reset시키고
      setWork(); // currentWork를 다시 세팅한다(next queue poll) - 이 시점에서 Value1, Value2, group key 가 다시 세팅된다.
      line = readLine(); // 다시 만들어진 currentReader에서 line을 읽는다.
    }

    if (line != null) { // line이 null이 아니라면 쭉쭉
      String[] dataArr = line.split("\\|"); // 한 라인당 3개의 값이 | 구분자로 전달됨
      String pk = TSIDGenerator.generateTsid();
      return new TestInfo(
          pk,
          currentGroupKey,
          currentValue2,
          dataArr[0],
          dataArr[1],
          dataArr[2],
          Double.parseDouble(dataArr[3]),
          currentValue1);
    } else { // 위에서 다시 세팅했는데도 line이 null이면 queue가 더이상 없는 것이므로 종료
      return null;
    }
  }
}

```

Custom ItemReader는 org.springframework.batch.item.ItemReader 인터페이스 구현을 통해 만들어야 한다.<br>
실행 구조는 read()가 null을 return 할때까지 반복해서 실행한다.<br>
때문에 조건을 잘 주어 올바른 시점에 null을 return 하도록 유도해야 한다.

<hr>

### 2. Processor

processor는 reader에서 리소스를 읽어 만들어준 item을 받아 추가적인 처리 작업을 수행할 수 있다. <br>
item의 특정 값을 기준으로 다른 값을 변경한다던가 하는 처리가 가능하다. <br>
다음은 item에 좌표정보(경도,위도)가 들어있을 때, 올바르지 않은 좌표정보를 가진 item을 filtering하는 processor 구현 예제이다.

```java
/**
 * custom item processor 구현
 * 유효하지 않은 좌표값을 필터링하기 위해 구현함
 * null 리턴 시 해당 item은 넘기지 않음 판정 -> 필터링됨
 *
 * @author LDB
 *
 */
@Component
public class TestItemProcessor implements ItemProcessor<TestInfo, TestInfo> {

  @Override
  public TestInfo process(TestInfo item) throws Exception {
    if (item.getXcrd().equals("0.0") || item.getYcrd().equals("0.0"))
      return null; //null을 return할 경우 해당 item은 필터링된다.
    return item;
  }

}

```

이 외에도 item에 특정 값을 세팅하거나, 다양한 작업이 가능하다. <br>
DDD방식으로 Item에 다양한 업무 로직 함수를 정의하여 프로세서를 통해 자동처리도 구현 가능할듯 하다.

<hr>

### 3. Writer

writer는 reader, processor step을 거쳐온 item들을 write하는 단계이다.<br>
아래는 DB에 item을 넣는 예제이다.

```java
  @RequiredArgsConstructor
  class TestItemWriter<TestInfo> implements ItemWriter<TestInfo> {

    private final JdbcTemplate jdbcTemplate;

    @Override
    public void write(List<TestInfo> items) throws Exception {

      // reader 에서 읽어온 데이터를 insert 한다.
      JdbcQuery query = PGQueryBuilder.insertLink(items);
      jdbcTemplate.update(query.sql(), query.params().toArray());
    }
  }
```

### 4. StepBuilder

read, process, write Step을 구성했으면 이제 해당 스텝들을 하나로 모아 TaskletStep을 만들어주어야 한다.<br>
아래는 StepBuilder를 이용하는 예제이다.

```java

  //testItemReader, testItemProcessor, testItemWriter 는 Spring bean으로 자동주입받은것을활용
  ItemReader<TestInfo> reader = (ItemReader<TestInfo>) testItemReader;
  ItemProcessor<TestInfo, TestInfo> processor = (ItemProcessor<TestInfo, TestInfo>) testItemProcessor;
  ItemWriter<TestInfo> writer = (ItemWriter<TestInfo>) testItemWriter; // DB에 읽어온 데이터를 저장하는 로직(실행은 step에서)

  // work step 을 만든다.
  SimpleStepBuilder<TestInfo, TestInfo> stepBuilder = stepBuilderFactory.get("StepName").<TestInfo, TestInfo>chunk(chunkSize).reader(reader).processor(processor).writer(writer);

```

SimpleStepBuilder에서 지정하는 chunk는 item을 처리할 때 안정성을 유지하기 위해 지정하는 처리 단위이다.<br>
StepName은 Spring의 StepBuilderFactory bean에서 StepBuilder를 가져올 때 해당 bulider의 name을 지정한다.

## 3. Listener

### 1. 리스너 등록

listener는 해당 Job이 완료되는 시점이나 Step이 완료된 시점에서 별도 Job을 실행할 수 있게 해준다.

```java

JobFlowBuilder jobBuilder = jobBuilderFactory.get(jobName) // jobName에 해당하는 job 생성
        .listener(new TestInfoBatchJobCompletionListener(jobLauncher, pgUpdateQueryBatchJob)) // job이 실행되고 끝났을 때 호출되는
                                                                                                   // 리스너 등록. job 완료 후
                                                                                                   // TestInfoBatchJobCompletionListener
                                                                                                   // 후처리 수행됨
        .flow(stepBuilder.listener(new CommonStepCompletionListener()).build());
```

jobBuilder의 listener에 해당 Job이 완료되는 시점에 실행할 Listener를 등록한다.<br>
.flow 부분을 보면 알 수 있듯이 stepBuilder에도 listener를 등록할 수 있고, step이 완료될 때 마다 해당 Listener가 실행된다.

### 2. 리스너 구현

CommonJobCompletionListener를 상속하여 구현한다.
아래는 afterJob 함수에서 jobExecution을 인자로 받아 성공적으로 Job이 완료되었는지 확인 후 다시 다음 job을 호출하는 예제이다.

```java
@RequiredArgsConstructor
public class TestInfoBatchJobCompletionListener extends CommonJobCompletionListener {

  private final JobLauncher jobLauncher;

  private final PGUpdateQueryBatchJob pgUpdateQueryBatchJob;

  @Override
  public void afterJob(JobExecution jobExecution) {
    super.afterJob(jobExecution);
    if (jobExecution.getStatus() == BatchStatus.COMPLETED) {

      String formattedJobId =
          TestInfo.class.getSimpleName() + "_" + "runAfterFileToDbBatchJob" + "_" + CustomDateUtils.getLogFormatCurrentDateTime();
      JobParameters params = new JobParametersBuilder().addString("jobId", formattedJobId).toJobParameters();
      try {
        jobLauncher.run(pgUpdateQueryBatchJob.create(TestInfo.class), params);
      } catch (JobExecutionAlreadyRunningException e) {
        throw new RuntimeException("postGIS 쿼리 업데이트 중 오류 발생 JobExecutionAlreadyRunningException", e);
      } catch (JobRestartException e) {
        throw new RuntimeException("postGIS 쿼리 업데이트 중 오류 발생 JobRestartException", e);
      } catch (JobInstanceAlreadyCompleteException e) {
        throw new RuntimeException("postGIS 쿼리 업데이트 중 오류 발생 JobInstanceAlreadyCompleteException", e);
      } catch (JobParametersInvalidException e) {
        throw new RuntimeException("postGIS 쿼리 업데이트 중 오류 발생 JobParametersInvalidException", e);
      }
    }
  }

}
```

afterJob 뿐 만 아니라 beforeJob 함수도 Override하여 구현하면 해당 작업이 시작되기 전에 특정 로직을 실행시킬 수 있다.

## 4. 추가 step 구현

작업이 완전히 종료되고나서 추가적인 step이 필요할 수 있다.<br>
연계 파일을 읽어서 전부 DB에 저장했다면, 해당 파일을 삭제하거나 backup 디렉토리로 옮기는 작업 등이 그렇다.

```java

    /** FileToDb 작업 후 해당 파일 삭제 (txt파일 삭제) */
    String cleanupStepName = this.getClass().getSimpleName() + "CleanupStep[TestInfo]";
    TaskletStepBuilder cleanupStepBuilder = stepBuilderFactory.get(cleanupStepName).tasklet(new CleanupTask());
    jobBuilder.next(cleanupStepBuilder.build());


```

```java

  @RequiredArgsConstructor
  class CleanupTask<TestInfo> implements Tasklet {

    private final Class<T> type;

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
      String fileName = "TestInfo";
      try {
        Resource[] resources = getResources(fileName, type); // 대충 압축 파일 풀고 했던 그 로직 재활용용
        for (Resource resource : resources) {
          if (resource.exists()) {
            File file = resource.getFile();
            if (!file.delete()) {
              throw new RuntimeException("파일 삭제에 실패했습니다. file :" + file.getName());
            }
          }
        }
      } catch (Exception e) {
        // 파일이 없더라도 멈추지 않도록
        e.printStackTrace();
      }

      return RepeatStatus.FINISHED;
    }
  }

```

## 5. Job 마무리

jobBuilder의 .build를 호출하여 Job을 생성하고 해당 Job을 Quartz Scheduler에 등록하면 된다.

```java
return jobBuilder.end().build();
```
