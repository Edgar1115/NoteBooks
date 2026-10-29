评分过程大致经过：

```Plain
InterviewEvaluationService
    ↓
evaluateAnswer()
    ↓
evaluateAnswerByScorerAgent()
    ↓
构造 AI 参数
    ↓
InterviewAiInvoker.callAiSyncWithParameters()
```

---

# evaluateAnswer()

业务起始点

> evalutateAnswer()要求传入参数：
> 
> 1. sessionId，当前面试会话的ID；
>     
> 2. requestId，当前请求的标识，用于业务日志等上下文；
>     
> 3. questionNumber，当前面试题号；
>     
> 4. questionContent，当前题目的文本；
>     
> 5. answerContent，用户提交的回答；
>     
> 6. scorerAgent，评分Agent的配置信息。
>     

## evaluateAnswerByScorerAgent()

> 优先评分工作流

> 开始准备AI工作流请求

> 先拿简历上下文

传入参数：sessionId,requestId,questionNumber,questionContent,answerContent,scorerAgent.

第一步：

```Java
String resumeContextText =
    buildResumeContextText(
        interviewQuestionCacheService
            .getSessionResumeContext(sessionId)
    );
```

> 根据当前SessionId，取得这个面试会话相关的简历上下文。

评分AI要看：

```Plain
题目 answerContent
答案 questionContent
简历背景 resumeContextText
```

将三类信息组成AI Workflow参数

```Java
Map<String, Object> parameters =
    buildScorerWorkflowParameters(
        answerContent,
        questionContent,
        resumeContextText
    );

// -------------------------------------------------
// buildScorerWorkflowParameters实现
private Map<String, Object> buildScorerWorkflowParameters(
    String answerContent,
    String questionContent,
    String resumeContextText
) {

    Map<String, Object> parameters =
        new LinkedHashMap<>();
    parameters.put(
        KEY_AGENT_USER_INPUT,
        answerContent
    );

    parameters.put(
        KEY_QUESTION,
        questionContent
    );

    parameters.put(
        KEY_RESUME_CONTEXT,
        resumeContextText
    );

    return parameters;
}
```

这里得到的`parameters`大概如下：

```Java
{
    AGENT_USER_INPUT = 用户回答,
    question = 当前题目,
    resume_context = 简历上下文
}
```

> 这里只是在准备参数，没有调用AI

## 进入`InterviewAiInvoker`

```Java
String workflowResponse =
    interviewAiInvoker.callAiSyncWithParameters(
        
        // 传入带'_score'后缀的模型调用会话标识
        // 判断两个请求应不应该归为同一Flight
        sessionId + "_score",
        
        // 评分Agent的配置信息
        scorerAgent,
        
        // buildScoreWorkflowParameters构造的结构化参数
        // 用户回答 + 当前题目 + 简历上下文
        parameters,

        // 用来标记AI请求属于哪一个业务阶段
        InterviewAiGuardStage.INTERVIEW_EVALUATION,
        
        // 生成Single-flight Key
        // 格式： stage | sessionId | questionNumber | answerHash
        // Single-flight Key里，不包含requestId
        // 所以不同的请求，即不同的requestId，如果Single-flight Key的四个参数相同，那么他们的Key就相同
        interviewAiInvoker.buildSingleFlightKey(
            InterviewAiGuardStage.INTERVIEW_EVALUATION,
            sessionId,
            questionNumber,
            answerContent
        )
    );
```

### buildSingleFlightKey()

1. 构造stage
    

```Java
// 如果stage为null,使用默认字符"interview-default"
// stage可以隔离不同的业务类型
// stage正常应为interveiw-evaluation
String safeStage =
    StrUtil.blankToDefault(
        stage,
        "interview-default"
    );
```

2. 对sessionId做归一化
    

```Java
// trimToEmpty：去掉字符串首尾空白
// 归一化，下一步同，让不同表现形式但含义相同的输入，尽量生成一致的key
// 举例： "   abc-123   " -> "abc-123"
String safeQuestionNumber = StrUtil.blankToDefault(
    StrUtil.trimToEmpty(questionNumber),
    "-"
);
```

3. 对`questionNumber`做同样的处理
    

```Java
String safeQuestionNumber =
    StrUtil.blankToDefault(
        StrUtil.trimToEmpty(questionNumber),
        "-"
    );
```

4. `answerContent`不直接放Key
    

```Java
String safeAnswerHash =
    // 先判断是否为空
    StrUtil.isBlank(answerContent)
        ? "-"
        : DigestUtil.sha256Hex(    // 将答案转换为SHA-256，取前16位
            answerContent.trim()
        ).substring(
            0,
            16
        );
        
// 避免将很长的答案原文直接塞进Key
// 相同答案经过同样归一化后，可以得到稳定的Key
```

举例：

请求A

```Plain
sessionId = S1
questionNumber = 3
answer = "Java 是……"
```

请求B

```Plain
sessionId = S1
questionNumber = 3
answer = "Java 是……"
```

两个请求answerHash相同，所以他们的Single-flight Key相同

5. 为什么Single-flight Key不加入`requestId`
    

Key更关注“业务上是不是同一个评分任务”，而不是“是不是同一个HTTP请求”

### callAiSyncWithParameters()

第一次出现“延迟执行”

```Java
return guardedCall(
    stage,
    singleFlightKey,
    
    // 这里等价于Callable<String> callable = () -> doChat(...);
    // 这个时候还没有执行
    () -> doChat(
        input,
        sessionId,
        agentProperties,
        null,
        parameters
    )
);
```

> **Callable与Supplier的区别**
> 
> Supplier
> 
> ```Java
> @FunctionalInterface
> public interface Supplier<T> {
>     T get();
> }
> ```
> 
> Callable
> 
> ```Java
> @FunctionalInterface
> public interface Callable<V> {
>     V call() throws Exception;
> }
> ```
> 
> Callable适合包装“可能抛Exception的操作”，Supplier不能声明抛出受检异常。

> `doChat()`在做什么？
> 
> ```Java
> private String doChat(
>     String input,
>     String sessionId,
>     AgentPropertiesDO agentProperties,
>     String fileUrl,
>     Map<String, Object> parameters
> ) throws Exception {
> ```
> 
> 调用
> 
> ```Java
> xingChenAIClient.chat(...)
> ```
> 
> 最终：
> 
> ```Plain
> doChat()
>     ↓
> XingChenAIClient.chat()
>     ↓
> 第三方星辰 AI
> ```
> 
> 最终会得到完整AI响应字符串。

### guardedCall()

guardedCall()构造，可见传入的是Callable

```Java
private String guardedCall(
    String stage,
    String singleFlightKey,
    Callable<String> callable
) throws Exception
```

第二次出现“延迟执行”

```Java
return distributedInterviewAiSingleFlightService.execute(
    safeStage,
    key,
    
    // 这里的callable就是刚刚传入的Lambda
    () -> aiCallGuardService.execute(
        safeStage,
        key,
        callable
    )
);
```

**两层Lambda的实际执行顺序**

```Plain
Distributed Single-flight

拿到的是 Supplier
│
└── supplier.get()
       ↓
    AiCallGuardService.execute()
       │
       └── 最终 callable.call()
              ↓
           doChat()
```