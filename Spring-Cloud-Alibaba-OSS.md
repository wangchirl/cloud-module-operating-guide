#### 使用阿里云OSS文件存储说明

##### 一、开通阿里云OSS服务

> - 使用钉钉账号或者支付宝账号登陆阿里云
> - 左边菜单栏找到OSS对象存储服务
> - 开通OSS对象存储服务
>
> [官网连接](https://help.aliyun.com/document_detail/31884.htm)

##### 二、OSS组件说明

> - **Bucket** ：前端访问地址
> - **endpoint** ：后端请求地址
> - **accessKeyId** ：通行证ID
> - **accessKeySecret** ：通行证密钥

##### 三、Java使用OOS

> - 简单上传
>
>   - pom.xml
>
>     ```xml
>     <dependency>
>         <groupId>com.aliyun.oss</groupId>
>         <artifactId>aliyun-sdk-oss</artifactId>
>         <version>3.8.0</version>
>     </dependency>
>     ```
>
>   - java 测试代码
>
>     ```java
>     // Endpoint以杭州为例，其它Region请按实际情况填写。
>     String endpoint = "http://oss-cn-hangzhou.aliyuncs.com";
>     // 云账号AccessKey有所有API访问权限，建议遵循阿里云安全最佳实践，创建并使用RAM子账号进行API访问或日常运维，请登录 https://ram.console.aliyun.com 创建。
>     String accessKeyId = "<yourAccessKeyId>";
>     String accessKeySecret = "<yourAccessKeySecret>";
>     
>     // 创建OSSClient实例。
>     OSS ossClient = new OSSClientBuilder().build(endpoint, accessKeyId, accessKeySecret);
>     
>     // 上传文件流。
>     InputStream inputStream = new FileInputStream("<yourlocalFile>");
>     ossClient.putObject("<yourBucketName>", "<yourObjectName>", inputStream);
>     
>     // 关闭OSSClient。
>     ossClient.shutdown();
>     ```
>
>   - 更多方式参考
>
>     [官方文档](https://help.aliyun.com/document_detail/84781.html)
>
> - 签名直传[只发送请求到我们服务获取签名]
>
>   - pom.xml
>
>     ```xml
>     <!-- 阿里云 OSS -->
>     <dependency>
>         <groupId>com.alibaba.cloud</groupId>
>         <artifactId>spring-cloud-starter-alicloud-oss</artifactId>
>     </dependency>
>     
>     <dependencyManagement>
>         <dependencies>
>             <dependency>
>                 <groupId>org.springframework.cloud</groupId>
>                 <artifactId>spring-cloud-dependencies</artifactId>
>                 <version>${spring-cloud.version}</version>
>                 <type>pom</type>
>                 <scope>import</scope>
>             </dependency>
>             <dependency>
>                 <groupId>com.alibaba.cloud</groupId>
>                 <artifactId>spring-cloud-alibaba-dependencies</artifactId>
>                 <version>2.2.0.RELEASE</version>
>                 <type>pom</type>
>                 <scope>import</scope>
>             </dependency>
>         </dependencies>
>     </dependencyManagement>
>     
>     ```
>
>   - java
>
>     ```java
>     package com.shadow.mall.third.controller;
>     
>     import com.aliyun.oss.OSS;
>     import com.aliyun.oss.OSSClient;
>     import com.aliyun.oss.common.utils.BinaryUtil;
>     import com.aliyun.oss.model.MatchMode;
>     import com.aliyun.oss.model.PolicyConditions;
>     import com.shadow.common.utils.R;
>     import org.springframework.beans.factory.annotation.Autowired;
>     import org.springframework.beans.factory.annotation.Value;
>     import org.springframework.web.bind.annotation.GetMapping;
>     import org.springframework.web.bind.annotation.RestController;
>     
>     import java.text.SimpleDateFormat;
>     import java.util.Date;
>     import java.util.LinkedHashMap;
>     import java.util.Map;
>     
>     @RestController
>     public class OSSController {
>     
>     
>         @Autowired
>         OSS ossClient;
>     
>         @Value("${spring.cloud.alicloud.oss.endpoint}")
>         private String endpoint;
>     
>         @Value("${spring.cloud.alicloud.oss.bucket}")
>         private String bucket;
>     
>         @Value("${spring.cloud.alicloud.access-key}")
>         private String accessId;
>     
>         /**
>          * OSS 签名直传服务
>          * @return
>          */
>         @GetMapping("/oss/policy")
>         public R policy() {
>     
>             String host = "https://" + bucket + "." + endpoint; // host的格式为 bucketname.endpoint
>             // callbackUrl为 上传回调服务器的URL，请将下面的IP和Port配置为您自己的真实信息。
>             //        String callbackUrl = "http://88.88.88.88:8888";
>             String format = new SimpleDateFormat("yyyy-MM-dd").format(new Date());
>             String dir = format + "/"; // 用户上传文件时指定的前缀。
>             Map<String, String> respMap = new LinkedHashMap<String, String>();
>             try {
>                 long expireTime = 30;
>                 long expireEndTime = System.currentTimeMillis() + expireTime * 1000;
>                 Date expiration = new Date(expireEndTime);
>                 PolicyConditions policyConds = new PolicyConditions();
>                 policyConds.addConditionItem(PolicyConditions.COND_CONTENT_LENGTH_RANGE, 0, 1048576000);
>                 policyConds.addConditionItem(MatchMode.StartWith, PolicyConditions.COND_KEY, dir);
>     
>                 String postPolicy = ossClient.generatePostPolicy(expiration, policyConds);
>                 byte[] binaryData = postPolicy.getBytes("utf-8");
>                 String encodedPolicy = BinaryUtil.toBase64String(binaryData);
>                 String postSignature = ossClient.calculatePostSignature(postPolicy);
>     
>     
>                 respMap.put("accessid", accessId);
>                 respMap.put("policy", encodedPolicy);
>                 respMap.put("signature", postSignature);
>                 respMap.put("dir", dir);
>                 respMap.put("host", host);
>                 respMap.put("expire", String.valueOf(expireEndTime / 1000));
>                 // respMap.put("expire", formatISO8601Date(expiration));
>     
>             } catch (Exception e) {
>                 // Assert.fail(e.getMessage());
>                 System.out.println(e.getMessage());
>             }
>     
>             return R.ok().put("data",respMap);
>         }
>     }
>     ```
>
>   - 注册中心配置
>
>     ```yaml
>     
>     spring:
>     # 每个微服务必须设置名称
>       application:
>         name: shadow-third-party
>       cloud:
>         nacos:
>           discovery:
>             server-addr: localhost:8848 # nacos服务注册中心地址
>     
>         alicloud: # OSS配置 - 这里可以不配置 直接使用统一配置中心配置的
>             access-key: LTAI4Fc1uRrRjBSPDw9YGaFep
>             secret-key: 5QSzN7uHD2sKb108ovtPsDZsVA4kG0
>             oss:
>                 endpoint: oss-cn-shanghai.aliyuncs.com
>                 bucket:  shadowmall-hello #自定义bucket名称
>     
>     server:
>       port: 30000
>     
>     ```
>
>   - 统一配置中心配置 bootstrap.properties
>
>     ```properties
>     spring.cloud.nacos.config.server-addr=localhost:8848
>     spring.cloud.nacos.config.namespace=a98633ab-e792-4a93-904f-644fdbeba2a7
>     
>     spring.cloud.nacos.config.extension-configs[0].data-id=oss.yml
>     spring.cloud.nacos.config.extension-configs[0].group=DEFAULT_GROUP
>     spring.cloud.nacos.config.extension-configs[0].refresh=true
>     ```
>
>   - nacos统一配置中心配置信息 oos.yml
>
>     ```yaml
>     spring:
>         cloud:
>             alicloud: # OSS配置
>                 access-key: LTAI4Fc1uRrRjBSPDw9YGaFep
>                 secret-key: 5QSzN7uHD2sKb108ovtPsDZsVA4kG0
>                 oss:
>                     endpoint: oss-cn-shanghai.aliyuncs.com
>     ```
>
>   [官方文档](https://help.aliyun.com/document_detail/31926.html)

