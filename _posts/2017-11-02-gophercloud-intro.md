---
layout: post
title:  "gophercloud简介--OpenStack golang SDK"
lang: zh
date: 2017-11-2
---

### gophercloud简介

gophercloud是OpenStack的Golang SDK包，第三方应用程序可以通过这个包提供的API接口调用到OpenStack云相关的API服务。


### gophercloud的流程

既然是通过API调用服务，那就和使用OpenStack API的程序处于同一位置，大致可以分为如下几个步骤。

1. 提供鉴权的地址，用户名&密码（或者是token），domain信息
2. 提供Cert 文件，Key文件
2. 通过鉴权地址建立安全连接后，发送1中的信息进行鉴权。
3. 鉴权成功后获得token和其它服务的地址（服务目录）
4. 通过服务的地址和token访问其它的服务。

这部分的核心代码参见[这里](https://github.com/freesky-edward/gophercloud/blob/master/openstack/client.go#L101)

### gophercloud具体实现——两个client + 1个factory

既然是要通过code调用API服务，那么肯定需要封装能访问这些API服务的client，在gophercloud中一共有两个client，分别代表了两个层次的client（严格上说是3个，这里有没有把http client这一层算作是gophercloud的client）。这两个client分别是[service-client](https://github.com/freesky-edward/gophercloud/blob/master/service_client.go)和[provider-client](https://github.com/freesky-edward/gophercloud/blob/master/provider_client.go)。大致关系如下：

![]({{ site.url }}/images/2017-11-02-gophercloud-intro/clients_relationship.png)

其中provider-client是service-client的一个generic implement，它是所有服务访问的基础client，provider-client主要是封装了http-client，构建http消息通过http-client发送，然后再处理返回消息以及异常。它的结构体声明如下：

```golang
// ProviderClient stores details that are required to interact with any
// services within a specific provider's API.
//
// Generally, you acquire a ProviderClient by calling the NewClient method in
// the appropriate provider's child package, providing whatever authentication
// credentials are required.
type ProviderClient struct {
	// IdentityBase is the base URL used for a particular provider's identity
	// service - it will be used when issuing authenticatation requests. It
	// should point to the root resource of the identity service, not a specific
	// identity version.
	IdentityBase string

	// IdentityEndpoint is the identity endpoint. This may be a specific version
	// of the identity service. If this is the case, this endpoint is used rather
	// than querying versions first.
	IdentityEndpoint string

	// TokenID is the ID of the most recently issued valid token.
	TokenID string

	// EndpointLocator describes how this provider discovers the endpoints for
	// its constituent services.
	EndpointLocator EndpointLocator

	// HTTPClient allows users to interject arbitrary http, https, or other transit behaviors.
	HTTPClient http.Client

	// UserAgent represents the User-Agent header in the HTTP request.
	UserAgent UserAgent

	// ReauthFunc is the function used to re-authenticate the user if the request
	// fails with a 401 HTTP response code. This a needed because there may be multiple
	// authentication functions for different Identity service versions.
	ReauthFunc func() error

	Debug bool
}
```

它的核心成员是HTTPClient，核心方法是[Request](https://github.com/freesky-edward/gophercloud/blob/master/provider_client.go#L108)方法，申明如下：

```golang
// Request performs an HTTP request using the ProviderClient's current HTTPClient. An authentication
// header will automatically be provided.
func (client *ProviderClient) Request(method, url string, options *RequestOpts) (*http.Response, error) 
```

基于provider-client，会衍生出各种服务client，如：计算服务client，存储服务client，鉴权服务client等等，这些服务clients并不是每一种服务定义了一个结构体，而是抽象成了一个结构体申明，那就是service-client.
这里先介绍下service-client的内部，至于如果构建这些不同类型的服务client的，稍后介绍。

service-client继承至provider-client，但是对provider-client进行了具体化，把provider的request请求装饰成了rest请求类型：get， put， post， delete， patch，并对provider的属性针对各个服务API进行了进一步定义，具体结构定义如下：

```golang
// ServiceClient stores details required to interact with a specific service API implemented by a provider.
// Generally, you'll acquire these by calling the appropriate `New` method on a ProviderClient.
type ServiceClient struct {
	// ProviderClient is a reference to the provider that implements this service.
	*ProviderClient

	// Endpoint is the base URL of the service's API, acquired from a service catalog.
	// It MUST end with a /.
	Endpoint string

	// ResourceBase is the base URL shared by the resources within a service's API. It should include
	// the API version and, like Endpoint, MUST end with a / if set. If not set, the Endpoint is used
	// as-is, instead.
	ResourceBase string

	// This is the service client type (e.g. compute, sharev2).
	// NOTE: FOR INTERNAL USE ONLY. DO NOT SET. GOPHERCLOUD WILL SET THIS.
	// It is only exported because it gets set in a different package.
	Type string

	// The microversion of the service to use. Set this to use a particular microversion.
	Microversion string
}
```
这里的核心就是provider-client，所有的业务处理都转换成provider-client的request进行发送和接收。

回到刚才的话题，针对不同的服务会创建出不同的service-client，那么service-client这些属性如何设置？如果不做封装使用起来将会是噩梦，gophercloud当然不会让这个噩梦出现，于是提供相应的工厂方法——[client](https://github.com/freesky-edward/gophercloud/blob/master/openstack/client.go)，这里的client是一个提供构建各种服务实例（service-client实例）的工厂方法的集合，在这个client里可以找到openstack已有的所有服务client的构建方法，如：计算服务、网络服务、存储服务等等。

```golang
// NewComputeV2 creates a ServiceClient that may be used with the v2 compute
// package.
func NewComputeV2(client *gophercloud.ProviderClient, eo gophercloud.EndpointOpts) (*gophercloud.ServiceClient, error) {
	return initClientOpts(client, eo, "compute")
}

// NewNetworkV2 creates a ServiceClient that may be used with the v2 network
// package.
func NewNetworkV2(client *gophercloud.ProviderClient, eo gophercloud.EndpointOpts) (*gophercloud.ServiceClient, error) {
	sc, err := initClientOpts(client, eo, "network")
	sc.ResourceBase = sc.Endpoint + "v2.0/"
	return sc, err
}

// NewBlockStorageV1 creates a ServiceClient that may be used to access the v1
// block storage service.
func NewBlockStorageV1(client *gophercloud.ProviderClient, eo gophercloud.EndpointOpts) (*gophercloud.ServiceClient, error) {
	return initClientOpts(client, eo, "volume")
}
```

在构建这些service-client时需要转入provider-client和EndpointOpts（后面介绍这个参数），而client工厂也提供provider-client的构建，见下：

```golang
A basic example of using this would be:
	ao, err := openstack.AuthOptionsFromEnv()
	provider, err := openstack.NewClient(ao.IdentityEndpoint)
	client, err := openstack.NewIdentityV3(provider, gophercloud.EndpointOpts{})
*/
func NewClient(endpoint string) (*gophercloud.ProviderClient, error)
```

通过上述这些工厂构建出相应的service-client，利用service-client的get，post，put，delete等方法，构建出相应的参数，就可以调用OpenStack的API了，具体每个OpenStack API的参数，请参见[这里](https://docs.openstack.org/pike/api/)，gophercloud针对各个服务需要的参数也定义成了相应的结构体，分别在https://github.com/freesky-edward/gophercloud/tree/master/openstack 的子目录下。如块存储的创建需要的参数定义在[CreateOpts](https://github.com/freesky-edward/gophercloud/blob/master/openstack/blockstorage/v2/volumes/requests.go#L17)里，并且封装了相应的方法，简要定义如下，详细参见[这里](https://github.com/freesky-edward/gophercloud/blob/master/openstack/blockstorage/v2/volumes/requests.go#L49)

```golang
// Create will create a new Volume based on the values in CreateOpts. To extract
// the Volume object from the response, call the Extract method on the
// CreateResult.
func Create(client *gophercloud.ServiceClient, opts CreateOptsBuilder) (r CreateResult) {
	b, err := opts.ToVolumeCreateMap()
	if err != nil {
		r.Err = err
		return
	}
	_, r.Err = client.Post(createURL(client), b, &r.Body, &gophercloud.RequestOpts{
		OkCodes: []int{202},
	})
	return
}
```

这个方法的核心就是调用service-client的Post方法，只是由于rest的uri是/v3/{project_id}/volumes需要指定是volume操作，所以需要通过createURL(client)构建该URI.

```golang
func createURL(c *gophercloud.ServiceClient) string {
	return c.ServiceURL("volumes")
}
```

这样所有的对外接口就全呈现出来了，对于gophercloud的使用者来讲，首先通过client.NewClient构建一个provider-client，然后利用这个provider-client和EndpointOpts通过各个服务工厂方法（如块存储服务工厂client.NewBlockStorageV1）构建出service-client。最后使用这个service-client加上相应的调用参数就可以调用相应的服务业务接口了，如创建块存储func Create(client *gophercloud.ServiceClient, opts CreateOptsBuilder)。

前面在介绍创建相应的service-client时，留有一个问题——EndpointOpts是啥？

在介绍EndpointOpts之前，先说明一下，上述的主流程介绍中有一个细节需要再详细说明一下，在OpenStack里每个API服务都有自己endpoint，这个endpoint注册在keystone服务里，通过admin查看keystone的服务目录，大致结果如下图：

![]({{ site.url }}/images/2017-11-02-gophercloud-intro/service-catalog.png)

在构建service-client时需要用到这个endpoint地址，在每次向OpenStack API发送rest请求时，需要根据这个endpoint来构建rest的uri，使用构建的这个uri通过provider的request发送请求。

如果获得这个endpoint就需要上面提到的EndpointOpts，在构建完成provider-client后，通过provider-client可以进行鉴权，调用client工厂的[Authenticate](https://github.com/freesky-edward/gophercloud/blob/master/openstack/client.go#L99)方法。鉴权成功后，就可以获取到当前用户可以访问的API目录，以V3版本为例，详细代码见[这里](https://github.com/freesky-edward/gophercloud/blob/master/openstack/client.go#L178)

```golang
v3Client, err := NewIdentityV3(client, eo)
	if err != nil {
		return err
	}

	if endpoint != "" {
		v3Client.Endpoint = endpoint
	}

	result := tokens3.Create(v3Client, opts)

	token, err := result.ExtractToken()
	if err != nil {
		return err
	}

	catalog, err := result.ExtractServiceCatalog()
	if err != nil {
		return err
	}

	client.TokenID = token.ID

	if opts.CanReauth() {
		client.ReauthFunc = func() error {
			client.TokenID = ""
			return v3auth(client, endpoint, opts, eo)
		}
	}
	client.EndpointLocator = func(opts gophercloud.EndpointOpts) (string, error) {
		return V3EndpointURL(catalog, opts)
	}
```

catalog, err := result.ExtractServiceCatalog()就是解析出相应的服务目录，获得服务目录后需要通过EndpointOpts来定义的参数进行过滤获得具体的服务endpoint，这里主要是根据region，服务提供的范围（上图中的Interface），详细代码见[这里](https://github.com/freesky-edward/gophercloud/blob/master/openstack/endpoint_location.go#L60):

```golang
for _, entry := range catalog.Entries {
		if (entry.Type == opts.Type) && (opts.Name == "" || entry.Name == opts.Name) {
			for _, endpoint := range entry.Endpoints {
				if opts.Availability != gophercloud.AvailabilityAdmin &&
					opts.Availability != gophercloud.AvailabilityPublic &&
					opts.Availability != gophercloud.AvailabilityInternal {
					err := &ErrInvalidAvailabilityProvided{}
					err.Argument = "Availability"
					err.Value = opts.Availability
					return "", err
				}
				if (opts.Availability == gophercloud.Availability(endpoint.Interface)) &&
					(opts.Region == "" || endpoint.Region == opts.Region) {
					endpoints = append(endpoints, endpoint)
				}
			}
		}
	}
```

而这个方法是在鉴权后保存在provider-client的EndpointLocator里，在client的工厂创建各个service-client时会根据这个方法得到endpoint，并将其注入到service-client的Endpoint里，代码请参见[这里](https://github.com/freesky-edward/gophercloud/blob/master/openstack/client.go#L265):

```golang
func initClientOpts(client *gophercloud.ProviderClient, eo gophercloud.EndpointOpts, clientType string) (*gophercloud.ServiceClient, error) {
    sc := new(gophercloud.ServiceClient)
	eo.ApplyDefaults(clientType)
	url, err := client.EndpointLocator(eo)
	if err != nil {
		return sc, err
	}
	sc.ProviderClient = client
	sc.Endpoint = url
	sc.Type = clientType
	return sc, nil
}
```

所以EndpointOpts就是需要指定这个client是用于哪个region调用内部或者外部的API服务的一个结构体。

### 总结

对于gophercloud来讲，看似代码量很大，但是只要我们理解了它的核心工作职责，抓住了它的骨架——两个client+1个工厂后，剩下的逻辑就非常简单了，都是针对不同服务去定义相应的参数和返回处理，以及请求URL的拼装。
