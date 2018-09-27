---
layout: post
title: spring-annotation
date: 2018-04-04
categories: spring
tags: [spring]
description: 深入了解spring注解。
---

<font color="#dd0000">1. @RestController与@Controller之间的区别？</font>
- @RestController注解的Controller的方法无法返回对应的页面，配置的视图解析器不起作用，返回的内容就是return中的内容，其实就是@Controller与@ResponseBody结合
- @Controller注解的Controller视图解析器会生效，会将响应的结果返回到相应的页面;若需要异步返回，只需在方法前加上注解@ResponseBody
```java
//@RestController直接return中的内容，是个异步的方法
@RequestMapping("/test")
@RestController   
public class HelloWorldController {

    @RequestMapping("/index")
    public String sayHello() {
        return "Hello,World!";
    }
}
```
```java
@Controller
public class CityController {

    @Autowired
    private CityService cityService;

    //返回结果映射至视图页面city.ftl中
    @RequestMapping(value = "/api/city/{id}", method = RequestMethod.GET)
    public String findOneCity(Model model, @PathVariable("id") Long id) {
        model.addAttribute("city", cityService.findCityById(id));
        return "city";
    }

    //加上@ResponseBody 不再返回视图页面了，直接return "cityAList",异步方法
    @RequestMapping(value = "/api/city", method = RequestMethod.GET)
    @ResponseBody 
    public String findAllCity(Model model) {
        return "cityList";
    }
}
```

<font color="#dd0000">2. @SpringBootApplication</font>
- 自动扫描当前类同级及子包下的响应注解注册为spring beans,在类中main方法中SpringApplication.run()启动应用

<font color="#dd0000">3. @RequestMapping</font>
- 用来处理请求地址映射，可作用于类上和方法上；作用于类上时，表示类中的所有响应请求的方法都是以该地址作为父路径
- 常用属性有value(uri地址)、method(提交方式)

<font color="#dd0000">4. @PathVariable</font>
- 用来处理映射 URL 绑定的占位符，可以将URL中绑定的变量通过该注解绑定到操作方法的入参中
```java
    @RestController
    public class CityRestController {
        
        @RequestMapping("/testPathVariable/{id}")
        public String testPathVariable(@PathVariable("id") Integer id)
        {
            System.out.println("testPathVariable:"+id);
            return SUCCESS;
        }
    
    }
```

<font color="#dd0000">5. @RequestParam</font>
- spring boot获取请求的参数：1.request.getParamter("paramName");2.用@RequestParam获取
- 注解底层还是实现了request.getParamter()，常用于处理简单的数据类型
- 用来处理Content-Type为application/x-www-form-urlencoded编码的内容，提交方式为get，post

```java
@RestController
public class CityRestController {

    @Autowired
    private CityService cityService;

    @RequestMapping(value = "/city/findOneCity", method = RequestMethod.GET)
    public City findOneCity(@RequestParam(value = "cityname", required = true) String cityName) {
        //@RequestParam注解作用：表名表单中传递的参数，去掉该注解的话表单中传递的参数得与接受参数名称一致
        return cityService.findCityByName(cityName);
    }
    
    @RequestMapping(value = "/city/addCityInfo", method = RequestMethod.POST)
    public Integer addCityInfo(@RequestBody City city) {
        return cityService.addCityInfo(city);
    }

}
```

<font color="#dd0000">6. @RequestBody</font>
- 请求参数为json类型
- 只能作用在post请求
- 用来处理Content-Type为application/json,application/xml编码的内容

<font color="#dd0000">7. @Autowired</font>
- 自动帮你把bean里面引用(对象)的setter/getter方法省略，自动帮你set/get

<font color="#dd0000">8. @Value</font>
- 简化properties文件中配置的读取
```html
    @Value("${cluster.datasource.url}")
    private String url;

    @Value("${cluster.datasource.username}")
    private String user;

    @Value("${cluster.datasource.password}")
    private String password;

    @Value("${cluster.datasource.driverClassName}")
    private String driverClass;
```

<font color="#dd0000">9. @Configuration和@Bean</font>
- @Configuration相当于application-config.xml中的beans标签，用于标注类
- @Bean相当于application-configs.xml中的bean标签，用于标注方法

<font color="#dd0000">10. @Primary</font>
- 常用于返回Bean的方法上，当该Bean有多个同类Bean时，优先选择该Bean
```html
    @Bean(name = "masterDataSource")
    @Primary    //优先选择该bean而不选择下面的
    public DataSource masterDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(driverClass);
        dataSource.setUrl(url);SqlSessionFactory
        dataSource.setUsername(user);
        dataSource.setPassword(password);
        return dataSource;
    }
    
    @Bean(name = "clusterDataSource")
    public DataSource clusterDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(driverClass);
        dataSource.setUrl(url);
        dataSource.setUsername(user);
        dataSource.setPassword(password);
        return dataSource;
    }
```

<font color="#dd0000">11. @Qualifier</font>
当一个接口有多个实现类时，自动注入bean时区分具体Bean<br/>
```java
//接口
public interface EmployeeService {
    public EmployeeDto getEmployeeById(Long id);
}

//实现类1
@Service("service")
public class EmployeeServiceImpl implements EmployeeService {
    public EmployeeDto getEmployeeById(Long id) {
        return new EmployeeDto();
    }
}

//实现类2
@Service("service1")
public class EmployeeServiceImpl1 implements EmployeeService {
    public EmployeeDto getEmployeeById(Long id) {
        return new EmployeeDto();
    }
}

@Controller
@RequestMapping("/emplayee.do")
public class EmployeeInfoControl {
    
    @Autowired
    @Qualifier("service")   //标注实现类为EmployeeServiceImpl
    EmployeeService employeeService;
     
    @RequestMapping(params = "method=showEmplayeeInfo")
    public void showEmplayeeInfo(HttpServletRequest request, HttpServletResponse response, EmployeeDto dto) {
        //...
    }
}
```




