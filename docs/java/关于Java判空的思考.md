# 关于Java判空的思考
###### 前言
> 当我们在提供一个服务或者说接口的时候，不管是Http服务还是RPC服务，总归有自己的实现逻辑。而接口实现逻辑的第一步往往是字段校验。

> 当然，复杂的时候为了安全我们接口间调用还要进行加解密或者自定义的业务校验，比如内部通信的时候会有类似sysId、bizId、bizKey这种服务提供方分配的校验字段。外部通信也就是专线通信或者公网通信的时候会有针对传输报文的的加解密校验。之后才是我们的业务逻辑处理。当然也有可能加解密业务校验之后再进行业务数据字段的校验。这个根据具体情况灵活安排。

> 总之，字段校验在我们的业务开发中是一个比较常见的逻辑。既然常见我们就可以把相对通用的部分拆解出来行程组件，简化我们的业务开发。

#### 目标
以终为始，我们先简单针对我们的背景对我们要进行的操作提出几点目标
- 灵活配置判断规则，当然大多数情况下是判空。这里我先埋下一个引子，其实我们可以给每个字段灵活配置自己的判断规则，就像Java中实现Comparable接口来自定义比较规则一样。扩展的话还可以支持正则等。
- 优雅调用。作为组件来说要让使用者专心去开发业务逻辑不用操心配置的事情。具象一点来说，我们要做到一行代码搞定判断。或者说一行代码都不用，只要“描述”到位，交给切面去处理！
- 灵活配置规则不符合，拿为空举例。要让使用者能灵活配置规则校验不通过的情况下，是丢弃还是处理，处理的话怎么处理返回空还是返回别的结果，还是抛异常？

#### 方法
要想不让业务开发人员写代码关心字段判断这种事情，又要可扩展，灵活配置。那么我们就要在某个地方对判断规则、处理方法、处理结果进行描述。具体到Java里可以是配置文件，也可以是Annotation.总的思路就是对字段应该满足什么条件进行描述。然后我们的组件读取描述条件然后代替业务开发人员去执行判断逻辑。就像前端H5好多富文本editor好多表格组件如easyUI中对字段的描述一样。用结构化描述代替if...else逻辑分支的执行。将规则抽象，将if...else通用

#### 现状1
###### 大量 if...else判空
在每个接口入口处，直接对需要判断的字段进行业务判空

```
//示例代码
  public void bizDeal(RequestVo requestVo) {
        //字段校验
        if(StringUtils.isBlank(requestVo.getData())){
	    // Do something...  返回什么或者抛异常等等
        }
        if(!"BIZ888".equals(requestVo.getBizData())){
        	// Do something...  返回什么或者抛异常等等
        }
        //...此处省略好多个字段校验
        if(StringUtils.isBlank(requestVo.getSignData())){
        	// Do something...  返回什么或者抛异常等等
        }
        // 处理业务逻辑
    }

```
##### 优点
直观，易于理解，判断灵活，想咋判断咋判断。

##### 缺点:
高度定制、无法通用，比如要有100个业务处理方法需要判空，每个方法里都要这么来一次可以说是很难受了


#### 现状2

在需要校验的地方调用返校验方法

```
  public void bizDeal(Map<String, String> reqMap) {
        //0.字段校验
        validXXXReq(reqMap);
        //1.防重
        //2.字段转换
        //3.处理逻辑..这里去处理业务逻辑
    }
   
```
抽取到类的私有方法内做if...else判空校验

```
private void validXXXReq(Map<String, String> reqMap) {
        String apiName = "XXX接口";
        ValidateUtil.assertTrue(StringUtils.isNotBlank(reqMap.get("version")), DirectRetEnum.COMMON_PARAM_ERROR, new Object[]{"XXX接口", "version", "版本号为空"});
        ValidateUtil.assertTrue(StringUtils.isNotBlank(reqMap.get("certId")), DirectRetEnum.COMMON_PARAM_ERROR, new Object[]{apiName, "certId", "ID为空"});
        ValidateUtil.assertTrue(StringUtils.isNotBlank(reqMap.get("appId")), DirectRetEnum.COMMON_PARAM_ERROR, new Object[]{apiName, "appId", "App代码为空"});
        ValidateUtil.assertTrue(StringUtils.isNotBlank(reqMap.get("userId")), DirectRetEnum.COMMON_PARAM_ERROR, new Object[]{apiName, "userId", "用户ID为空"});
        ValidateUtil.assertTrue(StringUtils.isNotBlank(reqMap.get"keyXXX"));
        ValidateUtil.assertTrue(StringUtils.isNotBlank(reqMap.get"keyXXX"));
        ValidateUtil.assertTrue(StringUtils.isNotBlank(reqMap.get"keyXXX"));
		//这里可能还有好多个字段
        ValidateUtil.assertTrue(StringUtils.isNotBlank(reqMap.get"keyXXX"));
        ValidateUtil.assertTrue(StringUtils.isNotBlank(reqMap.get"keyXXX"));
        ValidateUtil.assertTrue(StringUtils.isNotBlank(reqMap.get"keyXXX"));
        LOGGER.info("XXX接口，参数校验通过，orderId={},userserId={}", reqMap.get("orderId"), reqMap.get("userId"));
    }
```
校验方法*ValidateUtil#assertTrue*

