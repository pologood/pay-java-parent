
#pay-java-parent

##整合支付模块（微信支付，支付宝）


###特性

    1.支付请求调用支持HTTP和异步
    2.控制层统一异常处理
    3.LogBack日志记录
    4.简单快速完成支付模块的开发
    5.支持多种支付类型多支付账户扩展（目前已支持微信支付，支付宝支付，友店支付）
    6.支持http代理

###使用  
这里不多说直接上代码  集群的话,友店可能会有bug。暂未测试集群环境

### 快速入门
#####1.支付整合配置
```java


/**
 * 支付类型
 * @author egan
 * @email egzosn@gmail.com
 * @date 2016/11/20 0:30
 */
public enum PayType implements BasePayType{

    aliPay{
        @Override
        public PayService getPayService(ApyAccount apyAccount) {
            AliPayConfigStorage aliPayConfigStorage = new AliPayConfigStorage();
            aliPayConfigStorage.setPartner(apyAccount.getPartner());
            aliPayConfigStorage.setAliPublicKey(apyAccount.getPublicKey());
            aliPayConfigStorage.setKeyPrivate(apyAccount.getPrivateKey());
            aliPayConfigStorage.setNotifyUrl(apyAccount.getNotifyUrl());
            aliPayConfigStorage.setSignType(apyAccount.getSignType());
            aliPayConfigStorage.setSeller(apyAccount.getSeller());
            aliPayConfigStorage.setPayType(apyAccount.getPayType().toString());
            aliPayConfigStorage.setMsgType(apyAccount.getMsgType());
            aliPayConfigStorage.setInputCharset(apyAccount.getInputCharset());
            return new AliPayService(aliPayConfigStorage);
        }

        @Override
        public TransactionType getTransactionType(String transactionType) {
            return AliTransactionType.valueOf(transactionType);
        }


    },wxPay {
        @Override
        public PayService getPayService(ApyAccount apyAccount) {
            WxPayConfigStorage wxPayConfigStorage = new WxPayConfigStorage();
            wxPayConfigStorage.setMchId(apyAccount.getPartner());
            wxPayConfigStorage.setAppSecret(apyAccount.getPublicKey());
            wxPayConfigStorage.setAppid(apyAccount.getAppid());
            wxPayConfigStorage.setKeyPrivate(apyAccount.getPrivateKey());
            wxPayConfigStorage.setNotifyUrl(apyAccount.getNotifyUrl());
            wxPayConfigStorage.setSignType(apyAccount.getSignType());
            wxPayConfigStorage.setPayType(apyAccount.getPayType().toString());
            wxPayConfigStorage.setMsgType(apyAccount.getMsgType());
            wxPayConfigStorage.setInputCharset(apyAccount.getInputCharset());
            return  new WxPayService(wxPayConfigStorage);
        }

        /**
         * 根据支付类型获取交易类型
         * @param transactionType 类型值
         * @see WxTransactionType
         * @return
         */
        @Override
        public TransactionType getTransactionType(String transactionType) {

            return WxTransactionType.valueOf(transactionType);
        }
    },youdianPay {
        @Override
        public PayService getPayService(ApyAccount apyAccount) {
           // TODO 2017/1/23 14:12 author: egan  集群的话,友店可能会有bug。暂未测试集群环境
            WxYouDianPayConfigStorage wxPayConfigStorage = new WxYouDianPayConfigStorage();
            wxPayConfigStorage.setKeyPrivate(apyAccount.getPrivateKey());
            wxPayConfigStorage.setNotifyUrl(apyAccount.getNotifyUrl());
            wxPayConfigStorage.setSignType(apyAccount.getSignType());
            wxPayConfigStorage.setPayType(apyAccount.getPayType().toString());
            wxPayConfigStorage.setMsgType(apyAccount.getMsgType());
            wxPayConfigStorage.setSeller(apyAccount.getSeller());
            wxPayConfigStorage.setInputCharset(apyAccount.getInputCharset());
            return  new WxYouDianPayService(wxPayConfigStorage);
        }

        /**
         * 根据支付类型获取交易类型
         * @param transactionType 类型值
         * @see YoudianTransactionType
         * @return
         */
        @Override
        public TransactionType getTransactionType(String transactionType) {

            return YoudianTransactionType.valueOf(transactionType);
        }
    };

    public abstract PayService getPayService(ApyAccount apyAccount);


}

/**
 * 支付响应对象
 * @author: egan
 * @email egzosn@gmail.com
 * @date 2016/11/18 0:34
 */
public class PayResponse {
    @Resource
    private AutowireCapableBeanFactory spring;

    private PayConfigStorage storage;

    private PayService service;

    private PayMessageRouter router;

    public PayResponse() {

    }

    /**
     * 初始化支付配置
     * @param apyAccount 账户信息
     * @see ApyAccount 对应表结构详情--》 pay-java-demo/resources/apy_account.sql
     */
    public void init(ApyAccount apyAccount) {

        //根据不同的账户类型 初始化支付配置
        this.service = apyAccount.getPayType().getPayService(apyAccount);
        this.storage = service.getPayConfigStorage();

        buildRouter(apyAccount.getPayId());
    }





    /**
     * 配置路由
     * @param payId 指定账户id，用户多微信支付多支付宝支付
     */
      private void buildRouter(Integer payId) {
            router = new PayMessageRouter(this.service);
            router
                    .rule()
                    .async(false)
                    .msgType(MsgType.text.name()) //消息类型
                    .event(PayType.aliPay.name()) //支付账户事件类型
                    .interceptor(new AliPayMessageInterceptor()) //拦截器
                    .handler(autowire(new AliPayMessageHandler(payId))) //处理器
                    .end()
                    .rule()
                    .async(false)
                    .msgType(MsgType.xml.name())
                    .event(PayType.wxPay.name())
                    .handler(autowire(new WxPayMessageHandler(payId)))
                    .end()
            ;
      }

    
    private PayMessageHandler autowire(PayMessageHandler handler) {
        spring.autowireBean(handler);
        return handler;
    }

    public PayConfigStorage getStorage() {
        return storage;
    }
    
    public PayService getService() {
        return service;
    }

    public PayMessageRouter getRouter() {
        return router;
    }
}

```

