# 智能购物清单功能 - 代码骨架

## 文档信息
- **版本号**：v1.0
- **创建日期**：2026-05-04
- **创建者**：Coder Agent
- **技术栈**：Java 11 + Spring Boot 2.7

## 1. Controller 层

### SmartListController.java

```java
package com.hema.smartlist.controller;

import com.hema.smartlist.dto.*;
import com.hema.smartlist.service.SmartListService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

/**
 * 智能购物清单 Controller
 * 
 * 职责：
 * - 接收前端请求
 * - 参数校验
 * - 调用 Service 层处理业务逻辑
 * - 返回响应结果
 */
@RestController
@RequestMapping("/api/v1/smart-list")
public class SmartListController {

    @Autowired
    private SmartListService smartListService;

    /**
     * 生成智能购物清单
     * 
     * 核心逻辑：
     * 1. 校验用户输入
     * 2. 调用 LLM 服务解析场景
     * 3. 调用推荐引擎生成商品清单
     * 4. 查询商品实时库存和价格
     * 5. 按类别分组返回
     */
    @PostMapping("/generate")
    public Result<SmartListResponse> generateList(@RequestBody SmartListRequest request) {
        // 参数校验
        if (request.getUserId() == null || request.getSceneDescription() == null) {
            return Result.error("参数不完整");
        }
        
        // 调用 Service 层生成清单
        SmartListResponse response = smartListService.generateList(request);
        
        return Result.success(response);
    }

    /**
     * 保存清单模板
     * 
     * 核心逻辑：
     * 1. 校验清单 ID 是否存在
     * 2. 保存模板信息
     * 3. 返回模板 ID
     */
    @PostMapping("/template/save")
    public Result<TemplateResponse> saveTemplate(@RequestBody TemplateRequest request) {
        TemplateResponse response = smartListService.saveTemplate(request);
        return Result.success(response);
    }

    /**
     * 一键加购
     * 
     * 核心逻辑：
     * 1. 查询清单详情
     * 2. 批量查询商品库存
     * 3. 过滤不可用商品
     * 4. 调用购物车服务批量添加
     * 5. 返回加购结果
     */
    @PostMapping("/add-to-cart")
    public Result<AddToCartResponse> addToCart(@RequestBody AddToCartRequest request) {
        AddToCartResponse response = smartListService.addToCart(request);
        return Result.success(response);
    }
}
```

## 2. Service 层

### SmartListService.java（接口）

```java
package com.hema.smartlist.service;

import com.hema.smartlist.dto.*;

/**
 * 智能购物清单 Service 接口
 */
public interface SmartListService {

    /**
     * 生成智能购物清单
     * 
     * @param request 请求参数（用户ID、场景描述、人数、过敏原等）
     * @return 智能清单响应（清单ID、商品列表、预估金额等）
     */
    SmartListResponse generateList(SmartListRequest request);

    /**
     * 保存清单模板
     * 
     * @param request 请求参数（清单ID、模板名称、是否公开等）
     * @return 模板响应（模板ID、创建时间等）
     */
    TemplateResponse saveTemplate(TemplateRequest request);

    /**
     * 一键加购
     * 
     * @param request 请求参数（清单ID、商品列表等）
     * @return 加购响应（成功数量、失败数量、失败原因等）
     */
    AddToCartResponse addToCart(AddToCartRequest request);
}
```

### SmartListServiceImpl.java（实现类）

