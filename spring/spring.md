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
使用：
@Autowired
RestHighLevelClient restHighLevelClient;

@Autowired
@Qualifier("restHighLevelClient")
RestHighLevelClient client;
```

