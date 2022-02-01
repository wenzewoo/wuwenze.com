+++
title = "开放Spring Cloud Eureka API"
date = "2018-05-26 07:35:00"
url = "archives/434"
tags = ["SpringCloud","Eureka"]
categories = ["后端"]
+++

一般来说，Eureka 默认提供了一套 UI 界面，但在大多数情况下，由于 UI 风格问题并不适合直接嵌入到业务系统中使用；

本文通过扩展 Eureka 项目，实现相关的自定义接口，以便业务系统集成调用；

##### 1. Eureka Project; #####

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@EnableEurekaServer
@SpringBootApplication(scanBasePackages = "com.fastjee")
public class FastjeeEurekaApplication {

    public static void main(String[] args) {
        FastjeeApplication.run(FastjeeEurekaApplication.class, args);
    }
}
```

##### 2. RESTful API #####

```java
package com.fastjee.eureka.rest;

import com.google.common.collect.Lists;
import com.google.common.collect.Maps;

import com.netflix.appinfo.AmazonInfo;
import com.netflix.appinfo.ApplicationInfoManager;
import com.netflix.appinfo.DataCenterInfo;
import com.netflix.appinfo.InstanceInfo;
import com.netflix.config.ConfigurationManager;
import com.netflix.config.DeploymentContext;
import com.netflix.discovery.shared.Application;
import com.netflix.discovery.shared.Pair;
import com.netflix.eureka.EurekaServerContext;
import com.netflix.eureka.EurekaServerContextHolder;
import com.netflix.eureka.cluster.PeerEurekaNode;
import com.netflix.eureka.registry.PeerAwareInstanceRegistry;
import com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl;
import com.netflix.eureka.resources.StatusResource;
import com.netflix.eureka.util.StatusInfo;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.net.URI;
import java.util.ArrayList;
import java.util.Date;
import java.util.HashMap;
import java.util.Iterator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import javax.annotation.Resource;

import cn.hutool.core.util.ReflectUtil;

/**
 * Eureka RestAPI.
 */
@RestController
@RequestMapping("/eureka_rest_api")
public class EurekaRestApi {

    @Resource
    private ApplicationInfoManager applicationInfoManager;

    /**
     * Simple System Status, DS Replicas
     */
    @GetMapping("/system_status")
    public ResponseEntity<?> systemStatus() {
        Map<String, Object> model = Maps.newHashMap();
        DeploymentContext deploymentContext = ConfigurationManager.getDeploymentContext();
        model.put("currentTime", new Date());
        model.put("upTime", StatusInfo.getUpTime());
        model.put("applicationId", deploymentContext.getApplicationId());
        model.put("environment", deploymentContext.getDeploymentEnvironment());
        model.put("datacenter", deploymentContext.getDeploymentDatacenter());
        PeerAwareInstanceRegistry registry = getRegistry();
        // model.put("registry", registry);
        model.put("leaseExpirationEnabled", registry.isLeaseExpirationEnabled());
        model.put("renewsPerMinThreshold", registry.getNumOfRenewsPerMinThreshold());
        model.put("renewsInLastMin", registry.getNumOfRenewsInLastMin());
        // model.put("isBelowRenewThresold", registry.isBelowRenewThresold() == 1);
        String warningMessage = "";
        if (registry.isBelowRenewThresold() == 1) {
            if (!registry.isSelfPreservationModeEnabled()) {
                warningMessage = "注意: 续订量低于阈值, 自我保护模式被改变。";
            } else {
                warningMessage = "注意: 已经进入自我保护模式, 该模式下将不会注销任何微服务(即使服务已经失效)。";
            }
        } else if (!registry.isSelfPreservationModeEnabled()) {
            warningMessage = "注意: 自我保护模式已关闭。";
        }
        model.put("warningMessage", warningMessage);
        DataCenterInfo info = applicationInfoManager.getInfo().getDataCenterInfo();
        if (info.getName() == DataCenterInfo.Name.Amazon) {
            AmazonInfo amazonInfo = (AmazonInfo) info;
            model.put("amazonInfo", amazonInfo);
            model.put("amiId", amazonInfo.get(AmazonInfo.MetaDataKey.amiId));
            model.put("availabilityZone", amazonInfo.get(AmazonInfo.MetaDataKey.availabilityZone));
            model.put("instanceId", amazonInfo.get(AmazonInfo.MetaDataKey.instanceId));
        }
        Map<String, String> replicas = Maps.newHashMap();
        List<PeerEurekaNode> list = getServerContext().getPeerEurekaNodes().getPeerNodesView();
        for (PeerEurekaNode node : list) {
            try {
                URI uri = new URI(node.getServiceUrl());
                String href = scrubBasicAuth(node.getServiceUrl());
                replicas.put(uri.getHost(), href);
            } catch (Exception ex) {
                // ignore?
            }
        }
        model.put("replicas", replicas.entrySet());
        return ResponseEntity.ok(model);
    }

