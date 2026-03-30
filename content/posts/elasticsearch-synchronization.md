+++
date = '2026-03-30T10:09:27+08:00'
draft = false
title = '解决Elasticsearch数据同步痛点：如何实现业务零侵入与高性能兼得？'
+++

## 引言：
Elasticsearch （以下称es）是一个开源的分布式搜索和分析引擎。它是 Elastic Stack（简称 ELK Stack）的核心组件，能够让你以极快的速度存储、搜索和分析海量数据。

搜索场景下常结合数据库和es实现分词搜索等功能，还提供有搜索联想、高亮显示等强大功能。

而使用es的一大要点就是实现es和数据库之间的**数据同步**。
## 一、数据同步的类型
es的同步分为**全量同步**和**增量同步**。

- **全量同步**：把整个数据库的所有数据对应同步到es中，一般是应用启动或者出现数据缺失等时候使用。
- **增量同步**：系统在运行过程中数据发生改变，es随数据改变相应的修改数据。

全量同步一般都是扫描全表组装数据并添加，这里主要讨论增量同步的方法
## 二、数据同步的方法
### 1. 同步双写

这是最直接的同步es数据的方法。在数据库修改的代码之后，直接添加修改es的代码。
- 优点：操作简单，逻辑清晰，实时性高。
- 缺点：
    - 业务耦合度较高，所有需要修改es对应数据库的代码都需要添加修改ed的代码。
    - **安全性较差**，如果再同步时出错就有数据不一致性。

### 2. 异步消息队列

这种方法引入了消息中间件，业务代码只负责写入数据，成功后向消息队列发送消息，由消费者独立更新数据同步
- 优点：
    - **代码解耦**，只需要在修改后发送消息。
    - **性能提升**，通过异步来处理同步，不堵塞主流程。
- 缺点：
    - 引入额外中间件，维护的成本相对增高。
    - 需要处理消息丢失重试等问题。

### 3. 定时任务

添加定时任务，批量同步更新数据。
- 优点：对业务代码无侵入，只需要处理数据库修改。
- 缺点：
    - 定时任务轮询添加数据库压力。
    - 实时性差，存在分钟级甚至更长的同步延迟

### 4. Binlog监听

使用canal等工具伪装成数据库的从库，监听数据库的binlog日志并发送到对应消息队列中，由消费者处理更新。
- 优点：
    - **业务代码零侵入**，用户只需要编写数据库代码，不用考虑es同步问题。
    - **实时性好**，结合消息队列能几乎达到无感同步
- 缺点
    - 运维复杂度高，引入多个中间件
    - 同消息队列方法，需要处理消息丢失重试等问题。

## 三、项目实战
### 1. 核心架构思路
- 主流程（实时）：利用Alibaba Canal伪装成MySQL从库，监听 Binlog 日志
- 兜底流程（准实时）：对于影响范围大（如涉及大量关联字段计算）或不便直接更新的列，我们不直接操作es，而是将id缓存到redis中。
- 定时任务： 开启定时任务，扫描redis中的id，批量查询最新数据并全量覆盖es文档。
这种设计既保证了高频简单变更的实时性，又避免了复杂更新带来的性能抖动。

### 2. 代码实现
在消费者中，建议对DDL语句进行过滤，只保留对数据操作的处理。

