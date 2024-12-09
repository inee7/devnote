CompletionPolicy 청크의 완료 여부를 결정할 수 있다 스프링배치에서 제공해주는 구현체들이 있다 SimpleCompletionPolicy 는 정해둔 임곗값에 도달하면 청크 완료 TimeoutTerminationPolicy 는 시간이 넘을 때 완료로 처리 여러 조합을 하고 싶으면 CompositeCompletionPolicy를 써라
``` java
new CompositeCompletionPolicy().setPolicies( 
    new CompletionPolicy[] { policy1, policy2}
    );
```
policy는 stepBuilder에 chunk에 등록하면 된다

### 스텝에서 policy 까지

스텝에서 chunk인자로 policy가 들어오면 RepeatTemplate에 policy를 저장한다 하나씩 데이터를 처리하려고 시작할때 policy를 start처리한다 처리가 완료되면 policy 를 update하고 isComplete로 끝내는지 확인한다