#####2.支付处理器与拦截器简单实现

```java
    /**
     * 微信支付回调处理器
     * Created by ZaoSheng on 2016/6/1.
     */
    public class WxPayMessageHandler extends BasePayMessageHandler {
        public WxPayMessageHandler(Integer payId) {
            super(payId);
        }
        @Override
        public PayOutMessage handle(PayMessage payMessage, Map<String, Object> context, PayService payService) throws PayErrorException {
            //交易状态
            if ("SUCCESS".equals(payMessage.getPayMessage().get("result_code"))){
                /////这里进行成功的处理

                return  payService.getPayOutMessage("SUCCESS", "OK");
            }

            return  payService.getPayOutMessage("FAIL", "失败");
        }
    }

    /**
     * 支付宝回调信息拦截器
     * @author: egan
     * @email egzosn@gmail.com
     * @date 2017/1/18 19:28
     */
    public class AliPayMessageInterceptor implements PayMessageInterceptor {
        /**
         * 拦截支付消息
         *
         * @param payMessage     支付回调消息
         * @param context        上下文，如果handler或interceptor之间有信息要传递，可以用这个
         * @param payService
         * @return true代表OK，false代表不OK并直接中断对应的支付处理器
         * @see PayMessageHandler 支付处理器
         */
        @Override
        public boolean intercept(PayMessage payMessage, Map<String, Object> context, PayService payService) throws PayErrorException {

            //这里进行拦截器处理，自行实现
            return true;
        }
    }

```


#####3.支付响应PayResponse的获取


```java


public class ApyAccountService {


    @Inject
    private ApyAccountDao dao;

    @Inject
    private AutowireCapableBeanFactory spring;

    private final static Map<Integer, PayResponse> payResponses = new HashMap<Integer, PayResponse>();


    /**
     *  获取支付响应
     * @param id 账户id
     * @return
     */
    public PayResponse getPayResponse(Integer id) {

        PayResponse payResponse = payResponses.get(id);
        if (payResponse  == null) {
            ApyAccount apyAccount = dao.get(id);
            if (apyAccount == null) {
               throw new IllegalArgumentException ("无法查询");
            }
            payResponse = new PayResponse();
            spring.autowireBean(payResponse);
            payResponse.init(apyAccount);
            payResponses.put(id, payResponse);
            // 查询
        }
        return payResponse;
    }



}

```


#####4.根据账户id与业务id，组拼订单信息（支付宝、微信支付订单）获取支付信息所需的数据

```java
    /**
     *  获取支付预订单信息
     * @param payId 支付账户id
     * @param transactionType 交易类型
     * @return
     */
    @RequestMapping("getOrderInfo")
    public Object getOrderInfo( Integer payId, String transactionType){
        //获取对应的支付账户操作工具（可根据账户id）
        PayResponse payResponse =  service.getPayResponse(payId);;

        //这里之所以用Object，因为微信需返回Map， 支付吧String。
        Object orderInfo = payResponse.getService().orderInfo(new PayOrder("订单title", "摘要", new BigDecimal(0.01), "tradeNo", PayType.valueOf(payResponse.getStorage().getPayType()).getTransactionType(transactionType)));

        return orderInfo;
    }
  
```

#####5.支付回调
```java
     

    /**
     * 微信或者支付宝回调地址
     * @param request
     * @return
     */
    @ResponseBody
      @RequestMapping(value = "payBack{payId}.json")
      public String payBack(HttpServletRequest request, @PathVariable Integer payId) throws IOException {
          //根据账户id，获取对应的支付账户操作工具
          PayResponse payResponse = service.getPayResponse(payId);
          PayConfigStorage storage = payResponse.getStorage();
            //
          Map<String, String> params = payResponse.getService().getParameter2Map(request.getParameterMap(), request.getInputStream());
          if (null == params){
              return payResponse.getService().getPayOutMessage("fail","失败").toMessage();
          }

          //校验
          if (payResponse.getService().verify(params)){
              PayMessage message = new PayMessage(params, storage.getPayType(), storage.getMsgType().name());
              PayOutMessage outMessage = payResponse.getRouter().route(message);
              return outMessage.toMessage();
          }

          return payResponse.getService().getPayOutMessage("fail","失败").toMessage();
      }


        
```

##交流
很希望更多志同道合友友一起扩展新的的支付接口。

非常欢迎和感谢对本项目发起Pull Request的同学，不过本项目基于git flow开发流程，因此在发起Pull Request的时候请选择develop分支。

E-Mail：egzosn@gmail.com

QQ群：542193977

