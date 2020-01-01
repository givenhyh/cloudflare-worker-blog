# 如何充分使用CloudFlare的免费账户
想必各位朋友如果做网站，肯定接触或者听说过CloudFlare这个名词吧，今天我们就来看看怎么把CF免费账户的价值榨干
## 1. 使用CF workers进行反向代理，搭建博客
大家应该都知道，CF出了一个worker服务，可以让你免费运行serverless代码，比如制作反向代理，搭建博客等，只要能用node.js实现理论都可以运行，每日免费请求量是20万，够一个小型站点一天使用了。
源码
1.反代

```'use strict'

/**
 * static files (404.html, sw.js, conf.js)
 */
const ASSET_URL = '要访问的地址'

const JS_VER = 10
const MAX_RETRY = 1

/** @type {RequestInit} */
const PREFLIGHT_INIT = {
  status: 204,
  headers: new Headers({
    'access-control-allow-origin': '*',
    'access-control-allow-methods': 'GET,POST,PUT,PATCH,TRACE,DELETE,HEAD,OPTIONS',
    'access-control-max-age': '1728000',
  }),
}

/**
 * @param {any} body
 * @param {number} status
 * @param {Object<string, string>} headers
 */
function makeRes(body, status = 200, headers = {}) {
  headers['--ver'] = JS_VER
  headers['access-control-allow-origin'] = '*'
  return new Response(body, {status, headers})
}


/**
 * @param {string} urlStr 
 */
function newUrl(urlStr) {
  try {
    return new URL(urlStr)
  } catch (err) {
    return null
  }
}


addEventListener('fetch', e => {
  const ret = fetchHandler(e)
    .catch(err => makeRes('cfworker error:\n' + err.stack, 502))
  e.respondWith(ret)
})


/**
 * @param {FetchEvent} e 
 */
async function fetchHandler(e) {
  const req = e.request
  const urlStr = req.url
  const urlObj = new URL(urlStr)
  const path = urlObj.href.substr(urlObj.origin.length)

  if (urlObj.protocol === 'http:') {
    urlObj.protocol = 'https:'
    return makeRes('', 301, {
      'strict-transport-security': 'max-age=99999999; includeSubDomains; preload',
      'location': urlObj.href,
    })
  }

  if (path.startsWith('/http/')) {
    return httpHandler(req, path.substr(6))
  }

  switch (path) {
  case '/http':
    return makeRes('请更新 cfworker 到最新版本!')
  case '/ws':
    return makeRes('not support', 400)
  case '/works':
    return makeRes('it works')
  default:
    // static files
    return fetch(ASSET_URL + path)
  }
}


/**
 * @param {Request} req
 * @param {string} pathname
 */
function httpHandler(req, pathname) {
  const reqHdrRaw = req.headers
  if (reqHdrRaw.has('x-jsproxy')) {
    return Response.error()
  }

  // preflight
  if (req.method === 'OPTIONS' &&
      reqHdrRaw.has('access-control-request-headers')
  ) {
    return new Response(null, PREFLIGHT_INIT)
  }

  let acehOld = false
  let rawSvr = ''
  let rawLen = ''
  let rawEtag = ''

  const reqHdrNew = new Headers(reqHdrRaw)
  reqHdrNew.set('x-jsproxy', '1')

  // 此处逻辑和 http-dec-req-hdr.lua 大致相同
  // https://github.com/EtherDream/jsproxy/blob/master/lua/http-dec-req-hdr.lua
  const refer = reqHdrNew.get('referer')
  const query = refer.substr(refer.indexOf('?') + 1)
  if (!query) {
    return makeRes('missing params', 403)
  }
  const param = new URLSearchParams(query)

  for (const [k, v] of Object.entries(param)) {
    if (k.substr(0, 2) === '--') {
      // 系统信息
      switch (k.substr(2)) {
      case 'aceh':
        acehOld = true
        break
      case 'raw-info':
        [rawSvr, rawLen, rawEtag] = v.split('|')
        break
      }
    } else {
      // 还原 HTTP 请求头
      if (v) {
        reqHdrNew.set(k, v)
      } else {
        reqHdrNew.delete(k)
      }
    }
  }
  if (!param.has('referer')) {
    reqHdrNew.delete('referer')
  }

  // cfworker 会把路径中的 `//` 合并成 `/`
  const urlStr = pathname.replace(/^(https?):\/+/, '$1://')
  const urlObj = newUrl(urlStr)
  if (!urlObj) {
    return makeRes('invalid proxy url: ' + urlStr, 403)
  }

  /** @type {RequestInit} */
  const reqInit = {
    method: req.method,
    headers: reqHdrNew,
    redirect: 'manual',
  }
  if (req.method === 'POST') {
    reqInit.body = req.body
  }
  return proxy(urlObj, reqInit, acehOld, rawLen, 0)
}


/**
 * 
 * @param {URL} urlObj 
 * @param {RequestInit} reqInit 
 * @param {number} retryTimes 
 */
