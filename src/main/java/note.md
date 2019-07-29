# SpringBoot 自动配置原理
- 官方文档 配置文件能配置的属性
    https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#appendix
- 自动配置原理
    - SpringBoot启动 加载主配置类 @SpringBootApplication --> 开启自动配置功能 @EnableAutoConfiguration
    - @EnableAutoConfiguration 作用：
        利用EnableAutoConfigurationImportSelector给容器导入一些组件
            selectImports()方法
            ```
                List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
                //获取候选配置
                List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
                //扫描所有jar包类路径下 META-INF/spring.factories
                //把扫描到的文件内容包装成properties对象
                //从properties中获取EnableAutoConfiguration.class类（类名）对应的值 然后添加在容器中
            ```
            将类路径下 META-INF/spring.factories里面配置的所有EnableAutoConfiguration的值加入容器中
    - 每一个自动配置类进行自动配置功能
    - 以HttpEncodingAutoConfiguration为例
    ```java
        @Configuration    //表示这是一个配置类 可以给容器中添加组件
        @EnableConfigurationProperties({HttpProperties.class}) 
         //启用ConfigurationProperties功能 将配置文件中对应的值和HttpEncodingProperties绑定起来
        @ConditionalOnWebApplication(
            type = Type.SERVLET
        )
          //判断当前应用是否是web应用 Spring底层@Conditional注解 根据不同条件 满足指定条件 配置类中的配置才生效
        @ConditionalOnClass({CharacterEncodingFilter.class})
          //判断当前项目是否有CharacterEncodingFilter类（SpringMVC中进行乱码解决的过滤器）
        @ConditionalOnProperty(
            prefix = "spring.http.encoding",
            value = {"enabled"},
            matchIfMissing = true
        )
      
          //判断配置文件中是否存在spring.http.encoding.enabled的配置 如果不存在也认为生效
        //满足以上条件 配置类生效 一旦生效 配置类给容器中添加各种组件
    ```
    - 所有在配置文件中能配置的属性都是在xxxProperties类中封装
    ```java
      @ConfigurationProperties(
          prefix = "spring.http"
      ) //从配置文件中获取指定的值和bean的属性进行绑定
      public class HttpProperties {
          private boolean logRequestDetails;
          private final HttpProperties.Encoding encoding = new HttpProperties.Encoding();

    
    ```
    
    ```java
    public class HttpEncodingAutoConfiguration {
          //他已经和SpringBoot的配置文件映射了
        private final Encoding properties;
          //只有一个有参构造器的情况下 参数的值从容器中拿
        public HttpEncodingAutoConfiguration(HttpProperties properties) {
            this.properties = properties.getEncoding();
        }

      @Bean //给容器中添加一个组件 组件中的某些值从properties中获取
      @ConditionalOnMissingBean
      public CharacterEncodingFilter characterEncodingFilter() {
          CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
          filter.setEncoding(this.properties.getCharset().name());
          filter.setForceRequestEncoding(this.properties.shouldForce(org.springframework.boot.autoconfigure.http.HttpProperties.Encoding.Type.REQUEST));
          filter.setForceResponseEncoding(this.properties.shouldForce(org.springframework.boot.autoconfigure.http.HttpProperties.Encoding.Type.RESPONSE));
          return filter;
      }
    }

    ```
    - 我们能配置的属性来源于这个功能的properties类
    
- SpringBoot配置
    - SpringBoot启动会加载大量自动配置类
    - 看需要的功能有没有默认的自动配置类
    - 看自动配置类配置了哪些组件 要用的有 不需要配置 没有 自己配置
    - 给容器中自动配置类添加组件 会从properties类中获取某些属性 可以在配置文件中指定
 ```
    xxxAutoConfiguration:自动配置类 给容器中添加组件
    xxxProperties:封装配置文件中相关属性

```
    
# Conditional派生注解  
- @Conditional 指定的条件成立 才给容器中添加组件 配置里面的内容才生效
- 派生
    - @ConditionalOnJava 系统java版本是否符合要求
    - @ConditionalOnBean 容器中是否存在指定bean
    - @ConditionalOnMissingBean 是否不存在bean
    - @ConditionalOnExpression 满足SpEL表达式绑定
    - @ConditionalOnClass 系统中有指定的类
    - @ConditionalOnMissingClass 没有指定的类
    - @ConditionalOnSingleCandidate 容器中只有一个指定的bean 或者这个是首选bean
    - @ConditionalOnProperty 系统中指定的属性是否有指定的值
    - @ConditionalOnResource 类路径下是否存在指定资源文件
- 怎么确定哪些自动配置类生效
    - 开启debug模式 打印自动配置报告
      debug=true
    ============================
    CONDITIONS EVALUATION REPORT
    ============================
    
    
    Positive matches:（启用的自动配置类）
    -----------------
    
       CodecsAutoConfiguration matched:
          - @ConditionalOnClass found required class 'org.springframework.http.codec.CodecConfigurer' (OnClassCondition)
    
       CodecsAutoConfiguration.JacksonCodecConfiguration matched:
          - @ConditionalOnClass found required class 'com.fasterxml.jackson.databind.ObjectMapper' (OnClassCondition)
    
    Negative matches:（没有启用的）
    -----------------
    
       ActiveMQAutoConfiguration:
          Did not match:
             - @ConditionalOnClass did not find required class 'javax.jms.ConnectionFactory' (OnClassCondition)
    
       AopAutoConfiguration:
          Did not match:
             - @ConditionalOnClass did not find required class 'org.aspectj.lang.annotation.Aspect' (OnClassCondition)
    

