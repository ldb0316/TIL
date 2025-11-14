## Rest api 구현 시 에러 처리


1. ResponseEntityExceptionHandler를 상속받아 CustomExceptionHandler 를 구현한다.
2. ResponseEntityExceptionHandler에 미리 구현되어있는 다양한 핸들러 메서드를 Override하여 커스텀 구현한다.

3. 이 외에도 추가적인 Exception 핸들링을 하려면 @ExceptionHandler([Exception명칭.class]) 어노테이션을 붙인 메서드를 정의하고, 로직을 처리한다. 


### 예시

``` java 
/**
 * 예외처리를 위한 핸들러 클래스
 */
@Slf4j
@ControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
@ExceptionHandler(Exception.class)
  protected ResponseEntity<Object> handleAllException(Exception ex, WebRequest request) {
    // springframework 에서 기본으로 정의된 Exception 은 response 규격에 맞게 변환 필요
    // 외에는 BusinessException 상속 받았고, 내부에 ErrorStatus 설정한 대로 오류 처리됨
    if (ex instanceof AccessDeniedException) {
      return handleStatusException(ErrorStatus.FORBIDDEN, request, ex);
    }
    if (ex instanceof AuthenticationException || ex instanceof BadCredentialsException || ex instanceof LockedException) {
      return handleStatusException(ErrorStatus.UNAUTHORIZED, request, ex);
    } else if (ex instanceof BusinessException exception) {
      return handleBusinessException(exception, request);
    }
	// ErrorResponse는 표준화된 에러 메세지 전달을 위해 구현한 클래스이다.
    ErrorResponse errorResponse = ErrorResponse.errorStatus()
        .status(ErrorStatus.INTERNAL_SERVER_ERROR)
        .errors(FieldError.of(null, "N/A", request.getDescription(false)))
        .build();
    log.error(ex.getMessage());
    return new ResponseEntity<>(errorResponse, ErrorStatus.INTERNAL_SERVER_ERROR);
  }
  
  private final ResponseEntity<Object> handleStatusException(ErrorStatus status, WebRequest request, Exception ex) {
	// FieldError는 어떤 field에서 에러가 발생했는지 메세지를 전달하기위해 구현한 클래스이다
    ErrorResponse errorResponse =
        ErrorResponse.errorStatus().status(status).errors(FieldError.of(null, request.getDescription(false), ex.getMessage())).build();
    return new ResponseEntity<>(errorResponse, status);
  }
  
}
```

### override 하여 커스텀 로직 구현1 - request body 에 대한 validate 실패 시 

``` java 
  /* @Validated 또는 @Valid 사용하여 @RequestBody 오류 발생 시 */
  @Override
  protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
	  
	  ...
	  
	}
```

### override 하여 커스텀 로직 구현2 - ModelAttribute 에 대한 validate 실패 시 

``` java 
/* @Validated 또는 @Valid 사용하여 @ModelAttribute 오류 발생 시 */
  @Override
  protected ResponseEntity<Object> handleBindException(BindException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
  
	...
  
  }
```