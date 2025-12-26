# CORS
브라우저 초기에 보안상의 이유로 스크립트 내에서 시작된 교착 출처 HTTP 요청을 제한하는데, 이를 SOP(Same-Origin Policy, 동일 출처 정책)라 한다.
SOP는 두 Origin 간에 프로토콜, 포트, 호스트가 같아야 동일 Origin라고 할 수 있다.

그래서 이를 보완하기 위해 브라우저측에서 JSONP를 사용하거나, 서버측에서 **CORS**를 이용하여 해결할 수 있다. 여기서 **CORS(Cross-Origin Resource Sharing)** 란, 웹 서버 도메인간 액세스 제어 기능을 제공하여 보안 도메인간 데이터 전송을 가능하게 해준다.

#security
