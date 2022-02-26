## 4.客户端获取所有实例com.alibaba.nacos.client.naming.NacosNamingService#getAllInstances(java.lang.String, java.lang.String, java.util.List<java.lang.String>, boolean)
```java
public List<Instance> getAllInstances(String serviceName, String groupName, List<String> clusters,
        boolean subscribe) throws NacosException {
    
    ServiceInfo serviceInfo;
    if (subscribe) {
        serviceInfo = hostReactor.getServiceInfo(NamingUtils.getGroupedName(serviceName, groupName),
                StringUtils.join(clusters, ","));
    } else {
        serviceInfo = hostReactor
                .getServiceInfoDirectlyFromServer(NamingUtils.getGroupedName(serviceName, groupName),
                        StringUtils.join(clusters, ","));
    }
    List<Instance> list;
    if (serviceInfo == null || CollectionUtils.isEmpty(list = serviceInfo.getHosts())) {
        return new ArrayList<Instance>();
    }
    return list;
}
```
### 4.1com.alibaba.nacos.client.naming.core.HostReactor#getServiceInfo
```java
public ServiceInfo getServiceInfo(final String serviceName, final String clusters) {
        
    NAMING_LOGGER.debug("failover-mode: " + failoverReactor.isFailoverSwitch());
    String key = ServiceInfo.getKey(serviceName, clusters);
    if (failoverReactor.isFailoverSwitch()) {
        return failoverReactor.getService(key);
    }
    // 从serviceInfoMap中获取，即缓存
    ServiceInfo serviceObj = getServiceInfo0(serviceName, clusters);
    
    if (null == serviceObj) {
        serviceObj = new ServiceInfo(serviceName, clusters);
        
        serviceInfoMap.put(serviceObj.getKey(), serviceObj);
        
        updatingMap.put(serviceName, new Object());
        updateServiceNow(serviceName, clusters);
        updatingMap.remove(serviceName);
        
    } else if (updatingMap.containsKey(serviceName)) {
        
        if (UPDATE_HOLD_INTERVAL > 0) {
            // hold a moment waiting for update finish
            synchronized (serviceObj) {
                try {
                    serviceObj.wait(UPDATE_HOLD_INTERVAL);
                } catch (InterruptedException e) {
                    NAMING_LOGGER
                            .error("[getServiceInfo] serviceName:" + serviceName + ", clusters:" + clusters, e);
                }
            }
        }
    }
    // 启用定时任务
    scheduleUpdateIfAbsent(serviceName, clusters);
    
    return serviceInfoMap.get(serviceObj.getKey());
}
```
#### 4.1.1com.alibaba.nacos.client.naming.core.HostReactor#updateServiceNow
```java
 private void updateServiceNow(String serviceName, String clusters) {
    try {
        updateService(serviceName, clusters);
    } catch (NacosException e) {
        NAMING_LOGGER.error("[NA] failed to update serviceName: " + serviceName, e);
    }
}
```
- com.alibaba.nacos.client.naming.core.HostReactor#updateService
```java
public void updateService(String serviceName, String clusters) throws NacosException {
    ServiceInfo oldService = getServiceInfo0(serviceName, clusters);
    try {
        
        String result = serverProxy.queryList(serviceName, clusters, pushReceiver.getUdpPort(), false);
        
        if (StringUtils.isNotEmpty(result)) {
            processServiceJson(result);
        }
    } finally {
        if (oldService != null) {
            synchronized (oldService) {
                oldService.notifyAll();
            }
        }
    }
}
```
- com.alibaba.nacos.client.naming.net.NamingProxy#queryList
```java
public String queryList(String serviceName, String clusters, int udpPort, boolean healthyOnly)
            throws NacosException {
        
    final Map<String, String> params = new HashMap<String, String>(8);
    params.put(CommonParams.NAMESPACE_ID, namespaceId);
    params.put(CommonParams.SERVICE_NAME, serviceName);
    params.put("clusters", clusters);
    params.put("udpPort", String.valueOf(udpPort));
    params.put("clientIP", NetUtils.localIP());
    params.put("healthyOnly", String.valueOf(healthyOnly));
    
    return reqApi(UtilAndComs.nacosUrlBase + "/instance/list", params, HttpMethod.GET);
}
```