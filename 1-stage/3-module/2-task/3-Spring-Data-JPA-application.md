> 第三部分 Spring Data JPA 应用

需求：使用 Spring Data JPA 完成对 tb_resume 表 的 Dao 层操作（增删改查、排序，分页等）

数据表设计：

![image-20220412191129253](assest/image-20220412191129253.png)

初始化 sql 语句：

```sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;
-- ---------------------------- 
-- Table structure for tb_resume
-- ---------------------------- 
DROP TABLE IF EXISTS `tb_resume`; 
CREATE TABLE `tb_resume` (
	`id` BIGINT (20) NOT NULL AUTO_INCREMENT,
	`address` VARCHAR (255) DEFAULT NULL,
	`name` VARCHAR (255) DEFAULT NULL,
	`phone` VARCHAR (255) DEFAULT NULL,
	PRIMARY KEY (`id`)
) ENGINE = INNODB AUTO_INCREMENT = 4 DEFAULT CHARSET = utf8;
-- ---------------------------- 
-- Records of tb_resume
-- ---------------------------- 
BEGIN;
INSERT INTO `tb_resume` VALUES (1, '北京', '张三', '131000000');
INSERT INTO `tb_resume` VALUES (2, '上海', '李四', '151000000');
INSERT INTO `tb_resume` VALUES (3, '⼴州', '王五', '153000000');
COMMIT;
SET FOREIGN_KEY_CHECKS = 1;
```

# 1 Spring Data JPA 开发步骤梳理

- 构建工程
  - 创建工程导入坐标（Java框架于我们而言就是一堆 jar）
  - 配置 Spring 的配置文件（配置指定框架执行的细节）
  - 编写实体类 Resume，使用 JPA 注解配置映射关系
  - 编写一个符合 Spring Data JPA 的 Dao 层接口（ResumeDao 接口）
- 操作 ResumeDao 接口完成 Dao 层开发

# 2 Spring Data JPA 开发实现

