# API request/response 로깅

OncePerReuqestFilter 상속한 필터를 빈으로 등록

이 필터는 각 요청에 대해 한 번만 실행되므로, 요청별로 고유한 처리를 할 때 유용

MDC에 request_id 넣어서 기록하면 찾기 쉽다 

ContentCachingRequestWrapper를 이용하면 요청과 응답 본문을 캐싱하여 로깅후에도 유지 가능

로깅하고 마지막에 copyBodyToResponse()를 호출 

#spring