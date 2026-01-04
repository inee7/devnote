---
tags: [spring, retrofit2, kotlin, http-client, rest-client, external-api]
---

# retrofit2를 springboot+kotlin 에서 사용하기

```kotlin
implementation("com.squareup.retrofit2:retrofit:${retrofitVersion}")
implementation("com.squareup.retrofit2:converter-jackson:${retrofitVersion}")
implementation("com.squareup.okhttp3:okhttp:${okHttpVersion}")
```

[https://hun-developer.tistory.com/8](https://hun-developer.tistory.com/8)

[https://github.kakaocorp.com/apollo/apollo-admin/blob/208a376f50467f3049d1206a025567136406ed5a/src/main/kotlin/com/kakaopay/apollo/admin/adaptor/AdaptorConfig.kt](https://github.kakaocorp.com/apollo/apollo-admin/blob/208a376f50467f3049d1206a025567136406ed5a/src/main/kotlin/com/kakaopay/apollo/admin/adaptor/AdaptorConfig.kt)

헤더 넣는법 

```kotlin
OkHttpClient.Builder httpClient = new OkHttpClient.Builder();

httpClient.addInterceptor(new Interceptor() {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request().newBuilder().addHeader("parameter", "value").build();
        return chain.proceed(request);
    }
});
Retrofit retrofit = new Retrofit.Builder().addConverterFactory(GsonConverterFactory.create()).baseUrl(url).client(httpClient.build()).build();
```

response.code() Integer 값이 200 미만이거나 300이상 혹은 204나 205일 경우에는 response.body()는 null이 되고 response.errorBody() 를 byteArray로 받아서 파싱해주자