**1. 核心消费者：CanalListener**
```java
@Slf4j
@Component
@RequiredArgsConstructor
public class CanalListener {

    private final ElasticsearchUtils elasticsearchUtils;
    private final CacheService cacheService;
    private final ObjectMapper objectMapper;

    /**
     * 消费canal同步数据库的消息
     * @param message mysql修改数据信息
     */
    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(value = "canal.queue", durable = "true"),
            exchange = @Exchange(value = "canal.exchange"),
            key = "canal.routing.key"
    ), containerFactory = "rabbitListenerContainerFactory")
    public void listenCanal(org.springframework.amqp.core.Message message) {
        // 获取消息并转换类型
        String json = new String(message.getBody(), StandardCharsets.UTF_8);
        log.debug("收到canal mq消息" + json);
        CanalMessage msg = JSON.parseObject(json, CanalMessage.class);

        // 关键：过滤DDL操作
        if (Boolean.TRUE.equals(msg.getIsDdl())) {
            log.debug("跳过DDL操作: {}", msg.getSql());
            return;  // 直接返回，不处理
        }

        synchronousData(msg);
    }

    /**
     * 同步数据
     * @param msg canal的消息
     */
    private void synchronousData(CanalMessage msg) {
        switch (msg.getTable()) {
            case "course" -> synchronousCourse(msg);
            case "course_category" -> synchronousCourseCategory(msg);
            case "teacher_info" -> synchronousTeacherInfo(msg);
            case "category" -> synchronousCategory(msg);
            case "dict_education" -> synchronousDictEducation(msg);
            case "post" -> synchronousPost(msg);
            case "user_info" -> synchronousUserInfo(msg);
        }
    }

    /**
     * course_category表变化处理
     * @param msg canal的消息
     */
    private void synchronousCourseCategory(CanalMessage msg) {
        List<Long> courseIds = getDataFieldAsList(msg, "course_id", Long.class);

        if (CollectionUtil.isEmpty(courseIds)) {
            return;
        }

        // 缓存要更新的课程到redis
        switch (msg.getType()) {
            case "INSERT", "UPDATE", "DELETE" -> cacheService.cacheCanalSynchronousCourseId(courseIds);
        }
    }

    /**
     * course表变化处理
     * @param msg canal的消息
     */
    private void synchronousCourse(CanalMessage msg) {
        List<Long> courseIds = getDataFieldAsList(msg, "id", Long.class);
        if (CollectionUtil.isEmpty(courseIds)) {
            return;
        }

        switch (msg.getType()) {
            // 缓存要更新的课程到redis
            case "INSERT", "UPDATE" -> cacheService.cacheCanalSynchronousCourseId(courseIds);
            // 直接删除课程
            case "DELETE" -> {
                BulkResponse deleteResponses = elasticsearchUtils.bulkDelete(CourseIndexConstant.INDEX, courseIds);
                if (deleteResponses != null) {
                    log.info("批量删除课程数据 {} 条", deleteResponses.getItems().length);
                }
            }
        }
    }

    /**
     * teacher_info表变化处理
     * @param msg canal的消息
     */
    private void synchronousTeacherInfo(CanalMessage msg) {
        for (int i = 0; i < msg.getData().size(); i++) {
            // DELETE教师数据要求处理数据库对应课程信息，对应操作日志再处理
            if ("UPDATE".equals(msg.getType())) {
                // 获取变更数据
                TeacherInfo data = convertRow(msg.getData().get(i), TeacherInfo.class);
                TeacherInfo oldData = convertRow(msg.getOld().get(i), TeacherInfo.class);
                Set<String> changedFields = msg.getOld().get(i).keySet();
                // 构建需要更新字段的map
                HashMap<String, Object> courseMap = new HashMap<>();
                HashMap<String, Object> postMap = new HashMap<>();
                // 课程部分
                if (changedFields.contains("id")) {
                    courseMap.put(CourseIndexConstant.TEACHER_ID, data.getId());
                }
                if (changedFields.contains("name")) {
                    courseMap.put(CourseIndexConstant.TEACHER_NAME, data.getName());
                }
                // 更新课程字段
                BulkResponse updateResponses = elasticsearchUtils.batchUpdateByField(
                        CourseIndexConstant.INDEX, CourseIndexConstant.TEACHER_ID, oldData.getId() != null ? oldData.getId() : data.getId(), courseMap
                );
                if (updateResponses != null) {
                    log.info("批量更新课程数据 {} 条", updateResponses.getItems().length);
                }
            }
        }
    }

    /**
     * category表变化处理
     * @param msg canal的消息
     */
    private void synchronousCategory(CanalMessage msg) {
        for (int i = 0; i < msg.getData().size(); i++) {
            // DELETE类别数据要求处理数据库对应课程信息，对应操作日志再处理
            if ("UPDATE".equals(msg.getType())) {
                // 获取变更数据
                Category data = convertRow(msg.getData().get(i), Category.class);
                Category oldData = convertRow(msg.getOld().get(i), Category.class);
                Set<String> changedFields = msg.getOld().get(i).keySet();
                // 构建需要更新字段的map
                HashMap<String, Object> map = new HashMap<>();
                if (changedFields.contains("id")) {
                    map.put("id", data.getId());
                }
                if (changedFields.contains("name")) {
                    map.put("name", data.getName());
                }
                if (changedFields.contains("level")) {
                    map.put("level", data.getLevel());
                }
                if (map.isEmpty()) {
                    continue;
                }
                // 更新字段
                Long changeFieldId = oldData.getId() != null ? oldData.getId() : data.getId();
                BulkResponse updateResponses = elasticsearchUtils.batchUpdateNested(
                        CourseIndexConstant.INDEX,
                        CourseIndexConstant.CATEGORIES,
                        CourseIndexConstant.CATEGORIES_ID,
                        changeFieldId,
                        changeData -> changeData.stream()
                                .filter(item -> changeFieldId.equals((item.get("id"))))
                                .forEach(item -> {
                                    for (String key : map.keySet()) {
                                        item.put(key, map.get(key));
                                    }
                                })
                );
                if (updateResponses != null) {
                    log.info("批量更新课程数据 {} 条", updateResponses.getItems().length);
                }
            }
        }
    }

    /**
     * dict_education表变化处理
     * @param msg canal的消息
     */
    private void synchronousDictEducation(CanalMessage msg) {
        for (int i = 0; i < msg.getData().size(); i++) {
            // DELETE学历数据要求处理数据库对应课程信息，对应操作日志再处理
            if ("UPDATE".equals(msg.getType())) {
                // 获取变更数据
                DictEducation data = convertRow(msg.getData().get(i), DictEducation.class);
                DictEducation oldData = convertRow(msg.getOld().get(i), DictEducation.class);
                Set<String> changedFields = msg.getOld().get(i).keySet();
                // 构建需要更新字段的map
                HashMap<String, Object> map = new HashMap<>();
                if (changedFields.contains("id")) {
                    map.put("id", data.getId());
                }
                if (changedFields.contains("code")) {
                    map.put("code", data.getCode());
                }
                if (changedFields.contains("name")) {
                    map.put("name", data.getName());
                }
                // 更新字段
                Long changeFieldId = oldData.getId() != null ? oldData.getId() : data.getId();
                BulkResponse updateResponses = elasticsearchUtils.batchUpdateNested(
                        CourseIndexConstant.INDEX,
                        CourseIndexConstant.EDUCATION,
                        CourseIndexConstant.EDUCATION_ID,
                        changeFieldId,
                        changeData -> changeData.stream()
                                .filter(item -> changeFieldId.equals((item.get("id"))))
                                .forEach(item -> {
                                    for (String key : map.keySet()) {
                                        item.put(key, map.get(key));
                                    }
                                })
                );
                if (updateResponses != null) {
                    log.info("批量更新课程数据 {} 条", updateResponses.getItems().length);
                }
            }
        }
    }

    /**
     * @param row 把 Canal 的一行 Map 转成指定实体（下划线→驼峰）
     * @param clazz 转换的类型
     * @return 返回对应的实体
     * @param <T> 实体的类型
     */
    private <T> T convertRow(Map<String, Object> row, Class<T> clazz) {
        try {
            // 转成 JSON 字符串
            String json = objectMapper.writeValueAsString(row);

            // 再建一个仅本次用下划线策略的 mapper
            ObjectMapper snakeMapper = objectMapper.copy()
                    .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);

            // 用下划线策略反序列化
            return snakeMapper.readValue(json, clazz);
        } catch (Exception e) {
            throw new IllegalArgumentException("行转换失败", e);
        }
    }

    /**
     * 根据canal消息获取其中一个字段的list
     * @param msg canal的消息
     * @param fieldName 字段名
     * @param type 字段的类型
     * @return 返回字段的list
     * @param <T> 字段的类型
     */
    private <T> List<T> getDataFieldAsList(CanalMessage msg, String fieldName, Class<T> type) {
        if (msg == null || msg.getData() == null || msg.getData().isEmpty()) {
            return new ArrayList<>();
        }
        return msg.getData().stream()
                .map(item -> item.get(fieldName))
                .filter(Objects::nonNull)
                .map(obj -> objectMapper.convertValue(obj, type))
                .toList();
    }
}
```

