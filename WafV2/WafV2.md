# WAF V2

### 예상 도입 배경

- 기존 시스템
    - WAF 방화벽으로 외부에서의 웹 접근을 차단함.
    - 웹 ACL을 aws console에서 수동으로 처리
- ip 권한 관리 자동화 시스템 도입 필요.

### 목표

- [list/get api를 성공적으로 호출시켜보자](https://www.notion.so/WAF-V2-d858e553cf0a4f9f9755d9d02fdb215e)
- [update api를 성공적으로 호출시켜보자](https://www.notion.so/WAF-V2-d858e553cf0a4f9f9755d9d02fdb215e)
- [호출이 성공하면, 해당 api를 구조화시켜보자.](https://www.notion.so/WAF-V2-d858e553cf0a4f9f9755d9d02fdb215e)

### 시행착오

**왜 WAF V2인가?**

AWS WAF는 기존의 WAF Classic과 2019년 11월에 나온 WAF V2 2가지 종류가 있다. 둘은 별개의 독립적인 서비스이며, 따라서 기존에 적용되어 있던 classic의 rule을 WAF v2에서 호출하는 등의 상호연관 작업이 불가능하다. 또한 classic에서 v2로의 migration을 제공하는 자동화 도구도 없다.

 v2에서 바뀐 점은 다음과 같다.

- full range CIDR
    - classic : /8 ~ /32
    - v2 : /1 ~ /32
- rule에서 OR 연산 가능해짐

**AWS 개발자 문서 확인**

- 호환성 확인
    - 2019년 11월에 나온 waf v2의 java sdk 버전이 [1.11.682](https://mvnrepository.com/artifact/com.amazonaws/aws-java-sdk-wafv2/1.11.682) 이므로, 1.11.682 이상이면 됨
- http API 확인
    - [https://docs.aws.amazon.com/ko_kr/waf/latest/APIReference/API_Operations_AWS_WAFV2.html](https://docs.aws.amazon.com/ko_kr/waf/latest/APIReference/API_Operations_AWS_WAFV2.html)
    - GetIPSet, ListIPSets, UpdateIPSets 3개 사용
- java sdk API 확인
    - GetIPSet, ListIPSets, UpdateIPSets 3개 메서드 사용

**WAF get api를 성공적으로 호출시켜보자.**

- aws 공식 문서를 확인해보니 Region에 global이 없다 ...?
    - global(cloudfront) 는 clientBuilder의 region을 us-east-1로 설정해야 한다.
- api 호출 시에 request parameter중에 scope도 `CLOUDFRONT | REGIONAL` 을 제대로 명시해야 한다.

```java
//AWS Config
@Bean
@Qualifier("wafv2Client")
public AWSWAFV2Client wafv2Client(){
    return  (AWSWAFV2Client) AWSWAFV2ClientBuilder
            .standard()
            .withCredentials(new DefaultAWSCredentialsProviderChain())
            .withRegion(Regions.US_EAST_1) // cloudfront region : us-east-1
            .build();
}

//getIPSet, listIPSets API 호출
String id = "cf_ip_set_id";
String name = "cf_ip_set_name";
String scope = "CLOUDFRONT";
GetIPSetResult result = awsWafV2Service.getIPSet(id, name, scope);
ListIPSetsResult result = awsWafV2Service.listIPSets(0,"",scope);

```

**WAF update api를 성공적으로 호출시켜보자.**

- update 메서드의 경우, 기존 ip + 새 ip 이므로 이 부분만 2개의 파라미터로 분리하여 처리
    
    ```java
    public Boolean updateIPSet(
    List<String> existAddresses, List<String> newAddresses,
     String id, String lockToken, String name, String scope);
    ```
    

- update 메서드는 최소한 한번의 get api 호출이 필요함.(lockToken을 필요로 함)
    
    Request Syntax (**UpdateIPSet**)
    
    ```json
    {
       "Addresses": [ "string" ],
       "Description": "string",
       "Id": "string",
       "LockToken": "string",
       "Name": "string",
       "Scope": "string"
    }
    ```
    
    - lockToken은 optimistic locking에 쓰인다고함(it ensures that no changes have been made to the entity since you last retrieved it).
    

**호출이 성공하면, 해당 api를 구조화시켜보자.**

- JUnitException: TestEngine with ID 'junit-jupiter' failed to discover tests
    - 메서드 명이 바뀌고, 기존 메서드명을 호출하여 발생하는 오류
- 테스트와 api 호출 메서드 정의
    - 테스트는 결과값이 null 이 아닌지 정도만 assert
        - exception은 처리 안함.
    - api 호출 메서드
        - 메서드의 파라미터를 기존 api 명세와 똑같이 하려고 함.
        - get, list 메서드 api와 동일하게 처리.