```
/**
 * @throws 校验不通过抛异常
 */
public static void assertTrue(boolean assertion, RetEnum retEnum, Object... args) {
        if (!assertion) {
            throw new ParamCheckException(retEnum, args);
        }
    }
```

返回码枚举

```
public enum RetEnum {
    /**
     * 系统内部异常
     */
    SYS_ERROR("F99999", "未知错误异常"),
    /**
     *成功
     */
    SUCCESS("00000", "成功")
    //省略...
}
```
##### 优点：同一个类中或者业务相似的方法可以达到一定程度的复用
##### 缺点: 虽然看起来在业务参数校验处好看一些了，但是还是摆脱不了大量if...else

#### 现状3
1. 设置该对象的终的某些字段需要判空
2. 调用反射对这些对象进行判空
```
public RespRet bizDeal(RequestVo requestVo) {
        //字段校验,对version、userId、sysId进行基本字段校验
        boolean validRet = ValidUtil.validXXXReq(requestVo,["version","userId","sysId"]);
        if(validRet){
            // 处理业务逻辑    
        }else{
            // 组装并返回返回错误信息
        }
    }

```
校验方法ValidUtil#validXXXReq
```
/**
 *@return 校验通过返回true,校验不通过返回fasle
 */
public static boolean  validXXXReq(Object beValidObejct,String[] paramas) {
        //对beValidObejct进行field反射，取params中的字段进行判空等处理
}

```

##### 优点
逻辑公用，可重用度高。对于大多数业务场景下的判空可以实现。而对于业务特有的逻辑可以结合现状1和现状2进行处理。实际开发中大多数人也数这种方法用的最多

##### 缺点
不够灵活，调用了判空之后不能满足调用者全部的需求需要自己再行开发别的判断逻辑

### 解决方案
基于现状3的缺点，我们自然而然想到，如果想要灵活那么我把规则开发给调用者自己去配置就能实现。那么怎么开放给你调用者呢。对调用者侵入最小的方式可能就是使用Annotation了。那么我们就可以根据不同的Annotation去进行不同的判断。比如定义了@ParamNullcheck的Annotation当我们反射去判断的时候就可以判空。比如定义了@ParamNumberCheck的Annotation当我们反射取判断的时候就可以判断数字是否为空、为0。
```
@Retention(RetentionPolicy.RUNTIME)
@Target({java.lang.annotation.ElementType.FIELD})
public @interface ParamNullcheck {
}

@Retention(RetentionPolicy.RUNTIME)
@Target({java.lang.annotation.ElementType.FIELD})
public @interface ParamNumberCheck{
}
```
更进一步，我们可以支持表达式运算的注解，比如@ParamExpressionCheck,在使用的时候可以判断表达式判断，参考 Aviator 能扩展出更复杂的表达式
```
@Retention(RetentionPolicy.RUNTIME)
@Target({java.lang.annotation.ElementType.FIELD})
public @interface ParamExpressionCheck {
    String value();
}

```
使用的时候，在需要被我们校验的java对象上使用我们的注解，当我们反射校验的时候读取注解就知道我们该做什么样的判断，甚至可以做到该返回什么信息，甚至我们可以绑定一个校验不通过的时候的处理类(如CustomDataErroHander)返回结果处理。但是要甚用，或者说用一个比较通用的处理类。避免本末倒置。我们的出发点是为了让调用方更简单，而不是更复杂。
```
//需要校验的Java对象
public class BizRequestVo {
	@ParamNullcheck
	@ErrorDesc(errorMsg = "data is null !")//自定义返回结果
    private String data;
}

//需要校验的Java对象
public class BizRequestVo {
	@ParamExpressionCheck(">5")
	@ErrorDesc(errorMsg = "data不大于5，错误!")//自定义返回结果
    private int data;
}

//需要校验的Java对象
public class BizRequestVo {
    // 不满足>5 这个条件的时候调用CustomDataErroHander.handle(bizRequestVo)方法
	@ParamExpressionCheck(">5")
	@ParamValidHandler("com.XXX.XXX.param.check.handler.CustomDataErroHander")//自定义返回结果
    private int data;
}

/**
 *自定义字段校验处理接口
 */
public interface ParamErrorHander{
}

/**
 *自定义字段校验处理类
 */
class CustomDataErroHander implements ParamErrorHander{
    @Override
    public void handle(Object beValidObject){
        //打印日志、如操作日志、校验日志等等
        //做一些业务逻辑。。。任何你像做的事情
        //抛出自定义异常
    }
}
```
除了表达式以外我们可以配置自定义的校验类(如：	@ParamValidHandler("com.XXX.XXX.param.check.handler.CustomDataChecker"))，但是同样要避免本末倒置。如果为这些自定义的字段定义的校验处理类多的话就变得复杂了，只有很复杂的业务逻辑校验比如依赖有别的RPC交互或者与库进行交互的时候才做这种判断，但是这个时候也考虑将这些逻辑放到主流程去。同样原则是尽量抽取通用的逻辑做这种处理。所以大多数应用场景下我们使用自定义返回码和返回结果比较多，也足够了。更多定制的部分放到业务处理方法本身去处理。结合现状1，2

```
//这种方法比较实用
@ParamNullcheck
@ErrorDesc(errorMsg = "data is null !")//自定义返回结果
```

除了表达式以外另外我们考虑常见的一种通用校验，既多样化又不至于过“重”，那就是正则表达式。

以上仅仅是我自己的开发过程中遇到的问题和一些
//未完待续