**2. 定时任务轮询**
```java
@Component
@Slf4j
@RequiredArgsConstructor
public class CanalTask {
    private final CourseService courseService;
    private final CacheService cacheService;
    private final ElasticsearchUtils elasticsearchUtils;
    private final PostService postService;

    /**
     * 每10秒执行一次数据同步到ES
     */
    @Scheduled(cron = "*/10 * * * * ?")
    public void synchronousTask() {
        try {
            // 批量更新课程
            synchronousCourse();
        } catch (Exception e) {
            log.error("同步数据失败");
            throw e;
        }
    }

    /**
     * 同步课程数据
     */
    private void synchronousCourse() {
        List<Long> courseIds = cacheService.getAllSynchronousCourseIds();
        if (CollectionUtil.isEmpty(courseIds)) {
            return;
        }
        log.info("开始定时同步课程数据到es");
        List<CourseDocument> courseDocuments = courseService.getCourseDocumentsByIds(courseIds);
        // 获取包含suggest的数据
        List<CourseDocumentWithSuggest> courseDocumentWithSuggests = courseDocuments.stream().map(CourseDocumentWithSuggest::of).toList();
        // 新增或覆盖es的数据
        BulkResponse indexResponses = elasticsearchUtils.bulkIndex(CourseIndexConstant.INDEX, courseDocumentWithSuggests, doc -> String.valueOf(doc.getId()));
        log.info("批量同步课程数据 {} 条", indexResponses == null ? 0 : indexResponses.getItems().length);
        log.info("同步课程数据完成");
    }
}
```
---
**总结**

数据一致性是 ES 应用的关键挑战。本文实践了一套基于 SpringBoot 的混合同步方案：
利用 Canal 实现 Binlog 实时监听以处理简单变更，结合 Redis 缓存与定时任务处理复杂关联数据，有效平衡了系统的性能、实时性与维护成本。