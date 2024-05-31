之所以写这篇文档是因为在iac有两个人使用spring-validation，直接造成接口无法访问，好在只是干掉了一个还没投入使用的接口。



#### 简介

在使用spring-validation会用到两个注解@Valid和@Validated,建议默认使用@Validated。



@Validated 注解可用于类、方法参数上
用于类上方法可不必添加@Validated注解，但requestBody的参数仍然需要添加

@Valid 单独仅用于requestBody的参数上,或者在类上有声明@Validated时，@Valid可声明在方法上
这样也可以使requestBody的参数进行校验

类上声明@Validated后方法再声明@Validated，requestBody的参数仍然不会被校验



#### 示例代码

```java
@RestController
@Validated
*public class* MyController {

​    @RequestMapping("/test")
​    *public* Object test1(@RequestBody  @Validated TestParam param) {
​        *return* param;
​    }

​    @RequestMapping("/test2")
​    *public* Object test2( @NotNull(message = "名称不能为空")  String name) {
​        *return* name + ",hehe";
​    }

​    @RequestMapping("/test3")
​    *public* Object test3(@NotBlank(message = "名称不能为空") String name,@NotNull(message = "年龄不能为空") Integer age) {
​        *return* name + ",hehe," + age;
​    }
}

*public class* TestParam {

​    @NotBlank
​    *private* String name;

​    @NotNull
​    *private* Integer age;

​    @Pattern(regexp = "^1[1-9]{10}$", message = "请输入正确的手机号")
​    @NotEmpty
​    *private* String phone;

​	//省略getter、setter
}

@RestControllerAdvice
*public class* ExceptionAdvice {

​    @ExceptionHandler
​    @ResponseStatus(HttpStatus.BAD_REQUEST)
​    *public* Object test(MethodArgumentNotValidException e) {
​        HashMap<String, Object> map = *new* HashMap<>();
​        FieldError fieldError = e.getBindingResult().getFieldErrors().get(0);
​        map.put("status", "failed");
​        map.put("info", fieldError.getDefaultMessage());
​        *return* map;
​    }

​    @ExceptionHandler
​    @ResponseStatus(HttpStatus.BAD_REQUEST)
​    *public* Object test1(ConstraintViolationException e) {
​        StringJoiner sj = *new* StringJoiner(",");
​        e.getConstraintViolations().stream().map(ConstraintViolation::getMessage).findFirst().ifPresent(sj::add);
​        HashMap<String, Object> map = *new* HashMap<>();
​        map.put("status", "failed");
​        map.put("info", sj.toString());
​        *return* map;
​    }

}
```



TestParam 中使用的@NotBlank @NotNull注解都是javax.validation包下的，validation-api是定义了一个规范，而hibernate-validator是这个规范的实现，而hibernate-validator包下也有同样功能的注解，一般具体实现框架自己定义的注解会有一些功能的扩展，以及更强大的功能，但不建议直接使用实现框架的注解，最好是使用validation-api中的注解；

如果需要在core包或者api包里的Param实体加注解，那么需要在项目里引入validation-api，且一定要是optional的。



#### 错误示范

下面回顾下使用spring-validation不规范造成的接口无法访问：



![img](https://cdn.nlark.com/yuque/0/2021/png/2387107/1624527841562-ee7853d4-7717-41f4-8dd4-ec2d4a6f2212.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/2387107/1624527898786-9b68f241-c718-4cc1-8dce-cb6529f2ec66.png)

首先是iac中新增了一个pda的登录接口，然后加了@Validated注解，经过测试后没管了，已经上线。



然后有新增的param实体定义在iac-core中，用了hibernate-validator的注解

![img](https://cdn.nlark.com/yuque/0/2021/png/2387107/1624528055953-4bac659f-fe53-4a2d-9153-21a20960d420.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/2387107/1624528083989-36f13f98-9f9c-4436-b87c-3d8e2a1f73dd.png)

并且引入的包没有加optional，而spring-boot-starter-web引入的hibernate-validator的版本是6.0.18，hibernate-validator会引入validation-api，5.4的hibernate引入的是1.1.0的validation-api，是针对1.1.0规范的一个实现，而6.0引入的是2.0的api，是针对2.0规范的实现

![img](https://cdn.nlark.com/yuque/0/2021/png/2387107/1624528331159-26554d70-adb8-4748-b592-eaf3b828ff33.png)

这样直接导致spring中validation规范变成的1.0版本的实现，而其他地方使用validation-api注解有2.0版本的注解就会导致没有对应的实现

![img](https://cdn.nlark.com/yuque/0/2021/png/2387107/1624528439085-da790b61-0743-436c-b612-0b20f7967f09.png)

此时调用接口就会有如下提示：

![img](https://cdn.nlark.com/yuque/0/2021/png/2387107/1624528528162-7a185179-160f-40db-a45a-8d3b19ea07ee.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/2387107/1624528542953-501019f3-4b53-4934-bede-ae2728c78e88.png)







所以，最重要的一点：使用validation，最好看看该项目中其他地方都使用的什么样的注解，不要混用，最好都使用规范性注解，如同slf4j