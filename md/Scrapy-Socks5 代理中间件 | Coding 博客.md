> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [leeyongda.github.io](https://leeyongda.github.io/coding/2018/01/05/Scrapy-Socks5%E4%BB%A3%E7%90%86%E4%B8%AD%E9%97%B4%E4%BB%B6/)

> Scrapy-Socks5 代理中间件 Demo 环境： Python(2.7+) + Scrapy(1.8.0) + Twisted(19.7.0) 官网没直接提供 Socks 代理中间件 。

环境： Python(2.7+) + Scrapy(1.1.1) + Twisted(19.7.0)  
官网没直接提供 Socks 代理中间件 。  
所以写一个代理中间件 。  
需要依赖库 txsocksx 。  
pip install txsocksx  

```
# -*- coding: utf-8 -*-
# 需要依赖 txsocksx
# pip install txsocksx


from txsocksx.http import SOCKS5Agent
from twisted.internet import reactor
from twisted.internet.endpoints import TCP4ClientEndpoint
from scrapy.core.downloader.webclient import _parse
from scrapy.core.downloader.handlers.http11 import HTTP11DownloadHandler, ScrapyAgent

proxyHost = "xxxx.com"
proxyPort = 9020
proxyUser = "1234"
proxyPass = "pass"

class Socks5DownloadHandler(HTTP11DownloadHandler):

    def download_request(self, request, spider):
        """Return a deferred for the HTTP download"""
        agent = ScrapySocks5Agent(contextFactory=self._contextFactory, pool=self._pool)
        return agent.download_request(request)

class ScrapySocks5Agent(ScrapyAgent):

    def _get_agent(self, request, timeout):
       # bindAddress = request.meta.get('bindaddress') or self._bindAddress
        #proxy = request.meta.get('proxy')
        #if proxy:
            #_, _, proxyHost, proxyPort, proxyParams = _parse(proxy)
            #_, _, host, port, proxyParams = _parse(request.url)
        proxyEndpoint = TCP4ClientEndpoint(reactor, proxyHost, proxyPort, endpointArgs=dict(methods={'login': {proxyUser, proxyPass}}))
        agent = SOCKS5Agent(reactor, proxyEndpoint=proxyEndpoint)
        return agent
       # return self._Agent(reactor, contextFactory=self._contextFactory,
        #    connectTimeout=timeout, bindAddress=bindAddress, pool=self._pool)
```
