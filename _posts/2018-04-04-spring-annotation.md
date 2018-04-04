---
layout: post
title: 深入了解spring注解
date: 2018-04-04
categories: spring
tags: [spring-boot]
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

}

```

<font color="#dd0000">6. @RequestBody</font>
- 请求参数为json类型
- 只能作用在post请求
- 用来处理Content-Type为application/json,application/xml编码的内容

```java
@RestController
public class CityRestController {

    @Autowired
    private CityService cityService;

    @RequestMapping(value = "/city/addCityInfo", method = RequestMethod.POST)
    public Integer addCityInfo(@RequestBody City city) {
        return cityService.addCityInfo(city);
    }
}

```