    /**
     * General Info, Instance Info
     */
    @GetMapping("/info")
    public ResponseEntity<?> info() {
        Map<String, Object> model = Maps.newHashMap();
        StatusInfo statusInfo = null;
        try {
            statusInfo = new StatusResource().getStatusInfo();
            statusInfo.isHealthy(); // throw NullPointerException
        } catch (Exception e) {
            if (e instanceof NullPointerException) {
                ReflectUtil.setFieldValue(statusInfo, "isHeathly", true);
            } else {
                statusInfo = StatusInfo.Builder.newBuilder().isHealthy(false).build();
            }
        }
        // Instance Info
        model.put("instanceInfo", getInstanceInfo(statusInfo));

        // General Info
        model.put("generalInfo", getGeneralInfo(statusInfo));
        return ResponseEntity.ok(model);
    }

    /**
     * Instances currently registered with Eureka
     */
    @GetMapping("/registered_apps")
    public ResponseEntity<?> registeredApps() {
        List<Map<String, Object>> registeredApps = Lists.newArrayList();
        List<Application> sortedApplications = getRegistry().getSortedApplications();
        for (Application app : sortedApplications) {
            LinkedHashMap<String, Object> appData = new LinkedHashMap<>();
            registeredApps.add(appData);
            appData.put("name", app.getName());
            Map<String, Integer> amiCounts = new HashMap<>();
            Map<InstanceInfo.InstanceStatus, List<Pair<String, String>>> instancesByStatus = new HashMap<>();
            Map<String, Integer> zoneCounts = new HashMap<>();
            for (InstanceInfo info : app.getInstances()) {
                String id = info.getId();
                String url = info.getStatusPageUrl();
                InstanceInfo.InstanceStatus status = info.getStatus();
                String ami = "n/a";
                String zone = "";
                if (info.getDataCenterInfo().getName() == DataCenterInfo.Name.Amazon) {
                    AmazonInfo dcInfo = (AmazonInfo) info.getDataCenterInfo();
                    ami = dcInfo.get(AmazonInfo.MetaDataKey.amiId);
                    zone = dcInfo.get(AmazonInfo.MetaDataKey.availabilityZone);
                }
                Integer count = amiCounts.get(ami);
                if (count != null) {
                    amiCounts.put(ami, count + 1);
                } else {
                    amiCounts.put(ami, 1);
                }
                count = zoneCounts.get(zone);
                if (count != null) {
                    zoneCounts.put(zone, count + 1);
                } else {
                    zoneCounts.put(zone, 1);
                }
                List<Pair<String, String>> list = instancesByStatus.get(status);
                if (list == null) {
                    list = new ArrayList<>();
                    instancesByStatus.put(status, list);
                }
                list.add(new Pair<>(id, url));
            }
            appData.put("amiCounts", amiCounts.entrySet());
            appData.put("zoneCounts", zoneCounts.entrySet());
            ArrayList<Map<String, Object>> instanceInfos = new ArrayList<>();
            appData.put("instanceInfos", instanceInfos);
            for (Iterator<Map.Entry<InstanceInfo.InstanceStatus, List<Pair<String, String>>>> iter = instancesByStatus.entrySet().iterator(); iter.hasNext(); ) {
                Map.Entry<InstanceInfo.InstanceStatus, List<Pair<String, String>>> entry = iter.next();
                List<Pair<String, String>> value = entry.getValue();
                InstanceInfo.InstanceStatus status = entry.getKey();
                LinkedHashMap<String, Object> instanceData = new LinkedHashMap<>();
                instanceInfos.add(instanceData);
                instanceData.put("status", entry.getKey());
                ArrayList<Map<String, Object>> instances = new ArrayList<>();
                instanceData.put("instances", instances);
                instanceData.put("isNotUp", status != InstanceInfo.InstanceStatus.UP);
                for (Pair<String, String> p : value) {
                    LinkedHashMap<String, Object> instance = new LinkedHashMap<>();
                    instances.add(instance);
                    instance.put("id", p.first());
                    String url = p.second();
                    instance.put("url", url);
                    boolean isHref = url != null && url.startsWith("http");
                    instance.put("isHref", isHref);
                }
            }
        }
        return ResponseEntity.ok(registeredApps);
    }