async function proxy(urlObj, reqInit, acehOld, rawLen, retryTimes) {
  const res = await fetch(urlObj.href, reqInit)
  const resHdrOld = res.headers
  const resHdrNew = new Headers(resHdrOld)

  let expose = '*'
  
  for (const [k, v] of resHdrOld.entries()) {
    if (k === 'access-control-allow-origin' ||
        k === 'access-control-expose-headers' ||
        k === 'location' ||
        k === 'set-cookie'
    ) {
      const x = '--' + k
      resHdrNew.set(x, v)
      if (acehOld) {
        expose = expose + ',' + x
      }
      resHdrNew.delete(k)
    }
    else if (acehOld &&
      k !== 'cache-control' &&
      k !== 'content-language' &&
      k !== 'content-type' &&
      k !== 'expires' &&
      k !== 'last-modified' &&
      k !== 'pragma'
    ) {
      expose = expose + ',' + k
    }
  }

  if (acehOld) {
    expose = expose + ',--s'
    resHdrNew.set('--t', '1')
  }

  // verify
  if (rawLen) {
    const newLen = resHdrOld.get('content-length') || ''
    const badLen = (rawLen !== newLen)

    if (badLen) {
      if (retryTimes < MAX_RETRY) {
        urlObj = await parseYtVideoRedir(urlObj, newLen, res)
        if (urlObj) {
          return proxy(urlObj, reqInit, acehOld, rawLen, retryTimes + 1)
        }
      }
      return makeRes(res.body, 400, {
        '--error': `bad len: ${newLen}, except: ${rawLen}`,
        'access-control-expose-headers': '--error',
      })
    }

    if (retryTimes > 1) {
      resHdrNew.set('--retry', retryTimes)
    }
  }

  let status = res.status

  resHdrNew.set('access-control-expose-headers', expose)
  resHdrNew.set('access-control-allow-origin', '*')
  resHdrNew.set('--s', status)
  resHdrNew.set('--ver', JS_VER)

  resHdrNew.delete('content-security-policy')
  resHdrNew.delete('content-security-policy-report-only')
  resHdrNew.delete('clear-site-data')

  if (status === 301 ||
      status === 302 ||
      status === 303 ||
      status === 307 ||
      status === 308
  ) {
    status = status + 10
  }

  return new Response(res.body, {
    status,
    headers: resHdrNew,
  })
}


/**
 * @param {URL} urlObj 
 */
function isYtUrl(urlObj) {
  return (
    urlObj.host.endsWith('.googlevideo.com') &&
    urlObj.pathname.startsWith('/videoplayback')
  )
}

/**
 * @param {URL} urlObj 
 * @param {number} newLen 
 * @param {Response} res 
 */
async function parseYtVideoRedir(urlObj, newLen, res) {
  if (newLen > 2000) {
    return null
  }
  if (!isYtUrl(urlObj)) {
    return null
  }
  try {
    const data = await res.text()
    urlObj = new URL(data)
  } catch (err) {
    return null
  }
  if (!isYtUrl(urlObj)) {
    return null
  }
  return urlObj
}![]()
```
2.博客
源代码[GayHub](https://github.com)上一堆，不发了
## 2. 手动指定cf节点ip来达到最佳cdn效果
此操作使用于用CloudFlare Partner接入的用户
添加网站之后，通过A记录解析到这些IP：

```
108.162.236.1/24 联通 走美国
172.64.32.1/24 移动 走香港
104.16.160.1/24 电信 走美国洛杉矶
---------
172.64.0.0/24 电信 美国旧金山
104.20.157.0/24 联通 走日本
104.28.14.0/24 移动 走新加坡
 （联通移动推荐节点）
 104.23.240.0-104.23.243.254
 （电信推荐百度云合作ip）
 162.159.208.4-162.159.208.103
 162.159.209.4-162.159.209.103
 162.159.210.4-162.159.210.103
 162.159.211.4-162.159.211.103
```
速度快的：

```
104.20.157.2 
104.18.62.2 
141.101.115.3 
104.16.160.3
```
百度云加速合作节点：

```
162.159.211.4-103
103.21.244.0/22
103.22.200.0/22
103.31.4.0/22
104.16.0.0/12
108.162.192.0/18
131.0.72.0/22
141.101.64.0/18
162.158.0.0/15
172.64.0.0/13
173.245.48.0/20
188.114.96.0/20
190.93.240.0/20
197.234.240.0/22
198.41.128.0/17
```
其余节点

```
#适合电信的节点
104.23.240.*
#走欧洲各国出口 英国德国荷兰等 延迟比美国高一些 适合源站在欧洲的网站
172.64.32.*
#虽然去程走新加坡，但是回程线路的绕路的，实际效果不好，不推荐
104.16.160.*
#圣何塞的线路，比洛杉矶要快一点，推荐
108.162.236.*
#亚特兰大线路，延迟稳定，但是延迟较高
#适合移动的节点
162.158.133.* 
#走的丹麦，这一段ip只有部分能用，可以自己试一下。绕美国。
198.41.214.*
198.41.212.*
198.41.208.*
198.41.209.*
172.64.32.*
141.101.115.*
#移动走香港的IP段有很多，以上并不是全部。CF移动走香港的分直连和走ntt的效果都挺不错的，不过部分地区晚上还是会丢包。
172.64.0. *
#这是走圣何塞的，一般用香港的就行
172.64.16.* 
#欧洲线路.绕
 #1.0.0.1效果较好
电信部分
大多数省直接使用1.0.0.0即可，延迟低，丢包少，
 #移动部分
#新加坡
 104.18.48.0-104.18.63.255
104.24.112.0-104.24.127.255
104.27.128.0-104.27.143.255
104.28.0.0-104.28.15.255
 #移动部分
#圣何塞 
104.28.16.0-31.255
104.27.144.0-243.254
104.23.240.0-243.254
 #香港cloudflare1-100g.hkix.net
1.0.0.0-254
1.1.1.0-254
 #香港直连
104.16.0.0-79.255
104.16.96.0-175.254
104.16.192.0-207.255
```
