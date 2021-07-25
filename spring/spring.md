# spring Bean注入容器配置

1、xml方式

```
<Beans>
    <Bean id="restHighLevelClient", class="asa.asa.asa.asa.Xxx">
    </Bean>
</Beans>
```

2、注解方式

```
@Configuration
public class ESConfig {
    // 注解方式配置bean,方法名就是xml方式中id="XXX"
    // 使用：1、@Autowired方式，2、@Autowired + @Qualifier("xxx")方式
    @Bean
    public RestHighLevelClient restHighLevelClient() {
        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(
                        new HttpHost("localhost", 9200, "http"),
                        new HttpHost("localhost", 9201, "http")));// 集群方式
        return client;
    }

}

```

3、使用

```
使用，// 如同下面方式1变量注入
@Autowired
RestHighLevelClient restHighLevelClient;
或：
@Autowired
@Qualifier("restHighLevelClient")
RestHighLevelClient client;
```

```
    /**
     * 1、变量（filed）注入
     */
//    @Autowired
//    private UserLoginMapper userLoginMapper;

    /**
     * 2、构造函数注入
     */
//    private final UserLoginMapper userLoginMapper;
//
//    @Autowired
//    public UserLoginServiceImpl(UserLoginMapper userLoginMapper) {
//        this.userLoginMapper = userLoginMapper;
//    }

    /**
     * 3、set方式注入
     */
    private UserLoginMapper userLoginMapper;

    @Autowired
    public void setUserLoginMapper(UserLoginMapper userLoginMapper) {
        this.userLoginMapper = userLoginMapper;
    }
```