    /**
     * Last 1000 cancelled leases, Last 1000 newly registered leases
     */
    @GetMapping("/lastn_leases")
    public ResponseEntity<?> lastn() {
        Map<String, Object> lastN = Maps.newHashMap();
        PeerAwareInstanceRegistryImpl registry = (PeerAwareInstanceRegistryImpl) getRegistry();
        ArrayList<Map<String, Object>> lastNCanceled = new ArrayList<>();
        List<Pair<Long, String>> list = registry.getLastNCanceledInstances();
        for (Pair<Long, String> entry : list) {
            lastNCanceled.add(registeredInstance(entry.second(), entry.first()));
        }
        lastN.put("lastNCanceled", lastNCanceled);
        list = registry.getLastNRegisteredInstances();
        ArrayList<Map<String, Object>> lastNRegistered = new ArrayList<>();
        for (Pair<Long, String> entry : list) {
            lastNRegistered.add(registeredInstance(entry.second(), entry.first()));
        }
        lastN.put("lastNRegistered", lastNRegistered);
        return ResponseEntity.ok(lastN);
    }

    private Map<String, String> getGeneralInfo(StatusInfo statusInfo) {
        Map<String, String> generalInfoMap = Maps.newHashMap(statusInfo.getGeneralStats());
        generalInfoMap.putAll(statusInfo.getApplicationStats());
        if (generalInfoMap.get("registered-replicas").contains("@")) {
            generalInfoMap.put("registered-replicas", scrubBasicAuth(generalInfoMap.get("registered-replicas")));
        }
        if (generalInfoMap.get("unavailable-replicas").contains("@")) {
            generalInfoMap.put("unavailable-replicas", scrubBasicAuth(generalInfoMap.get("unavailable-replicas")));
        }
        if (generalInfoMap.get("available-replicas").contains("@")) {
            generalInfoMap.put("available-replicas", scrubBasicAuth(generalInfoMap.get("available-replicas")));
        }
        return generalInfoMap;
    }

    private Map<String, String> getInstanceInfo(StatusInfo statusInfo) {
        InstanceInfo instanceInfo = statusInfo.getInstanceInfo();
        Map<String, String> instanceMap = new HashMap<>();
        instanceMap.put("ipAddr", instanceInfo.getIPAddr());
        instanceMap.put("status", instanceInfo.getStatus().toString());
        if (instanceInfo.getDataCenterInfo().getName() == DataCenterInfo.Name.Amazon) {
            AmazonInfo info = (AmazonInfo) instanceInfo.getDataCenterInfo();
            instanceMap.put("availability-zone",info.get(AmazonInfo.MetaDataKey.availabilityZone));
            instanceMap.put("public-ipv4", info.get(AmazonInfo.MetaDataKey.publicIpv4));
            instanceMap.put("instance-id", info.get(AmazonInfo.MetaDataKey.instanceId));
            instanceMap.put("public-hostname",info.get(AmazonInfo.MetaDataKey.publicHostname));
            instanceMap.put("ami-id", info.get(AmazonInfo.MetaDataKey.amiId));
            instanceMap.put("instance-type",info.get(AmazonInfo.MetaDataKey.instanceType));
        }
        return instanceMap;
    }

    private Map<String, Object> registeredInstance(String lease, long date) {
        Map<String, Object> map = Maps.newHashMap();
        map.put("timestamp", new Date(date));
        map.put("lease", lease);
        return map;
    }

    private PeerAwareInstanceRegistry getRegistry() {
        return getServerContext().getRegistry();
    }

    private EurekaServerContext getServerContext() {
        return EurekaServerContextHolder.getInstance().getServerContext();
    }

    private String scrubBasicAuth(String urlList) {
        String[] urls = urlList.split(",");
        StringBuilder filteredUrls = new StringBuilder();
        for (String u : urls) {
            if (u.contains("@")) {
                filteredUrls.append(u.substring(0, u.indexOf("//") + 2)).append(u.substring(u.indexOf("@") + 1, u.length())).append(",");
            } else {
                filteredUrls.append(u).append(",");
            }
        }
        return filteredUrls.substring(0, filteredUrls.length() - 1);
    }
}
```