```java
package com.hema.smartlist.service.impl;

import com.hema.smartlist.dto.*;
import com.hema.smartlist.service.SmartListService;
import com.hema.smartlist.service.LLMService;
import com.hema.smartlist.service.RecommendationEngine;
import com.hema.smartlist.service.ProductService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.util.StringUtils;
import java.util.*;
import java.util.stream.Collectors;

/**
 * 智能购物清单 Service 实现类
 * 
 * 核心逻辑：
 * - 协调各个服务完成业务流程
 * - 处理缓存逻辑
 * - 处理异常和降级
 */
@Service
public class SmartListServiceImpl implements SmartListService {

    @Autowired
    private LLMService llmService;

    @Autowired
    private RecommendationEngine recommendationEngine;

    @Autowired
    private ProductService productService;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    private static final String CACHE_KEY_PREFIX = "smart:list:";
    private static final String RECOMMEND_CACHE_KEY_PREFIX = "smart:recommend:";
    private static final long CACHE_EXPIRE_SECONDS = 86400; // 24小时

    @Override
    public SmartListResponse generateList(SmartListRequest request) {
        // 1. 参数校验
        if (request.getUserId() == null || !StringUtils.hasText(request.getSceneDescription())) {
            throw new IllegalArgumentException("用户ID和场景描述不能为空");
        }

        // 2. 检查 Redis 缓存
        String cacheKey = buildCacheKey(request);
        SmartListResponse cachedResponse = (SmartListResponse) redisTemplate.opsForValue().get(cacheKey);
        if (cachedResponse != null) {
            return cachedResponse;
        }

        // 3. 调用 LLM 服务解析场景描述
        SceneParseResult parseResult;
        try {
            parseResult = llmService.parseScene(request.getSceneDescription());
            // 如果 LLM 未解析出人数，使用请求中的默认值
            if (parseResult.getPeopleCount() == null) {
                parseResult.setPeopleCount(request.getPeopleCount() != null ? request.getPeopleCount() : 4);
            }
        } catch (Exception e) {
            // LLM 服务降级：使用规则引擎
            parseResult = fallbackParseScene(request.getSceneDescription(), request.getPeopleCount());
        }

        // 4. 调用推荐引擎生成商品清单
        List<RecommendationItem> recommendations;
        try {
            recommendations = recommendationEngine.generateRecommendations(
                parseResult, 
                request.getUserId(), 
                request.getAllergens()
            );
        } catch (Exception e) {
            throw new RuntimeException("推荐引擎异常，请稍后重试", e);
        }

        if (recommendations == null || recommendations.isEmpty()) {
            throw new RuntimeException("未找到合适的推荐商品");
        }

        // 5. 实时查询商品库存和价格
        List<String> productIds = recommendations.stream()
            .map(RecommendationItem::getProductId)
            .collect(Collectors.toList());
        
        Map<String, ProductInfo> productInfoMap = productService.batchQueryProducts(productIds);

        // 6. 构建响应数据
        SmartListResponse response = new SmartListResponse();
        response.setListId(generateListId());
        response.setSceneDescription(request.getSceneDescription());
        response.setPeopleCount(parseResult.getPeopleCount());
        response.setTotalItems(recommendations.size());

        // 7. 按类别分组商品
        Map<String, List<SmartListResponse.ProductItem>> categoryMap = new LinkedHashMap<>();
        double totalAmount = 0.0;

        for (RecommendationItem item : recommendations) {
            ProductInfo productInfo = productInfoMap.get(item.getProductId());
            if (productInfo == null || !productInfo.isAvailable()) {
                continue; // 跳过不可用商品
            }

            SmartListResponse.ProductItem productItem = new SmartListResponse.ProductItem();
            productItem.setProductId(item.getProductId());
            productItem.setProductName(item.getProductName());
            productItem.setSpecification(productInfo.getSpecification());
            productItem.setPrice(productInfo.getPrice());
            productItem.setQuantity(item.getQuantity());
            productItem.setImageUrl(productInfo.getImageUrl());
            productItem.setStock(productInfo.getStock());
            productItem.setReason(item.getReason());

            totalAmount += productInfo.getPrice() * item.getQuantity();

            // 按类别分组
            String categoryName = item.getCategoryName();
            categoryMap.computeIfAbsent(categoryName, k -> new ArrayList<>()).add(productItem);
        }

        // 转换为分类列表
        List<SmartListResponse.CategoryItem> categories = categoryMap.entrySet().stream()
            .map(entry -> {
                SmartListResponse.CategoryItem categoryItem = new SmartListResponse.CategoryItem();
                categoryItem.setCategoryName(entry.getKey());
                categoryItem.setItems(entry.getValue());
                return categoryItem;
            })
            .collect(Collectors.toList());

        response.setCategories(categories);
        response.setEstimatedAmount(totalAmount);

        // 8. 缓存结果到 Redis
        redisTemplate.opsForValue().set(cacheKey, response, CACHE_EXPIRE_SECONDS, java.util.concurrent.TimeUnit.SECONDS);

        return response;
    }

    @Override
    public TemplateResponse saveTemplate(TemplateRequest request) {
        // 1. 参数校验
        if (request.getUserId() == null || !StringUtils.hasText(request.getListId())) {
            throw new IllegalArgumentException("用户ID和清单ID不能为空");
        }
        if (!StringUtils.hasText(request.getTemplateName())) {
            throw new IllegalArgumentException("模板名称不能为空");
        }

        // 2. 查询清单详情
        SmartListResponse listResponse = (SmartListResponse) redisTemplate.opsForValue()
            .get(CACHE_KEY_PREFIX + request.getListId());
        
        if (listResponse == null) {
            throw new IllegalArgumentException("清单不存在或已过期");
        }

        // 3. 生成模板ID并保存到数据库
        String templateId = generateTemplateId();
        
        // 这里应该调用 Mapper 层保存到数据库
        // templateMapper.insert(buildTemplateEntity(request, templateId, listResponse));

        // 4. 构建响应
        TemplateResponse response = new TemplateResponse();
        response.setTemplateId(templateId);
        response.setTemplateName(request.getTemplateName());
        response.setCreateTime(new Date());

        return response;
    }

    @Override
    public AddToCartResponse addToCart(AddToCartRequest request) {
        // 1. 参数校验
        if (request.getUserId() == null || !StringUtils.hasText(request.getListId())) {
            throw new IllegalArgumentException("用户ID和清单ID不能为空");
        }

        // 2. 查询清单详情
        SmartListResponse listResponse = (SmartListResponse) redisTemplate.opsForValue()
            .get(CACHE_KEY_PREFIX + request.getListId());
        
        if (listResponse == null) {
            throw new IllegalArgumentException("清单不存在或已过期");
        }

        // 3. 收集所有商品ID
        List<String> productIds = new ArrayList<>();
        for (SmartListResponse.CategoryItem category : listResponse.getCategories()) {
            for (SmartListResponse.ProductItem item : category.getItems()) {
                productIds.add(item.getProductId());
            }
        }

        // 4. 批量查询商品库存
        Map<String, ProductInfo> productInfoMap = productService.batchQueryProducts(productIds);

        // 5. 过滤不可用商品并构建购物车商品列表
        List<CartItem> validItems = new ArrayList<>();
        List<AddToCartResponse.FailedItem> failedItems = new ArrayList<>();

        for (SmartListResponse.CategoryItem category : listResponse.getCategories()) {
            for (SmartListResponse.ProductItem item : category.getItems()) {
                ProductInfo productInfo = productInfoMap.get(item.getProductId());
                
                if (productInfo == null) {
                    // 商品不存在
                    failedItems.add(buildFailedItem(item.getProductId(), "商品不存在"));
                    continue;
                }
                
                if (!productInfo.isAvailable()) {
                    // 商品已下架
                    failedItems.add(buildFailedItem(item.getProductId(), "商品已下架"));
                    continue;
                }
                
                if (productInfo.getStock() < item.getQuantity()) {
                    // 库存不足
                    failedItems.add(buildFailedItem(item.getProductId(), "库存不足"));
                    continue;
                }

                // 有效商品
                CartItem cartItem = new CartItem();
                cartItem.setProductId(item.getProductId());
                cartItem.setQuantity(item.getQuantity());
                validItems.add(cartItem);
            }
        }

        // 6. 调用购物车服务批量添加
        String cartId;
        try {
            cartId = productService.batchAddToCart(request.getUserId(), validItems);
        } catch (Exception e) {
            throw new RuntimeException("加购失败，请稍后重试", e);
        }

        // 7. 构建响应
        AddToCartResponse response = new AddToCartResponse();
        response.setCartId(cartId);
        response.setSuccessCount(validItems.size());
        response.setFailedCount(failedItems.size());
        response.setFailedItems(failedItems);

        return response;
    }

    // ========== 辅助方法 ==========

    /**
     * 构建缓存键
     */
    private String buildCacheKey(SmartListRequest request) {
        String key = request.getUserId() + ":" + request.getSceneDescription();
        if (request.getPeopleCount() != null) {
            key += ":" + request.getPeopleCount();
        }
        return CACHE_KEY_PREFIX + key.hashCode();
    }

    /**
     * 生成清单ID
     */
    private String generateListId() {
        return "SL" + System.currentTimeMillis() + UUID.randomUUID().toString().substring(0, 8);
    }

    /**
     * 生成模板ID
     */
    private String generateTemplateId() {
        return "T" + System.currentTimeMillis() + UUID.randomUUID().toString().substring(0, 8);
    }

    /**
     * LLM 服务降级：使用规则引擎解析场景
     */
    private SceneParseResult fallbackParseScene(String sceneDescription, Integer peopleCount) {
        SceneParseResult result = new SceneParseResult();
        result.setSceneType("通用聚会");
        result.setPeopleCount(peopleCount != null ? peopleCount : 4);
        result.setOriginalDescription(sceneDescription);
        result.setSpecialRequirements(new ArrayList<>());
        result.setConfidence(0.5);
        return result;
    }

    /**
     * 构建失败商品项
     */
    private AddToCartResponse.FailedItem buildFailedItem(String productId, String reason) {
        AddToCartResponse.FailedItem item = new AddToCartResponse.FailedItem();
        item.setProductId(productId);
        item.setReason(reason);
        return item;
    }
}
```

