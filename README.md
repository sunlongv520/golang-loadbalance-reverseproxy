# golang-loadbalance-reverseproxy
基于golang的反向代理 负载均衡

**反向代理：**

![](http://img.ichunt.com/images/%E5%AD%99%E9%BE%99/a7094bfabd164aa7f2058a7f1f2c6a4e.png)

实现一个多层代理

一层代理
go run ichunt_micro_service/test/reverseProxy/one/oneAgent.go

二层代理
go  run  ichunt_micro_service/test/reverseProxy/two/twoAgent.go

真实ip
go run ichunt_micro_service/test/reverseProxy/realServer.go



代理实现

```
func main() {
	rs1 := "http://127.0.0.1:2002"
	url1, err1 := url.Parse(rs1)
	if err1 != nil {
		log.Println(err1)
	}
	urls := []*url.URL{url1}
	proxy := NewMultipleHostsReverseProxy(urls)
	log.Println("Starting httpserver at " + addr)
	log.Fatal(http.ListenAndServe(addr, proxy))
}

var transport = &http.Transport{
	DialContext: (&net.Dialer{
		Timeout:   30 * time.Second, //连接超时
		KeepAlive: 30 * time.Second, //长连接超时时间
	}).DialContext,
	MaxIdleConns:          100,              //最大空闲连接
	IdleConnTimeout:       90 * time.Second, //空闲超时时间
	TLSHandshakeTimeout:   10 * time.Second, //tls握手超时时间
	ExpectContinueTimeout: 1 * time.Second,  //100-continue 超时时间
}


func NewMultipleHostsReverseProxy(targets []*url.URL) *httputil.ReverseProxy {
	//请求协调者
	director := func(req *http.Request) {
		//url_rewrite
		//127.0.0.1:2002/dir/abc ==> 127.0.0.1:2003/base/abc ??
		//127.0.0.1:2002/dir/abc ==> 127.0.0.1:2002/abc
		//127.0.0.1:2002/abc ==> 127.0.0.1:2003/base/abc
		re, _ := regexp.Compile("^/dir(.*)");
		req.URL.Path = re.ReplaceAllString(req.URL.Path, "$1")

		//随机负载均衡
		targetIndex := rand.Intn(len(targets))
		target := targets[targetIndex]
		targetQuery := target.RawQuery
		req.URL.Scheme = target.Scheme
		req.URL.Host = target.Host

		// url地址重写：重写前：/aa 重写后：/base/aa
		req.URL.Path = singleJoiningSlash(target.Path, req.URL.Path)
		if targetQuery == "" || req.URL.RawQuery == "" {
			req.URL.RawQuery = targetQuery + req.URL.RawQuery
		} else {
			req.URL.RawQuery = targetQuery + "&" + req.URL.RawQuery
		}
		if _, ok := req.Header["User-Agent"]; !ok {
			req.Header.Set("User-Agent", "user-agent")
		}
		//只在第一代理中设置此header头
		req.Header.Set("X-Real-Ip", req.RemoteAddr)
	}
	//更改内容
	modifyFunc := func(resp *http.Response) error {
		//请求以下命令：curl 'http://127.0.0.1:2002/error'
		if resp.StatusCode != 200 {
			//获取内容
			oldPayload, err := ioutil.ReadAll(resp.Body)
			if err != nil {
				return err
			}
			//追加内容
			newPayload := []byte("StatusCode error:" + string(oldPayload))
			resp.Body = ioutil.NopCloser(bytes.NewBuffer(newPayload))
			resp.ContentLength = int64(len(newPayload))
			resp.Header.Set("Content-Length", strconv.FormatInt(int64(len(newPayload)), 10))
		}
		return nil
	}
	//错误回调 ：关闭real_server时测试，错误回调
	errFunc := func(w http.ResponseWriter, r *http.Request, err error) {
		http.Error(w, "ErrorHandler error:"+err.Error(), 500)
	}
	return &httputil.ReverseProxy{
		Director:       director,
		Transport:      transport,
		ModifyResponse: modifyFunc,
		ErrorHandler:   errFunc}
}

func singleJoiningSlash(a, b string) string {
	aslash := strings.HasSuffix(a, "/")
	bslash := strings.HasPrefix(b, "/")
	switch {
	case aslash && bslash:
		return a + b[1:]
	case !aslash && !bslash:
		return a + "/" + b
	}
	return a + b
}

```


**反向代理，通过负载均衡获取服务地址**

运行实际服务 
127.0.0.1:2003
127.0.0.1:2004

go  run  ichunt_micro_service/test/reverseProxy/realServer.go  

启动反向代理：内置负载均衡器
go run  ichunt_micro_service/test/load_balance/main.go



//负载均衡获取服务地址
load_balance.LoadBanlanceFactory(load_balance.LbWeightRoundRobin)
```
func main() {
	rb := load_balance.LoadBanlanceFactory(load_balance.LbWeightRoundRobin)
	if err := rb.Add("http://127.0.0.1:2003/", "10"); err != nil {
		log.Println(err)
	}
	if err := rb.Add("http://127.0.0.1:2004/", "20"); err != nil {
		log.Println(err)
	}
	proxy := NewMultipleHostsReverseProxy(rb)
	log.Println("Starting httpserver at " + addr)
	log.Fatal(http.ListenAndServe(addr, proxy))
}

```

**内置几种负载均衡算法：**

hash算法 基于url和ip地址的hash

```
package load_balance

import (
	"fmt"
	"testing"
)

func TestNewConsistentHashBanlance(t *testing.T) {
	rb := NewConsistentHashBanlance(10, nil)
	rb.Add("127.0.0.1:2003") //0
	rb.Add("127.0.0.1:2004") //1
	rb.Add("127.0.0.1:2005") //2
	rb.Add("127.0.0.1:2006") //3
	rb.Add("127.0.0.1:2007") //4

	//url hash
	fmt.Println(rb.Get("http://127.0.0.1:2002/base/getinfo"))
	fmt.Println(rb.Get("http://127.0.0.1:2002/base/error"))
	fmt.Println(rb.Get("http://127.0.0.1:2002/base/getinfo"))
	fmt.Println(rb.Get("http://127.0.0.1:2002/base/changepwd"))

	//ip hash
	fmt.Println(rb.Get("127.0.0.1"))
	fmt.Println(rb.Get("192.168.0.1"))
	fmt.Println(rb.Get("127.0.0.1"))
}

```

随机算法


```
package load_balance

import (
	"fmt"
	"testing"
)

func TestRandomBalance(t *testing.T) {
	rb := &RandomBalance{}
	rb.Add("127.0.0.1:2003") //0
	rb.Add("127.0.0.1:2004") //1
	rb.Add("127.0.0.1:2005") //2
	rb.Add("127.0.0.1:2006") //3
	rb.Add("127.0.0.1:2007") //4

	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
}


```

轮询算法：

```
package load_balance

import (
	"fmt"
	"testing"
)

func Test_main(t *testing.T) {
	rb := &RoundRobinBalance{}
	rb.Add("127.0.0.1:2003") //0
	rb.Add("127.0.0.1:2004") //1
	rb.Add("127.0.0.1:2005") //2
	rb.Add("127.0.0.1:2006") //3
	rb.Add("127.0.0.1:2007") //4

	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
}




```


加权轮询算法：


```
package load_balance

import (
	"fmt"
	"testing"
)

func TestLB(t *testing.T) {
	rb := &WeightRoundRobinBalance{}
	rb.Add("127.0.0.1:2003", "4") //0
	rb.Add("127.0.0.1:2004", "3") //1
	rb.Add("127.0.0.1:2005", "2") //2

	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
	fmt.Println(rb.Next())
}


```













