+++
title = "RestTemplate 跳过SSL证书验证"
date = "2019-07-09 16:44:00"
url = "archives/635"
tags = ["Spring","Java"]
categories = ["后端"]
+++

在使用`RestTemplate`请求接口的过程中，遇到`HTTPS`请求又没有证书的情况，只能通过配置来忽略证书验证了

```kotlin
package com.ewei.custom.yto.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.client.ClientHttpRequestFactory
import org.springframework.http.client.SimpleClientHttpRequestFactory
import org.springframework.web.client.RestTemplate
import java.net.HttpURLConnection
import java.security.SecureRandom
import java.security.cert.X509Certificate
import javax.net.ssl.HttpsURLConnection
import javax.net.ssl.SSLContext
import javax.net.ssl.SSLSocketFactory
import javax.net.ssl.X509TrustManager


/**
 * @author wuwenze
 * @date 2019-06-21
 */
@Configuration
class RestTemplateClientConfig {

    @Bean
    fun restTemplate(factory: ClientHttpRequestFactory): RestTemplate {
        return RestTemplate(factory)
    }

    @Bean
    fun simpleClientHttpRequestFactory(): ClientHttpRequestFactory {
        val factory = SkipSSLSimpleClientHttpRequestFactory()
        factory.setReadTimeout(30000)
        factory.setConnectTimeout(30000)
        return factory
    }

    class SkipSSLSimpleClientHttpRequestFactory : SimpleClientHttpRequestFactory() {
        override fun prepareConnection(connection: HttpURLConnection, httpMethod: String) {
            if (connection is HttpsURLConnection) {
                try {
                    connection.setHostnameVerifier { _, _ -> true }
                    connection.sslSocketFactory = createSslSocketFactory()
                } catch (e: Throwable) {
                    // ignore
                }
            }
            super.prepareConnection(connection, httpMethod)
        }

        private fun createSslSocketFactory(): SSLSocketFactory {
            val context: SSLContext = SSLContext.getInstance("TLS")
            context.init(null, arrayOf(SkipX509TrustManager()), SecureRandom())
            return context.socketFactory
        }

        class SkipX509TrustManager : X509TrustManager {
            override fun getAcceptedIssuers(): Array<X509Certificate> = arrayOf()
            override fun checkClientTrusted(chain: Array<out X509Certificate>?, authType: String?) {}
            override fun checkServerTrusted(chain: Array<out X509Certificate>?, authType: String?) {}
        }
    }
}
```