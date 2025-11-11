maven repository에서 다음 라이브러리를 다운로드 후 로컬 maven repository에 넣고 pom.xml에 다음을 추가한다.

commons-compress 1.27.1
commons-io 2.20.0

주의: commons-io 라이브러리가 없어도 컴파일 에러는 발생하지 않으나, ClassNotFoundException이 발생한다. (BoundedInputStream 못찾음)


``` xml
<dependency>
	<groupId>org.apache.commons</groupId>
	<artifactId>commons-compress</artifactId>
	<version>1.27.1</version>
</dependency>
<dependency>
	<groupId>org.apache.commons</groupId>
	<artifactId>commons-io</artifactId>
	<version>2.20.0</version>
</dependency>
```

이제 정상적으로 commons-compress 라이브러리를 사용할 수 있다.

로직에서 활용하기 위해 util 함수를 만들어준다.

``` java
	public static List<Resource> decompressTarGz(Resource resource, String outputDir) throws IOException {
		try (InputStream fis = resource.getInputStream();
			GzipCompressorInputStream gis = new GzipCompressorInputStream(fis);
			TarArchiveInputStream tis = new TarArchiveInputStream(gis)) {

		  TarArchiveEntry entry;
		  List<Resource> extractedResources = new ArrayList<>();

		  while ((entry = tis.getNextEntry()) != null) {
			if (entry.isDirectory())
			  continue;

			File outputFile = new File(outputDir, entry.getName());
			try (OutputStream os = new FileOutputStream(outputFile)) {
			  tis.transferTo(os);
			}

			extractedResources.add(new FileSystemResource(outputFile));
		  }
		  return extractedResources;
		}
	}
```


file resource를 읽는 로직에서 resource를 함수에 전달하고 압축 해제된 resource를 받아서 처리한다


``` java
	private <T extends LinkEntity> Resource[] getResources(String fileName, Class<T> type) {
		try {
		  String currentDate = CustomDateUtils.getCurrentDate();
		  final String resourcesPath = path + "/" + currentDate + "/" + fileName + "*.tar.gz";

		  Resource[] resources = new PathMatchingResourcePatternResolver().getResources("file:" + resourcesPath);

		  // 필터링: fileName.tar.gz 또는 fileName_숫자.tar.gz만 포함
		  List<Resource> filteredAndDecompressesResources = new ArrayList<>();
		  Arrays.stream(resources).forEach(resource -> {
			String filename = resource.getFilename();
			if (filename != null && filename.matches(fileName + "(\\.tar.gz|_\\d+\\.tar.gz)")) {
			  try {
				List<Resource> decompressesResources = Decompressor.decompressTarGz(resource, resourcesPath);
				filteredAndDecompressesResources.addAll(decompressesResources);
			  } catch (IOException e) {
				throw new RuntimeException("리소스 압축 해제를 실패했습니다", e);
			  }
			}
		  });

		  if (filteredAndDecompressesResources.isEmpty()) {
			throw new RuntimeException("리소스가 존재하지 않습니다.");
		  }

		  return filteredAndDecompressesResources.toArray(new Resource[0]); // List를 배열로 변환하여 반환
		} catch (IOException e) {
		  throw new RuntimeException("리소스를 로드하는 데 실패했습니다", e);
		}
	}
```