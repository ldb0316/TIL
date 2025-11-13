# Spring batch 를 이용한 스케줄러 구현

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
package myapp.scheduler.domain.test;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.LinkedList;
import java.util.List;
import java.util.Queue;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.NonTransientResourceException;
import org.springframework.batch.item.ParseException;
import org.springframework.batch.item.UnexpectedInputException;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.stereotype.Component;
import myapp.scheduler.global.util.CustomDateUtils;
import myapp.scheduler.global.util.LinkEntityUtils;
import myapp.scheduler.global.util.TSIDGenerator;
import myapp.scheduler.global.util.batch.io.Decompressor;
import lombok.extern.slf4j.Slf4j;

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

### 2. Processor

### 3. Writer
