---
title: Https安全机制的实现
date: 2019-09-02 14:12:16
tags: Https
categories: 网络

---

本文记录HTTPS简要原理，实现过程以及iOS上如何对证书进行验证。

### HTTPS
HTTPS是运行在TLS/SSL之上的HTTP，与普通的HTTP相比，在数据传输的安全性上有很大的提升。在传输数据之前有一个协商生成密钥（session key）的过程，这个密钥用来随后传递数据的时候对数据的加密与解密。

#### 生成密钥的操作是如何做到安全的？
简单的来说，采用的非对称加密的方式。server端发送给client一个公钥，客户端用公钥加密随机字符串，并且把这个字符串加密传给server。这里存在一个问题，如果公钥在传给client的过程中被第三方截获，替换了公钥，那么随后的传递的数据都可以被第三方解密然后转发。所以这里server不单单是传一个公钥，而是以证书的方式传给客户端。

#### 证书如何保证了公钥传递的安全性？

0. 客户端拿到的数据要验证域名等**服务器信息**这个信息最后在客户端还会校验；
1. 客户端还需要用来产生对称密钥的**公钥**；
2. 对**服务器信息和公钥**进行打包签名（CA机构的私钥加密）生成一个摘要信息，两者也通过hash算法也生成一个摘要，都打包到证书。客户端收到证书之后，通过对比两个摘要信息（CA机构的公钥解密然后hash，两个hash值对比）判断是否被篡改；

如果第三方截获了证书，替换了公钥或者修改了服务器信息，客户端会摘要对比会校验失败。单纯的获取到了公钥实际上也做不了什么事情，因为没有私钥去解密随后client传给服务器的公钥session key。签名的值（hash值而且被CA私钥加密）对于截获方也没有任何意义，这样就确保了公钥不被篡改。除非截获方像一些抓包工具一样，自己生成一个全新的证书返回给客户端（客户端须要信任这种自颁发的证书）。

如果客户端内置了CA的证书，就会改证书对应的公钥，服务器一开始就向认证中心申请了这些证书了。所以客户端不要随便装一些证书；

#### iOS对证书的验证

证书校验策略可以在以下代理方法里面进行选择。如果我们要实现这个代理方法的话，需要提供 NSURLSessionAuthChallengeDisposition（处置方式）和 NSURLCredential（资格认证）这两个参数给 completionHandler 这个 block。

```
/*
 只要访问的是HTTPS的路径就会调用
 该方法的作用就是处理服务器返回的证书, 需要在该方法中告诉系统是否需要安装服务器返回的证书
 */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * __nullable credential))completionHandler
{
    if (!challenge) {
        return;
    }
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    NSURLCredential *credential = nil;
    /*
     * 获取原始域名信息。
     */
    NSString *host = [[self.mutableRequest allHTTPHeaderFields] objectForKey:@"host"];
    if (!host) {
        host = self.mutableRequest.URL.host;
    }
    /*
     1.从服务器返回的受保护空间中拿到证书的类型
     2.判断服务器返回的证书是否是服务器信任的
     */
    /*
     disposition：如何处理证书
     NSURLSessionAuthChallengePerformDefaultHandling:默认方式处理
     NSURLSessionAuthChallengeUseCredential：使用指定的证书
     NSURLSessionAuthChallengeCancelAuthenticationChallenge：取消请求
     */
    if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        if ([self evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:host]) {
            // 3.确认是服务器信任的证书，根据服务器返回的受保护空间创建一个证书
            disposition = NSURLSessionAuthChallengeUseCredential;
            // 4.创建证书
            credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
        } else {
            disposition = NSURLSessionAuthChallengePerformDefaultHandling;
        }
    } else {
        disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    }
    // 安装证书
    completionHandler(disposition, credential);
}

- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust forDomain:(NSString *)domain
{
    /*
     * 创建证书校验策略
     */
    NSMutableArray *policies = [NSMutableArray array];
    if (domain) {
        [policies addObject:(__bridge_transfer id) SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
    } else {
        [policies addObject:(__bridge_transfer id) SecPolicyCreateBasicX509()];
    }
    /*
     * 绑定校验策略到服务端的证书上
     */
    SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);
    /*
     * 评估当前serverTrust是否可信任，
     * 官方建议在result = kSecTrustResultUnspecified 或 kSecTrustResultProceed
     * 的情况下serverTrust可以被验证通过，https://developer.apple.com/library/ios/technotes/tn2232/_index.html
     * 关于SecTrustResultType的详细信息请参考SecTrust.h
     */
    SecTrustResultType result;
    SecTrustEvaluate(serverTrust, &result);
    return (result == kSecTrustResultUnspecified || result == kSecTrustResultProceed);
}

```
在这里也可以做验证自颁发的证书，打在bundle包里面。