## 3. DTO 层

### SmartListRequest.java

```java
package com.hema.smartlist.dto;

import lombok.Data;

/**
 * 生成智能清单请求 DTO
 */
@Data
public class SmartListRequest {
    
    /**
     * 用户ID
     */
    private Long userId;
    
    /**
     * 场景描述（如：周末请6个朋友来家里吃火锅）
     */
    private String sceneDescription;
    
    /**
     * 人数
     */
    private Integer peopleCount;
    
    /**
     * 过敏原列表
     */
    private String[] allergens;
}
```

### SmartListResponse.java

```java
package com.hema.smartlist.dto;

import lombok.Data;
import java.util.List;

/**
 * 智能清单响应 DTO
 */
@Data
public class SmartListResponse {
    
    /**
     * 清单ID
     */
    private String listId;
    
    /**
     * 场景描述
     */
    private String sceneDescription;
    
    /**
     * 人数
     */
    private Integer peopleCount;
    
    /**
     * 商品总数
     */
    private Integer totalItems;
    
    /**
     * 预估金额
     */
    private Double estimatedAmount;
    
    /**
     * 分类商品列表
     */
    private List<CategoryItem> categories;
    
    /**
     * 分类商品项
     */
    @Data
    public static class CategoryItem {
        /**
         * 分类名称
         */
        private String categoryName;
        
        /**
         * 商品列表
         */
        private List<ProductItem> items;
    }
    
    /**
     * 商品项
     */
    @Data
    public static class ProductItem {
        /**
         * 商品ID
         */
        private String productId;
        
        /**
         * 商品名称
         */
        private String productName;
        
        /**
         * 规格
         */
        private String specification;
        
        /**
         * 价格
         */
        private Double price;
        
        /**
         * 数量
         */
        private Integer quantity;
        
        /**
         * 商品图片URL
         */
        private String imageUrl;
        
        /**
         * 库存
         */
        private Integer stock;
        
        /**
         * 推荐理由
         */
        private String reason;
    }
}
```

### AddToCartRequest.java

```java
package com.hema.smartlist.dto;

import lombok.Data;
import java.util.List;

/**
 * 一键加购请求 DTO
 */
@Data
public class AddToCartRequest {
    
    /**
     * 用户ID
     */
    private Long userId;
    
    /**
     * 清单ID
     */
    private String listId;
    
    /**
     * 商品列表
     */
    private List<CartItem> items;
    
    /**
     * 购物车商品项
     */
    @Data
    public static class CartItem {
        /**
         * 商品ID
         */
        private String productId;
        
        /**
         * 数量
         */
        private Integer quantity;
    }
}
```

### AddToCartResponse.java

```java
package com.hema.smartlist.dto;

import lombok.Data;
import java.util.List;

/**
 * 一键加购响应 DTO
 */
@Data
public class AddToCartResponse {
    
    /**
     * 购物车ID
     */
    private String cartId;
    
    /**
     * 成功数量
     */
    private Integer successCount;
    
    /**
     * 失败数量
     */
    private Integer failedCount;
    
    /**
     * 失败商品列表
     */
    private List<FailedItem> failedItems;
    
    /**
     * 失败商品项
     */
    @Data
    public static class FailedItem {
        /**
         * 商品ID
         */
        private String productId;
        
        /**
         * 失败原因
         */
